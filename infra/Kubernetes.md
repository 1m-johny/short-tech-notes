# Kubernetes

## Container orchestration

Kubernetes is used for deploying and managing hundreds or thousands of containers in a clustered environment.

## Core components

| Component                | Responsibility                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------ |
| API Server               | Front door to Kubernetes; validates and processes API requests                       |
| etcd                     | Strongly consistent key-value store containing cluster state                         |
| Scheduler                | Selects a suitable node for every unscheduled Pod                                    |
| Controller Manager       | Runs reconciliation controllers such as Deployment, ReplicaSet, and Node controllers |

## Deployment creation flow

A more accurate flow is:

1. A Deployment controller creates a ReplicaSet.
2. The ReplicaSet controller creates Pod objects.
3. The scheduler assigns each Pod to a node.
4. The node's kubelet asks the container runtime to start the containers.

Controllers do not normally create worker nodes themselves. Node provisioning is handled by systems such as Karpenter, Cluster Autoscaler, managed node groups, or cloud APIs.

## Worker node components

| Component         | Responsibility                                                     |
| ----------------- | ------------------------------------------------------------------ |
| kubelet           | Ensures the node's assigned Pods and containers are running        |
| Container runtime | Runs containers, commonly containerd or CRI-O                      |
| kube-proxy        | Implements Service networking, usually with iptables or IPVS rules |
| CNI plugin        | Provides Pod networking and IP allocation, aws vpc cni / cilium    |

## Pods

A Pod is the smallest object that can be created in Kubernetes.

- One Pod typically maps to one container.
- We cannot create the same container inside the same Pod.
- Multiple helper containers can run inside the same Pod.

### Why use Pods?

If we need to run a Python app with helper APIs, we might have:

- Python-1, Python-2, Python-3, Python-4
- Helper-1, Helper-2, Helper-3, Helper-4

We need to manage:

- network communication between Python and helper containers
- shared storage
- mapping details
- cleanup if Python crashes

A Pod handles these concerns and makes life easier.

## Replication Controller vs ReplicaSet

Replication Controller == ReplicaSet

- It ensures a desired number of Pods are running.
- It recreates Pods if they crash or are deleted.

## Deployments

Deployments provide the capability to upgrade underlying instances seamlessly using rolling updates.

A Deployment will create a ReplicaSet, which will create Pods, and those Pods use images to create containers.

### Strategies

- Recreate: destroy and update all at once.
- Rolling update: destroy and update one by one.

Whenever the deployment definition file is updated, Kubernetes performs a rolling update.

## Services

### Why use Services?

- Pods are ephemeral and can be recreated frequently.
- A restarted Pod may get a new IP.
- A Service provides a stable IP address and DNS name.
- It exposes the application to the outside world.
- If multiple Pods share the same labels across nodes, the Service selects them and distributes traffic.

To access an application deployed as a Pod and exposed through a Service, you only need to specify the Service name. It resolves to the Service IP via CoreDNS, which is deployed by default with an Amazon EKS cluster. This works when you are inside the cluster.

### Service types

#### ClusterIP

- Default type
- Used for internal communication only

#### NodePort

- Exposes a Service on a static port on each node
- Provides less load balancing, service discovery, and network security than a LoadBalancer Service
- Traffic is forwarded to each node IP:static_port

#### LoadBalancer

- Provisions an ALB in the cloud
- Combines load balancing, service discovery, and service security
- User -> ALB -> Service -> Deployment

#### Ingress

- Out of scope for this note
- Requires an Ingress controller
- Creates a cloud load balancer
- Routes traffic to multiple Services based on path and hostname

## Volumes

A Pod is ephemeral, so anything stored inside it is lost when the Pod restarts. This is where Volumes are useful.

### Volume types

- `emptyDir`: shared storage between containers in the same Pod; lost when the Pod restarts
- `hostPath`: storage created at the node level; lost if the node is disrupted or the Pod is rescheduled elsewhere
- `awsElasticBlockStore`: requires adding a volume ID to the manifest file

## Kubernetes dependency basics

- Without the Controller Manager, a Deployment will not create a ReplicaSet.
- Without the Scheduler, a ReplicaSet will not create Pods because it does not know where to schedule them.
- `version error` means kubelet is not connected to the API server.
- `node not ready` means kubelet has stopped.

## CNI

CNI manages all networking-related things, including CIDR allocation and Pod networking.

## CoreDNS and Reloader in EKS

### CoreDNS

- DNS server: resolves names within the cluster
- Service discovery: allows Services to be discovered by DNS names
- Caching and forwarding: caches DNS records and forwards unresolved requests to upstream DNS servers
- Load balancing: distributes DNS queries across multiple backends

Example nameserver:

```bash
nameserver 10.100.0.10
```

This is the default nameserver assigned to Pods.

### Reloader

Reloader is an open-source tool that automatically triggers a rolling restart of Pods when ConfigMaps or Secrets are updated.

It monitors Kubernetes ConfigMaps and Secrets and restarts dependent Pods to apply new configuration without manual steps.

## Scheduling constraints

### Affinities

#### Node affinity

Directs the scheduler to only place certain Pods on nodes with specific labels or in certain availability zones.

#### Pod affinity

Prioritizes scheduling based on other Pods already running on the target node.

#### Pod anti-affinity

Ensures that certain Pods do not run on the same node as other Pods.

### Topology spread

Topology spread helps distribute Pods across topological domains such as nodes, zones, or regions for better fault tolerance and resource utilization.

### Persistent volume topology

This is used to align persistent storage placement with node topology requirements.

### Taints and tolerations

- Nodes can be tainted to allow only specific Pods to schedule on them.
- Pods can have tolerations to be allowed on tainted nodes.

### Pod Disruption Budget (PDB)

A Pod Disruption Budget defines the minimum number or percentage of Pods that must remain available during voluntary disruptions.

- Pods are not forcibly deleted.
- If a Pod fails to terminate cleanly, it can prevent node deprovisioning.
- It blocks node termination when evicting a Pod would violate the budget.

## Service account

A Kubernetes Service Account is attached to AWS IAM policies and roles.

- Restricted to a namespace
- Works via IAM OIDC Provider exposed by EKS
- IAM roles are mapped to the Service Account via OIDC
- EKS admission controllers inject AWS session credentials into Pods based on the Service Account annotation

## StatefulSets

A StatefulSet is used to manage stateful applications.

Unlike Deployments, which are designed for stateless apps, StatefulSets are built for applications that require:

- persistent storage
- stable network identities
- ordered and predictable deployment/scaling

### Key features

1. Stable network identity: each Pod gets a unique hostname such as `my-app-0`, `my-app-1`, `my-app-2`
2. Stable storage: each Pod gets its own PersistentVolumeClaim (PVC)
3. Ordered deployment and scaling: Pods are created and terminated in a defined order
4. Graceful termination: Pods are terminated in reverse order during scale-down or deletion

## EKS

`eksctl` creates default resources when creating a cluster:

- cluster
- VPC
  - public subnets (2)
  - private subnets (2)
  - route tables
  - security groups
  - Internet Gateway
  - NAT Gateway
- addons
  - vpc-cni
  - kube-proxy
  - coredns
- addon roles
- no OIDC, so no attachment for addon roles
- no CloudWatch logs
- kubeconfig file

### Cluster fields

- metadata
- vpc
- managedNodeGroups
- nodegroup
- iam
- addons
- accessConfig
  - authenticationMode
- outpost

### EKS Pod Identity Associations

- No need for OIDC for this specific use case
- Same role can be used across namespaces
- Can also be added to EKS add-ons
- Existing IAM service accounts and add-ons can be migrated to Pod Identity Associations with a single command

### Other EKS notes

- Non-EKS clusters can be registered with EKS Connector
- kubelet configuration can be managed
- CloudWatch logging can be enabled
- fully private clusters are supported
- add-ons and Fargate can be used
- cluster upgrades include control plane, add-ons, and nodes

## Webhooks

### MutatingAdmissionWebhook

Used to modify or mutate a resource before it is persisted in etcd. Example: adding a default sidecar to Pods during deployment.

### ValidatingAdmissionWebhook

Used to validate a resource's configuration before it is accepted. Example: enforcing naming conventions or required fields.

## Commands

```bash
kubectl create -f pod-definition.yml
kubectl apply -f pod.yml
kubectl run redis --image=redis123 --dry-run=client -o yaml > pod.yaml
kubectl edit pod redis
kubectl delete -f pod.yaml
kubectl get pods
kubectl get nodes
kubectl get nodes -o wide
kubectl describe pods
kubectl describe pod podName
kubectl create -f replicaset-def.yml
kubectl get replicaset
kubectl delete replicaset replicaset-name
kubectl replace -f replicaset-definition.yml
kubectl scale --replicas=6 replicaset replicaset-name
kubectl scale --replicas=6 -f replicaset-def.yml
kubectl create -f deployment-definition.yml
kubectl get deployments
kubectl apply -f deployment-definition.yml
kubectl set image deployment/deployment-name nginx=nginx:1.9.1
kubectl rollout status deployment/deployment-name
kubectl rollout history deployment/deployment-name
kubectl rollout undo deployment/deployment-name
```

### CRUD commands

```bash
kubectl create deployment deployment_name --image=imageName
kubectl run nginx --image=nginx
kubectl edit deployment deployment_name
kubectl delete deployment deployment_name
```

### Status commands

```bash
kubectl get deployments
kubectl get deployment deployment_name
kubectl describe deployment deployment_name
```

### Debugging commands

```bash
kubectl logs deployment_name
kubectl exec -it podName -- bin/bash
kubectl describe replicaset deployment_name
kubectl edit replicaset deployment_name
kubectl scale replicaset deployment_name --replicas=3
kubectl get replicasets.apps
kubectl get replicaset deployment_name
kubectl describe replicaset deployment_name
kubectl delete replicaset deployment_name
kubectl get all
kubectl get events -n "$TARGET_NAMESPACE" --sort-by='.metadata.creationTimestamp' --field-selector type=Warning > events.log
```

## Quick troubleshooting notes

- `version error` -> kubelet is not connected to the API server
- `node not ready` -> kubelet is stopped
- `without Controller Manager` -> Deployment will not create a ReplicaSet
- `without Scheduler` -> ReplicaSet will not create Pods because it lacks a scheduling target

## Summary

Kubernetes is a container orchestration system that manages scheduling, scaling, networking, updates, and service exposure across a cluster of nodes. Concepts such as Pods, Services, Deployments, StatefulSets, and networking plugins are the foundation for modern cloud-native workloads.
