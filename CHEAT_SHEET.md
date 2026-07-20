# Kubernetes Cheat Sheet

> A quick-reference guide for Kubernetes commands, architecture, debugging, and interview revision.
>
> **Purpose:** Fast recall during interviews, troubleshooting, and daily Kubernetes development.

---

# Table of Contents

1. kubectl Commands
2. Architecture Flows
3. Workload Relationships
4. Scheduling
5. Networking
6. Storage
7. Security
8. Resource Management
9. Debugging Commands
10. Pod States & Common Errors
11. Rollouts & Production Operations
12. Important Ports
13. Interview Nuggets
14. One-Line Definitions
15. Who Does What?

---

# 1. kubectl Commands

## Cluster Information

```bash
kubectl cluster-info
kubectl version
kubectl config view
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <context-name>
```

---

## Get Resources

```bash
kubectl get nodes
kubectl get namespaces
kubectl get pods
kubectl get pods -A
kubectl get pods -o wide
kubectl get deployments
kubectl get services
kubectl get ingress
kubectl get pvc
kubectl get pv
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get all
```

Useful output options

```bash
-o yaml
-o json
-o wide
--show-labels
```

---

## Describe Resources

```bash
kubectl describe pod <pod>
kubectl describe deployment <deployment>
kubectl describe service <service>
kubectl describe node <node>
kubectl describe pvc <pvc>
```

---

## Logs

Current logs

```bash
kubectl logs <pod>
```

Specific container

```bash
kubectl logs <pod> -c <container>
```

Previous container logs

```bash
kubectl logs <pod> --previous
```

Follow logs

```bash
kubectl logs -f <pod>
```

---

## Execute Inside Container

```bash
kubectl exec -it <pod> -- bash
kubectl exec -it <pod> -- sh
```

---

## Apply / Delete

```bash
kubectl apply -f file.yaml
kubectl apply -k .

kubectl delete -f file.yaml
kubectl delete pod <pod>
kubectl delete deployment <deployment>
```

---

## Edit Resources

```bash
kubectl edit deployment <deployment>
kubectl edit service <service>
kubectl edit configmap <configmap>
```

---

## Scale

```bash
kubectl scale deployment <deployment> --replicas=5
```

---

## Rollout

```bash
kubectl rollout status deployment <deployment>

kubectl rollout history deployment <deployment>

kubectl rollout restart deployment <deployment>

kubectl rollout undo deployment <deployment>

kubectl rollout pause deployment <deployment>

kubectl rollout resume deployment <deployment>
```

---

## Labels & Annotations

Add label

```bash
kubectl label node worker-1 gpu=true
```

Remove label

```bash
kubectl label node worker-1 gpu-
```

Add annotation

```bash
kubectl annotate pod mypod owner=dev-team
```

Remove annotation

```bash
kubectl annotate pod mypod owner-
```

---

## Taints

Add taint

```bash
kubectl taint nodes worker-1 gpu=true:NoSchedule
```

Remove taint

```bash
kubectl taint nodes worker-1 gpu=true:NoSchedule-
```

---

## Metrics

```bash
kubectl top pod
kubectl top node
```

---

## Debugging

```bash
kubectl debug <pod>

kubectl debug node/<node>

kubectl cp <pod>:<path> .

kubectl port-forward pod/<pod> 8080:80
```

---

## Namespace

```bash
kubectl create namespace dev

kubectl get ns

kubectl config set-context --current --namespace=dev
```

---

## Useful Selectors

```bash
kubectl get pods -l app=frontend

kubectl get pods -l env=prod

kubectl get pods --field-selector=status.phase=Running
```

---
# 2. Kubernetes Architecture

## Control Plane Components

| Component | Responsibility |
|----------|----------------|
| API Server | Entry point to the Kubernetes cluster |
| etcd | Stores the entire cluster state |
| Scheduler | Assigns newly created Pods to Nodes |
| Controller Manager | Reconciles desired state with actual state |
| Cloud Controller Manager | Integrates Kubernetes with cloud providers |

---

## Worker Node Components

| Component | Responsibility |
|----------|----------------|
| kubelet | Runs Pods on the Node |
| kube-proxy | Handles Service networking |
| Container Runtime | Runs containers (containerd, CRI-O, etc.) |

---

## Cluster Architecture

```text
                  kubectl
                      │
                      ▼
               +--------------+
               |  API Server  |
               +--------------+
                 │    │     │
                 │    │     │
        Scheduler etcd Controller Manager
                 │
        ------------------------
        │                      │
        ▼                      ▼
   Worker Node 1         Worker Node 2
        │                      │
    kubelet               kubelet
        │                      │
 containerd              containerd
        │                      │
      Pods                   Pods
```

---

## Request Flow

```text
kubectl apply

↓

API Server

↓

etcd (Stores Desired State)

↓

Scheduler (Chooses Node)

↓

kubelet

↓

Container Runtime

↓

Pod Running
```

---

## External Request Flow

```text
Client

↓

DNS

↓

Load Balancer

↓

Ingress

↓

Service

↓

EndpointSlice

↓

Pod

↓

Container
```

---

# 3. Core Objects & Relationships

## Workload Hierarchy

```text
Deployment

↓

ReplicaSet

↓

Pods

↓

Containers
```

---

## Service Discovery

```text
Service

↓

EndpointSlice

↓

Pods
```

---

## Storage Flow

```text
Pod

↓

PersistentVolumeClaim (PVC)

↓

PersistentVolume (PV)

↓

StorageClass

↓

Physical Storage
```

---

## Job Flow

```text
CronJob

↓

Job

↓

Pod

↓

Completed
```

---

## DaemonSet

```text
DaemonSet

↓

One Pod Per Node
```

---

## StatefulSet

```text
StatefulSet

↓

Stable Pod Identity

↓

Persistent Storage

↓

Ordered Deployment
```

---

## Object Responsibilities

| Object | Purpose |
|---------|----------|
| Pod | Smallest deployable unit |
| ReplicaSet | Maintains replica count |
| Deployment | Manages ReplicaSets and rolling updates |
| Service | Stable network endpoint |
| Ingress | HTTP/HTTPS routing |
| ConfigMap | Non-sensitive configuration |
| Secret | Sensitive configuration |
| Job | Runs to completion |
| CronJob | Scheduled Jobs |
| DaemonSet | One Pod on every Node |
| StatefulSet | Stateful applications |
| PVC | Requests persistent storage |
| PV | Provides persistent storage |

---

# 4. Scheduling

## Scheduling Flow

```text
Pod Created

↓

Scheduler

↓

Filter Nodes

↓

Score Nodes

↓

Select Best Node

↓

Bind Pod
```

---

## Scheduling Mechanisms

| Feature | Controlled By | Purpose |
|----------|---------------|----------|
| nodeSelector | Pod | Exact node selection |
| Node Affinity | Pod | Advanced node selection |
| Pod Affinity | Pod | Place Pods together |
| Pod Anti-Affinity | Pod | Separate Pods |
| Taints | Node | Reject Pods |
| Tolerations | Pod | Allow Pod on tainted Nodes |
| PriorityClass | Pod | Priority during scheduling |
| QoS Class | Kubernetes | Eviction priority |

---

## Node Selector vs Affinity

| Node Selector | Node Affinity |
|--------------|---------------|
| Simple | Advanced |
| Exact match | Expressions |
| AND logic only | AND / OR logic |
| No preferences | Preferred & Required rules |

---

## Taints vs Tolerations

| Taint | Toleration |
|--------|------------|
| Applied to Node | Applied to Pod |
| Rejects Pods | Allows scheduling |
| Protects Nodes | Grants permission |

---

## Pod Affinity vs Anti-Affinity

| Feature | Purpose |
|----------|---------|
| Pod Affinity | Schedule Pods together |
| Pod Anti-Affinity | Spread Pods across Nodes |

---

## Common Scheduling Scenarios

| Scenario | Feature |
|----------|---------|
| GPU workloads | nodeSelector + Toleration |
| Database Nodes | Affinity + Taints |
| SSD Nodes | nodeSelector |
| Spread replicas | Pod Anti-Affinity |
| Monitoring on every Node | DaemonSet |
| Control Plane Pods | Tolerations |
| Zone-aware scheduling | Node Affinity |

---

## Scheduling Decision Order

```text
1. Resource Availability

↓

2. nodeSelector / Affinity

↓

3. Taints & Tolerations

↓

4. Pod Affinity / Anti-Affinity

↓

5. Priority

↓

6. Best Node Selected
```

---
# 5. Networking

## Service Types

| Service | Use Case | External Access |
|----------|----------|-----------------|
| ClusterIP | Internal communication | ❌ No |
| NodePort | Development / Testing | ✅ Yes |
| LoadBalancer | Cloud environments | ✅ Yes |
| ExternalName | Maps to external DNS | N/A |

---

## Networking Flow

```text
Client

↓

Ingress

↓

Service

↓

EndpointSlice

↓

Pod
```

---

## DNS Resolution

```
<service-name>.<namespace>.svc.cluster.local
```

Example

```
mysql.default.svc.cluster.local
```

---

## Important Networking Components

| Component | Purpose |
|----------|----------|
| Service | Stable virtual IP |
| EndpointSlice | Stores Pod endpoints |
| kube-proxy | Implements Service networking |
| CoreDNS | Cluster DNS |
| Ingress | HTTP/HTTPS routing |
| Ingress Controller | Implements Ingress rules |
| NetworkPolicy | Controls Pod-to-Pod communication |

---

## NetworkPolicy

| Policy Type | Controls |
|-------------|----------|
| Ingress | Incoming traffic |
| Egress | Outgoing traffic |

---

## Common Ports

| Port | Purpose |
|------|----------|
| 80 | HTTP |
| 443 | HTTPS |
| 53 | DNS |

---

# 6. Storage

## Storage Flow

```text
Application

↓

Pod

↓

PVC

↓

PV

↓

StorageClass

↓

Physical Storage
```

---

## Storage Objects

| Object | Purpose |
|---------|----------|
| PV | Actual storage resource |
| PVC | Request for storage |
| StorageClass | Dynamic provisioning |
| CSI | Storage driver interface |

---

## Volume Types

| Volume | Persistent | Typical Use |
|---------|------------|-------------|
| emptyDir | ❌ | Temporary files |
| hostPath | Node only | Development |
| ConfigMap | ❌ | Configuration |
| Secret | ❌ | Credentials |
| PVC | ✅ | Databases |
| CSI | ✅ | Cloud storage |

---

## PVC Access Modes

| Mode | Meaning |
|------|----------|
| ReadWriteOnce (RWO) | One node can read/write |
| ReadOnlyMany (ROX) | Many nodes read only |
| ReadWriteMany (RWX) | Many nodes read/write |
| ReadWriteOncePod (RWOP) | Single Pod only |

---

## Reclaim Policies

| Policy | Action |
|----------|---------|
| Delete | Delete storage |
| Retain | Keep storage |
| Recycle *(deprecated)* | Basic cleanup |

---

# 7. Security

## Security Layers

```text
Authentication

↓

Authorization (RBAC)

↓

Admission Controllers

↓

Pod Security
```

---

## RBAC Objects

| Object | Scope |
|---------|-------|
| Role | Namespace |
| RoleBinding | Namespace |
| ClusterRole | Cluster |
| ClusterRoleBinding | Cluster |

---

## Authentication vs Authorization

| Authentication | Authorization |
|----------------|--------------|
| Who are you? | What can you do? |

---

## Service Accounts

| Purpose |
|----------|
| Identity for Pods |

---

## Secrets vs ConfigMaps

| ConfigMap | Secret |
|------------|--------|
| Plain configuration | Sensitive data |
| Not encrypted by default | Base64 encoded (optionally encrypted at rest) |

---

## Security Components

| Component | Responsibility |
|----------|----------------|
| RBAC | Access control |
| ServiceAccount | Pod identity |
| Secret | Sensitive information |
| Admission Controller | Validate & mutate requests |
| Pod Security | Restrict Pod capabilities |

---

# 8. Resource Management

## Requests vs Limits

| Requests | Limits |
|-----------|--------|
| Minimum guaranteed resources | Maximum allowed resources |

---

## QoS Classes

| QoS | Condition | Eviction Priority |
|-----|-----------|-------------------|
| Guaranteed | Requests = Limits | Last |
| Burstable | Requests < Limits | Medium |
| BestEffort | No requests/limits | First |

---

## Common Resource Issues

| Problem | Cause |
|----------|-------|
| OOMKilled | Memory limit exceeded |
| CPU Throttling | CPU limit reached |
| Pending | Insufficient resources |
| Evicted | Node under pressure |

---

## Autoscaling

| Component | Scales |
|-----------|--------|
| HPA | Pods |
| VPA | CPU/Memory requests |
| Cluster Autoscaler | Nodes |

---

## Eviction Order

```text
BestEffort

↓

Burstable

↓

Guaranteed
```

---

## Resource Monitoring

```bash
kubectl top pod

kubectl top node
```

---

## Resource Best Practices

- Always define CPU and memory **requests**
- Always define CPU and memory **limits**
- Avoid `BestEffort` Pods in production
- Use HPA for stateless workloads
- Monitor CPU throttling and OOMKilled events

---

# 9. Troubleshooting & Debugging

## Universal Debugging Workflow

```text
Incident

↓

Observe the Symptom

↓

Identify the Failed Component

↓

Collect Evidence

↓

Build Hypothesis

↓

Verify

↓

Find Root Cause

↓

Fix

↓

Validate

↓

Document RCA
```

---

## Failure Domains

| Component | Typical Problems |
|------------|------------------|
| Control Plane | API Server unavailable, etcd issues |
| Node | NotReady, Disk Pressure, Memory Pressure |
| Scheduler | Pending Pods |
| kubelet | Pods not starting |
| Container Runtime | Container creation failures |
| Networking | Service unreachable, DNS failures |
| Storage | PVC Pending, Mount failures |
| Application | CrashLoopBackOff, OOMKilled |

---

## Investigation Order

```text
1. Pod Status

↓

2. Events

↓

3. Describe

↓

4. Logs

↓

5. Resources

↓

6. Configuration

↓

7. Network

↓

8. Storage
```

---

## Most Useful Debugging Commands

```bash
kubectl get pods

kubectl describe pod <pod>

kubectl logs <pod>

kubectl logs <pod> --previous

kubectl exec -it <pod> -- sh

kubectl get events --sort-by=.metadata.creationTimestamp

kubectl top pod

kubectl top node

kubectl describe node <node>

kubectl get endpoints

kubectl get endpointslices

kubectl describe service <service>

kubectl debug <pod>
```

---

## Common Pod States

| Status | Meaning |
|----------|---------|
| Pending | Waiting for scheduling |
| ContainerCreating | Container is being created |
| Running | Healthy |
| Completed | Job finished successfully |
| Succeeded | Successfully completed |
| Failed | Pod terminated with failure |
| Unknown | Node communication lost |
| Terminating | Pod shutting down |

---

## Common Errors

| Error | Meaning | First Thing to Check |
|--------|----------|----------------------|
| CrashLoopBackOff | App repeatedly crashes | Logs |
| ImagePullBackOff | Image pull failed | Image name / Registry |
| ErrImagePull | Image unavailable | Registry access |
| Pending | Scheduler couldn't place Pod | Events |
| OOMKilled | Memory exceeded | Limits |
| Evicted | Node pressure | Node status |
| CreateContainerConfigError | ConfigMap / Secret missing | Describe |
| CreateContainerError | Runtime/container issue | Events |
| NodeNotReady | Node unavailable | Node status |

---

## Quick Investigation Commands

| Problem | Command |
|----------|----------|
| Pod won't start | describe pod |
| Application crashing | logs |
| Scheduling issue | get events |
| Service unreachable | describe svc |
| DNS issue | exec nslookup |
| PVC issue | describe pvc |
| Node issue | describe node |
| High CPU | top pod |
| High Memory | top pod |

---

# 10. Production Operations

## Rolling Updates

```bash
kubectl rollout status deployment <deployment>

kubectl rollout history deployment <deployment>

kubectl rollout restart deployment <deployment>

kubectl rollout undo deployment <deployment>

kubectl rollout pause deployment <deployment>

kubectl rollout resume deployment <deployment>
```

---

## Scaling

```bash
kubectl scale deployment <deployment> --replicas=5
```

---

## Node Maintenance

Cordon (prevent scheduling)

```bash
kubectl cordon <node>
```

Drain (evict workloads)

```bash
kubectl drain <node> --ignore-daemonsets
```

Enable scheduling again

```bash
kubectl uncordon <node>
```

---

## Labels

```bash
kubectl label node worker-1 gpu=true
```

---

## Remove Label

```bash
kubectl label node worker-1 gpu-
```

---

## Taints

```bash
kubectl taint nodes worker-1 gpu=true:NoSchedule
```

---

## Remove Taint

```bash
kubectl taint nodes worker-1 gpu=true:NoSchedule-
```

---

## Port Forward

```bash
kubectl port-forward pod/<pod> 8080:80
```

---

## Copy Files

```bash
kubectl cp pod:/tmp/file .
```

---

## Exec

```bash
kubectl exec -it <pod> -- bash
```

---

# 11. Interview Nuggets

## Controllers

- Deployment manages ReplicaSets.
- ReplicaSet manages Pods.
- StatefulSet manages stateful Pods.
- DaemonSet runs one Pod per Node.
- CronJob creates Jobs.
- Job creates Pods.

---

## Scheduling

- Scheduler assigns a Pod to a Node **only once**.
- kubelet continuously reconciles desired state.
- Taints protect Nodes.
- Tolerations allow Pods onto tainted Nodes.
- nodeSelector performs exact label matching.
- Node Affinity supports advanced scheduling rules.
- Pod Anti-Affinity helps distribute replicas.

---

## Networking

- Services route traffic using **EndpointSlices**.
- kube-proxy implements Service networking.
- CoreDNS provides service discovery.
- Ingress only manages HTTP/HTTPS traffic.
- Ingress requires an Ingress Controller.

---

## Storage

- PVC requests storage.
- PV provides storage.
- StorageClass enables dynamic provisioning.
- StatefulSets commonly use PVCs.

---

## Security

- ConfigMaps are not for secrets.
- Secrets are Base64 encoded (not encrypted by default).
- RBAC controls permissions.
- ServiceAccounts provide Pod identities.

---

## Resources

- Requests determine scheduling.
- Limits prevent excessive resource usage.
- Guaranteed Pods are evicted last.
- BestEffort Pods are evicted first.

---

## Architecture

- API Server is the cluster entry point.
- etcd is the source of truth.
- Controller Manager continuously reconciles cluster state.
- kubelet manages containers on a Node.
- kube-proxy is responsible for Service networking.

---

# 12. One-Line Definitions

| Component | Definition |
|------------|------------|
| Pod | Smallest deployable unit in Kubernetes |
| ReplicaSet | Maintains a desired number of Pod replicas |
| Deployment | Manages ReplicaSets and rolling updates |
| StatefulSet | Manages stateful applications with stable identity |
| DaemonSet | Ensures one Pod runs on every Node |
| Job | Runs a task until completion |
| CronJob | Schedules Jobs |
| Service | Stable endpoint for accessing Pods |
| Ingress | HTTP/HTTPS routing into the cluster |
| ConfigMap | Stores non-sensitive configuration |
| Secret | Stores sensitive configuration |
| Namespace | Logical isolation within a cluster |
| Node | Machine that runs Pods |
| kubelet | Agent responsible for managing Pods on a Node |
| kube-proxy | Implements Service networking |
| Scheduler | Chooses the Node for new Pods |
| API Server | Entry point for the Kubernetes API |
| etcd | Distributed key-value store for cluster state |
| Controller Manager | Reconciles desired and actual cluster state |
| EndpointSlice | Stores backend Pod endpoints for Services |
| PV | Persistent storage resource |
| PVC | Request for persistent storage |
| StorageClass | Defines how storage is provisioned |
| Ingress Controller | Implements Ingress rules |
| CoreDNS | Provides DNS for the cluster |
| Container Runtime | Runs containers on Nodes |

---

## Kubernetes Component Responsibilities

| Component | Responsibility |
|------------|----------------|
| API Server | Accepts all Kubernetes API requests |
| Scheduler | Selects the best Node for new Pods |
| Controller Manager | Maintains desired cluster state |
| etcd | Stores cluster configuration and state |
| kubelet | Creates and monitors Pods on a Node |
| kube-proxy | Manages networking rules for Services |
| CoreDNS | Resolves Service DNS names |
| Container Runtime | Pulls images and runs containers |

---

# 13. Most Asked Interview Comparisons

## Pod vs Container

| Pod | Container |
|-----|-----------|
| Smallest deployable unit | Runs an application |
| Can contain one or more containers | Single running process |
| Has its own IP | Shares Pod IP |
| Shares volumes and network | Uses Pod resources |

---

## Deployment vs ReplicaSet

| Deployment | ReplicaSet |
|------------|------------|
| Manages ReplicaSets | Manages Pods |
| Supports rolling updates | No rolling updates |
| Supports rollbacks | No rollback support |
| Recommended for applications | Rarely created directly |

---

## Deployment vs StatefulSet

| Deployment | StatefulSet |
|------------|-------------|
| Stateless applications | Stateful applications |
| Pods are interchangeable | Stable Pod identity |
| Random Pod names | Predictable Pod names |
| Shared storage | Dedicated storage per Pod |
| Parallel deployment | Ordered deployment |

---

## Deployment vs DaemonSet

| Deployment | DaemonSet |
|------------|-----------|
| Fixed number of replicas | One Pod per Node |
| Used for applications | Used for agents |
| Supports scaling | No replicas field |

---

## Job vs CronJob

| Job | CronJob |
|-----|----------|
| Runs once | Runs on schedule |
| Executes until completion | Creates Jobs periodically |
| Manual execution | Automatic execution |

---

## Service vs Ingress

| Service | Ingress |
|----------|---------|
| Exposes Pods | Routes HTTP/HTTPS traffic |
| L4 Networking | L7 Networking |
| Internal or external access | External web traffic |
| Works without Ingress Controller | Requires Ingress Controller |

---

## ClusterIP vs NodePort vs LoadBalancer

| ClusterIP | NodePort | LoadBalancer |
|------------|----------|--------------|
| Internal | Node IP + Port | Cloud Load Balancer |
| Default Service | Development | Production |
| Not externally accessible | Limited external access | Public access |

---

## ConfigMap vs Secret

| ConfigMap | Secret |
|------------|--------|
| Non-sensitive data | Sensitive data |
| Plain text | Base64 encoded |
| Configuration | Passwords, Tokens, Keys |

---

## PV vs PVC

| PV | PVC |
|----|-----|
| Actual storage | Storage request |
| Created by admin or dynamically | Created by user |
| Supplies storage | Consumes storage |

---

## Requests vs Limits

| Requests | Limits |
|-----------|--------|
| Minimum guaranteed resources | Maximum allowed resources |
| Used by Scheduler | Enforced by kubelet |
| Determines scheduling | Prevents resource overuse |

---

## Labels vs Annotations

| Labels | Annotations |
|---------|-------------|
| Used for selection | Metadata only |
| Small and queryable | Large descriptive data |
| Used by selectors | Cannot be used by selectors |

---

## nodeSelector vs Node Affinity

| nodeSelector | Node Affinity |
|--------------|---------------|
| Exact label matching | Advanced scheduling |
| Simple | Flexible |
| AND logic only | AND, OR, operators |
| No preferences | Preferred & Required rules |

---

## Taints vs Tolerations

| Taints | Tolerations |
|---------|-------------|
| Applied to Nodes | Applied to Pods |
| Repels Pods | Allows scheduling |
| Protects Nodes | Grants permission |

---

## Pod Affinity vs Pod Anti-Affinity

| Pod Affinity | Pod Anti-Affinity |
|--------------|-------------------|
| Schedule Pods together | Keep Pods apart |
| Improve locality | Improve availability |

---

## Role vs ClusterRole

| Role | ClusterRole |
|------|-------------|
| Namespace scoped | Cluster scoped |
| Access within one namespace | Access across cluster |

---

## RoleBinding vs ClusterRoleBinding

| RoleBinding | ClusterRoleBinding |
|--------------|--------------------|
| Grants permissions in a namespace | Grants permissions cluster-wide |

---

## Authentication vs Authorization

| Authentication | Authorization |
|----------------|---------------|
| Who are you? | What are you allowed to do? |

---

## Liveness vs Readiness vs Startup Probe

| Liveness | Readiness | Startup |
|-----------|-----------|----------|
| Detects dead containers | Controls traffic | Handles slow-starting applications |
| Restarts container | Removes Pod from Service | Prevents premature restarts |

---

## emptyDir vs hostPath vs PVC

| emptyDir | hostPath | PVC |
|-----------|----------|-----|
| Temporary | Node filesystem | Persistent storage |
| Deleted with Pod | Tied to Node | Independent of Pod |

---

## HPA vs VPA vs Cluster Autoscaler

| HPA | VPA | Cluster Autoscaler |
|-----|-----|--------------------|
| Scales Pods | Changes resource requests | Scales Nodes |
| Horizontal scaling | Vertical scaling | Infrastructure scaling |

---

## Rolling Update vs Recreate

| Rolling Update | Recreate |
|----------------|-----------|
| Zero/minimal downtime | Full downtime |
| Gradual replacement | All Pods replaced at once |
| Default Deployment strategy | Suitable for incompatible versions |

---

# Kubernetes Interview Revision (30 Seconds)

## Workload Hierarchy

```text
Deployment
        │
ReplicaSet
        │
      Pods
        │
   Containers
```

---

## Networking

```text
Ingress
    │
Service
    │
EndpointSlice
    │
   Pods
```

---

## Storage

```text
Pod
 │
PVC
 │
PV
 │
StorageClass
 │
Disk
```

---

## Scheduling

```text
Pod

↓

Scheduler

↓

Best Node

↓

kubelet

↓

Container Runtime
```

---

## Debugging Order

```text
Status

↓

Events

↓

Describe

↓

Logs

↓

Resources

↓

Configuration

↓

Network

↓

Storage
```

---

> **End of Cheat Sheet**
