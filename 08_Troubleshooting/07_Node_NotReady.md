# 07 - Node NotReady

> **Objective:** Learn how to investigate, identify, remediate, and verify Kubernetes nodes that have entered the **NotReady** state.

---

# Table of Contents

1. What is a NotReady Node?
2. Node Lifecycle
3. Node Conditions
4. Common Causes
5. Symptoms
6. Investigation Workflow
7. Root Cause Analysis Commands
8. Common Root Cause Scenarios
9. Linux-Level Investigation
10. Fixes
11. Verification
12. Prevention
13. AI-SRE Investigation Flow
14. Interview Questions

---

# 1. What is a NotReady Node?

A node enters the **NotReady** state when the Kubernetes control plane determines that it is unhealthy or cannot reliably run workloads.

When a node becomes **NotReady**:

* New Pods are not scheduled.
* Existing Pods may eventually be evicted.
* Cluster capacity is reduced.
* High availability may be affected.

Unlike **Pending** or **CrashLoopBackOff**, this is a **node-level** infrastructure issue.

---

# 2. Node Lifecycle

```text
Worker Node Boots
        │
        ▼
Kubelet Starts
        │
        ▼
Registers with API Server
        │
        ▼
Heartbeat Sent
        │
        ▼
Node Ready
        │
        ▼
Heartbeat Stops / Node Fails
        │
        ▼
Node NotReady
```

---

# 3. Node Conditions

Each node reports several health conditions.

```bash
kubectl describe node worker-1
```

Example

```text
Conditions:

Ready              True

MemoryPressure     False

DiskPressure       False

PIDPressure        False

NetworkUnavailable False
```

Meaning

| Condition          | Description                   |
| ------------------ | ----------------------------- |
| Ready              | Node can run Pods             |
| MemoryPressure     | Node is low on memory         |
| DiskPressure       | Node storage is nearly full   |
| PIDPressure        | Too many running processes    |
| NetworkUnavailable | Networking is not functioning |

---

# 4. Common Causes

## Kubelet Failure

* kubelet service stopped
* kubelet crash
* kubelet configuration error

---

## Container Runtime Failure

* containerd stopped
* Docker stopped
* CRI-O failure

---

## Network Problems

* CNI plugin failure
* Node cannot reach API Server
* Firewall issues

---

## Resource Problems

* Disk full
* Memory exhausted
* CPU starvation

---

## Certificate Problems

* kubelet certificate expired
* authentication failure

---

## Cloud Infrastructure

* VM rebooted
* Instance terminated
* Network interface failure

---

# 5. Symptoms

```bash
kubectl get nodes
```

Output

```text
NAME        STATUS

worker-1    Ready

worker-2    NotReady

worker-3    Ready
```

---

# 6. Investigation Workflow

```text
kubectl get nodes
        │
        ▼
kubectl describe node
        │
        ▼
Review Conditions
        │
        ▼
Review Events
        │
        ▼
SSH into Node
        │
        ▼
Check kubelet
        │
        ▼
Check container runtime
        │
        ▼
Check CNI
        │
        ▼
Check disk & memory
        │
        ▼
Restore Node
        │
        ▼
Verify Ready
```

---

# 7. Root Cause Analysis Commands

## Step 1 – Check Nodes

```bash
kubectl get nodes
```

---

## Step 2 – Describe Node

```bash
kubectl describe node worker-2
```

Check

* Conditions
* Allocatable Resources
* Events

---

## Step 3 – Check Events

```bash
kubectl get events \
--sort-by=.metadata.creationTimestamp
```

Look for

* NodeNotReady
* NodeHasDiskPressure
* NodeHasMemoryPressure

---

## Step 4 – View Node Metrics

```bash
kubectl top nodes
```

Determine whether the node is resource constrained.

---

# 8. Common Root Cause Scenarios

---

## Scenario 1 – kubelet Stopped

Symptoms

```text
Ready=False
```

SSH

```bash
systemctl status kubelet
```

Output

```text
inactive (dead)
```

Root Cause

kubelet service stopped.

Fix

```bash
sudo systemctl restart kubelet
```

---

## Scenario 2 – containerd Failure

Check

```bash
systemctl status containerd
```

Output

```text
failed
```

Fix

```bash
sudo systemctl restart containerd
```

---

## Scenario 3 – Disk Full

Linux

```bash
df -h
```

Output

```text
Filesystem

100% Used
```

Root Cause

No disk space available.

---

## Scenario 4 – Memory Pressure

```bash
free -h
```

Shows

Very little available memory.

---

## Scenario 5 – API Server Unreachable

Test

```bash
curl -k https://<api-server>:6443
```

Fails.

Root Cause

Network connectivity issue.

---

## Scenario 6 – CNI Failure

Pods remain

```text
ContainerCreating
```

Node

```text
NotReady
```

Check

```bash
kubectl get pods -n kube-system
```

Look for

* calico
* cilium
* flannel

---

## Scenario 7 – Certificate Expired

Logs

```bash
journalctl -u kubelet
```

Output

```text
certificate has expired
```

---

# 9. Linux-Level Investigation

Once logged into the node, investigate:

Check kubelet

```bash
systemctl status kubelet
```

Restart

```bash
sudo systemctl restart kubelet
```

---

View kubelet logs

```bash
journalctl -u kubelet -n 100
```

---

Check runtime

```bash
crictl ps
```

---

Runtime information

```bash
crictl info
```

---

Disk

```bash
df -h
```

---

Memory

```bash
free -h
```

---

Processes

```bash
top
```

or

```bash
htop
```

---

Network

```bash
ip addr
```

```bash
ip route
```

---

Connectivity

```bash
ping <api-server>
```

---

# 10. Fixes

Possible remediations

* Restart kubelet
* Restart container runtime
* Clean disk
* Free memory
* Fix certificates
* Repair CNI
* Restore network connectivity
* Reboot node (last resort)

---

# 11. Verification

Verify

```bash
kubectl get nodes
```

Expected

```text
worker-2

Ready
```

Verify workloads

```bash
kubectl get pods -A -o wide
```

Pods should begin scheduling on the node again.

---

# 12. Prevention

* Monitor node health.
* Alert on Ready=False.
* Monitor disk utilization.
* Monitor kubelet service.
* Rotate certificates before expiry.
* Keep CNI components healthy.
* Use Cluster Autoscaler.
* Configure node monitoring with Prometheus.

---

# 13. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── Node Status
        ├── Conditions
        ├── Events
        ├── Metrics
        ├── kubelet Logs
        ├── Runtime Status
        ├── Disk
        ├── Memory
        └── Network
                │
                ▼
Node NotReady Playbook
                │
                ▼
Adaptive Investigation
                │
                ▼
LLM Root Cause Analysis
                │
                ▼
Risk Assessment
                │
                ▼
Approval
                │
                ▼
Execute Fix
                │
                ▼
Verification
```

---

# 14. Interview Questions

### 1. What causes a node to become NotReady?

A node becomes **NotReady** when it stops sending healthy heartbeats to the control plane or reports unhealthy conditions such as kubelet failure, network issues, or severe resource pressure.

---

### 2. Which command should you run first?

```bash
kubectl get nodes
```

---

### 3. Which command provides the most detailed node information?

```bash
kubectl describe node <node-name>
```

---

### 4. Which Linux service is most critical?

The **kubelet** service, because it communicates with the control plane and manages Pods on the node.

---

### 5. How do you check whether the container runtime is healthy?

```bash
systemctl status containerd
```

or

```bash
crictl info
```

---

### 6. Can a node be NotReady even if the kubelet is running?

Yes. Network failures, certificate issues, container runtime failures, or severe resource pressure can still prevent the node from becoming Ready.

---

### 7. What is the last-resort fix?

Reboot the node only after investigating and addressing the underlying cause, as rebooting alone may only provide a temporary recovery.

---

### 8. How do you verify the issue is resolved?

* `kubectl get nodes` reports the node as `Ready`.
* Node conditions are healthy.
* Pods are successfully scheduled.
* No new `NodeNotReady` events appear.
