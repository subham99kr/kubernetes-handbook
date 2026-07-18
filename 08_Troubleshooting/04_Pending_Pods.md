# 04 - Pending Pods

> **Objective:** Learn how to investigate, identify, remediate, and verify Kubernetes Pods that remain in the **Pending** state.

---

# Table of Contents

1. What is a Pending Pod?
2. Pod Scheduling Lifecycle
3. Common Causes
4. Symptoms
5. Investigation Workflow
6. Root Cause Analysis Commands
7. Common Root Cause Scenarios
8. Fixes
9. Verification
10. Prevention
11. AI-SRE Investigation Flow
12. Interview Questions

---

# 1. What is a Pending Pod?

A Pod is in the **Pending** state when it has been accepted by the Kubernetes API Server but **cannot yet be scheduled onto a node or cannot complete initialization**.

Unlike **CrashLoopBackOff**, the application has **never started**.

---

# 2. Pod Scheduling Lifecycle

```text
kubectl apply deployment.yaml
          │
          ▼
API Server Creates Pod
          │
          ▼
Scheduler Selects Node
          │
          ▼
Node Accepts Pod
          │
          ▼
Images Pulled
          │
          ▼
Containers Started
```

A Pod remains **Pending** if any step after creation cannot be completed.

---

# 3. Common Causes

## Resource Constraints

* Insufficient CPU
* Insufficient Memory
* Insufficient Ephemeral Storage
* GPU unavailable

---

## Scheduling Constraints

* Node Selector mismatch
* Node Affinity mismatch
* Pod Affinity rules
* Pod Anti-Affinity rules

---

## Node Issues

* All nodes NotReady
* Nodes cordoned
* Nodes drained

---

## Taints & Tolerations

* Untolerated taints
* Dedicated nodes
* GPU taints

---

## Storage Issues

* PVC Pending
* StorageClass unavailable
* Volume binding failures

---

## Namespace Restrictions

* ResourceQuota exceeded
* LimitRange restrictions

---

# 4. Symptoms

```bash
kubectl get pods
```

Example

```text
NAME           READY   STATUS    RESTARTS   AGE
frontend       0/1     Pending   0          5m
```

Notice:

* Restart Count = 0
* No container has started.

---

# 5. Investigation Workflow

```text
kubectl get pods
        │
        ▼
kubectl describe pod
        │
        ▼
Read Scheduling Events
        │
        ▼
Check Nodes
        │
        ▼
Check Resources
        │
        ▼
Check Taints
        │
        ▼
Check Affinity
        │
        ▼
Check PVC
        │
        ▼
Check ResourceQuota
        │
        ▼
Apply Fix
        │
        ▼
Verify Scheduling
```

---

# 6. Root Cause Analysis Commands

## Step 1 – Check Pod

```bash
kubectl get pods
```

Output

```text
NAME         READY   STATUS
frontend     0/1     Pending
```

---

## Step 2 – Describe Pod

```bash
kubectl describe pod frontend
```

The **Events** section usually contains the root cause.

Example

```text
Warning  FailedScheduling

0/3 nodes are available:

Insufficient memory.
```

Interpretation

The scheduler could not find any node with enough free memory.

---

## Step 3 – Check Nodes

```bash
kubectl get nodes
```

Output

```text
NAME        STATUS
worker-1    Ready
worker-2    Ready
worker-3    Ready
```

If a node is `NotReady`, investigate the node.

---

## Step 4 – Check Resource Usage

```bash
kubectl top nodes
```

Example

```text
NAME       CPU%   MEMORY%

worker-1    94      97
worker-2    98      95
worker-3    91      96
```

Interpretation

The cluster has almost no remaining capacity.

---

## Step 5 – Check Node Capacity

```bash
kubectl describe node worker-1
```

Review

* Allocatable CPU
* Allocatable Memory
* Running Pods

---

## Step 6 – Check Taints

```bash
kubectl describe node worker-1
```

Example

```text
Taints:

gpu=true:NoSchedule
```

If the Pod lacks a matching toleration, scheduling fails.

---

## Step 7 – Check Node Selector

```bash
kubectl get pod frontend -o yaml
```

Example

```yaml
nodeSelector:
  disktype: ssd
```

Verify at least one node has:

```text
disktype=ssd
```

---

## Step 8 – Check Node Labels

```bash
kubectl get nodes --show-labels
```

Verify required labels exist.

---

## Step 9 – Check PVC

```bash
kubectl get pvc
```

Example

```text
NAME      STATUS

db-data   Pending
```

Storage is preventing scheduling.

---

## Step 10 – Check Resource Quotas

```bash
kubectl describe quota
```

Example

```text
Used CPU

4

Hard CPU

4
```

Namespace has reached its CPU quota.

---

# 7. Root Cause Scenarios

---

## Scenario 1 – Insufficient CPU

Events

```text
0/3 nodes are available

Insufficient cpu
```

Fix

* Scale cluster
* Reduce CPU requests
* Remove unnecessary workloads

---

## Scenario 2 – Insufficient Memory

Events

```text
Insufficient memory
```

Fix

Increase node capacity or reduce requests.

---

## Scenario 3 – Node Selector Mismatch

Deployment

```yaml
nodeSelector:

disktype: ssd
```

Node Labels

```text
disktype=hdd
```

Root Cause

No matching nodes.

---

## Scenario 4 – Untolerated Taint

Node

```text
gpu=true:NoSchedule
```

Pod

No toleration defined.

Result

Scheduler rejects the node.

---

## Scenario 5 – PVC Pending

```text
PersistentVolumeClaim

Pending
```

Root Cause

Storage unavailable.

---

## Scenario 6 – ResourceQuota Exceeded

Events

```text
exceeded quota
```

Root Cause

Namespace limits reached.

---

## Scenario 7 – Node NotReady

```text
worker-2

NotReady
```

Scheduler avoids unhealthy nodes.

---

# 8. Fixes

Common fixes include:

* Add more nodes
* Reduce resource requests
* Fix node labels
* Add tolerations
* Remove unnecessary affinity rules
* Provision PersistentVolumes
* Increase ResourceQuota
* Uncordon nodes

Example

```bash
kubectl uncordon worker-1
```

---

# 9. Verification

Verify Pod

```bash
kubectl get pods
```

Expected

```text
NAME        READY   STATUS

frontend    1/1     Running
```

Verify Node Assignment

```bash
kubectl get pod -o wide
```

Ensure the Pod has been assigned to a node.

---

# 10. Prevention

* Monitor cluster utilization.
* Set realistic resource requests.
* Review affinity rules.
* Avoid unnecessary taints.
* Monitor ResourceQuota usage.
* Provision storage proactively.
* Maintain node capacity planning.

---

# 11. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── Pod
        ├── Scheduler Events
        ├── Nodes
        ├── Node Labels
        ├── Taints
        ├── Affinity Rules
        ├── PVC
        ├── ResourceQuota
        └── Metrics
                │
                ▼
Pending Pod Playbook
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

# 12. Interview Questions

### 1. What does the Pending state mean?

The Pod has been created but cannot yet be scheduled or initialized.

---

### 2. Which command is most useful?

```bash
kubectl describe pod <pod-name>
```

The Events section usually identifies the scheduling issue.

---

### 3. Can a Pod be Pending if all nodes are Ready?

Yes. Reasons include insufficient resources, taints, affinity mismatches, PVC issues, or namespace quotas.

---

### 4. What is the difference between a nodeSelector and a taint?

* **nodeSelector** expresses where a Pod *wants* to run by matching node labels.
* **Taints** repel Pods unless they have a matching toleration.

---

### 5. What causes `0/3 nodes are available`?

The scheduler evaluated all nodes but none satisfied the Pod's scheduling requirements.

---

### 6. How do you check node resource utilization?

```bash
kubectl top nodes
```

---

### 7. How do you verify the issue is resolved?

* Pod transitions to `Running`
* Pod is assigned to a node (`kubectl get pod -o wide`)
* Scheduler no longer reports `FailedScheduling`
* Events no longer show scheduling failures
