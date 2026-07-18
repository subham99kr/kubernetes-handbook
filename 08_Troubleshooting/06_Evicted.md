# 06 - Evicted

> **Objective:** Learn how to investigate, identify, remediate, and verify Kubernetes Pods that have been **Evicted** due to node resource pressure.

---

# Table of Contents

1. What is an Evicted Pod?
2. How Pod Eviction Works
3. Difference Between Evicted and OOMKilled
4. Common Causes
5. Symptoms
6. Investigation Workflow
7. Root Cause Analysis Commands
8. Common Root Cause Scenarios
9. Fixes
10. Verification
11. Prevention
12. AI-SRE Investigation Flow
13. Interview Questions

---

# 1. What is an Evicted Pod?

An **Evicted** Pod is one that Kubernetes forcibly removes from a node because the **node is under resource pressure**.

Unlike **OOMKilled**, where the Linux kernel kills a container, **Eviction** is initiated by the **Kubelet**.

The application itself may be perfectly healthy.

---

# 2. How Pod Eviction Works

When a node experiences resource pressure:

* MemoryPressure
* DiskPressure
* PIDPressure

the kubelet attempts to reclaim resources.

It starts by:

1. Garbage collecting unused images
2. Removing dead containers
3. Evicting lower-priority Pods

Flow

```text
Node Running Normally
        │
        ▼
Resource Usage Increases
        │
        ▼
MemoryPressure / DiskPressure / PIDPressure
        │
        ▼
Kubelet Detects Pressure
        │
        ▼
Garbage Collection
        │
        ▼
Eviction Threshold Still Exceeded
        │
        ▼
Pods Evicted
```

---

# 3. Evicted vs OOMKilled

| OOMKilled                       | Evicted                |
| ------------------------------- | ---------------------- |
| Container-level                 | Node-level             |
| Linux Kernel                    | Kubelet                |
| Exceeded container memory limit | Node resource pressure |
| Exit Code 137                   | Pod Status = Evicted   |
| Application problem             | Infrastructure problem |

---

# 4. Common Causes

## Memory Pressure

Node RAM exhausted.

---

## Disk Pressure

* Image filesystem full
* Log files consuming storage
* Ephemeral storage exhausted

---

## PID Pressure

Too many running processes.

---

## Ephemeral Storage

Applications writing excessive temporary files.

---

## Low Priority Pods

During pressure Kubernetes evicts lower-priority Pods first.

---

# 5. Symptoms

```bash
kubectl get pods
```

Example

```text
NAME         READY   STATUS     RESTARTS   AGE

frontend     0/1     Evicted    0          2h
```

Unlike CrashLoopBackOff, the Pod has already been terminated.

---

# 6. Investigation Workflow

```text
kubectl get pods
        │
        ▼
kubectl describe pod
        │
        ▼
Read Eviction Message
        │
        ▼
Check Node Conditions
        │
        ▼
Check Node Resource Usage
        │
        ▼
Determine Resource Pressure
        │
        ▼
Fix Node
        │
        ▼
Redeploy Pod
```

---

# 7. Root Cause Analysis Commands

## Step 1 – Check Pod

```bash
kubectl get pods
```

---

## Step 2 – Describe Pod

```bash
kubectl describe pod frontend
```

Example

```text
Status: Failed

Reason: Evicted

Message:

The node had condition:

MemoryPressure
```

This immediately identifies the resource that triggered eviction.

---

## Step 3 – Check Node

```bash
kubectl get nodes
```

---

## Step 4 – Describe Node

```bash
kubectl describe node worker-1
```

Look for:

```text
Conditions

MemoryPressure=True

DiskPressure=False

PIDPressure=False
```

---

## Step 5 – Check Node Usage

```bash
kubectl top nodes
```

Example

```text
NAME

CPU%

MEMORY%

worker-1

88

99
```

Memory is nearly exhausted.

---

## Step 6 – Check Pod Usage

```bash
kubectl top pods
```

Identify workloads consuming excessive memory.

---

## Step 7 – Check Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for eviction-related events.

---

# 8. Root Cause Scenarios

---

## Scenario 1 – Memory Pressure

Node

```text
MemoryPressure=True
```

Root Cause

Node RAM exhausted.

Fix

* Add memory
* Scale cluster
* Remove unnecessary Pods

---

## Scenario 2 – Disk Pressure

```text
DiskPressure=True
```

Node storage full.

Common reasons

* Huge log files
* Old images
* Container filesystem growth

---

## Scenario 3 – PID Pressure

```text
PIDPressure=True
```

Too many processes running.

Usually caused by:

* Fork bombs
* Misbehaving applications
* Excessive containers

---

## Scenario 4 – Ephemeral Storage

Events

```text
Pod ephemeral local storage usage exceeds limit
```

Root Cause

Application writes too many temporary files.

---

## Scenario 5 – Low Priority Pod

Higher priority workloads scheduled.

Lower priority Pods evicted.

---

# 9. Fixes

Possible remediations

* Increase node memory
* Increase disk capacity
* Clean unused images
* Remove old logs
* Configure log rotation
* Increase ephemeral storage
* Scale the cluster
* Tune resource requests
* Assign Pod priorities

Delete evicted Pods

```bash
kubectl delete pod frontend
```

Deployment automatically creates a replacement Pod.

---

# 10. Verification

Verify node

```bash
kubectl describe node worker-1
```

Expected

```text
MemoryPressure=False

DiskPressure=False

PIDPressure=False
```

Verify replacement Pod

```bash
kubectl get pods
```

Expected

```text
frontend

Running
```

---

# 11. Prevention

* Monitor node utilization.
* Configure resource requests and limits.
* Enable log rotation.
* Clean unused container images.
* Monitor ephemeral storage.
* Use Pod PriorityClasses appropriately.
* Configure Cluster Autoscaler.

---

# 12. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── Pod Status
        ├── Eviction Message
        ├── Node Conditions
        ├── Node Metrics
        ├── Pod Metrics
        ├── Events
        └── Resource Configuration
                │
                ▼
Evicted Playbook
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

# 13. Interview Questions

### 1. What is an Evicted Pod?

A Pod that Kubernetes removed from a node because the node experienced resource pressure.

---

### 2. What is the difference between Evicted and OOMKilled?

* **OOMKilled** is triggered by the Linux kernel when a container exceeds its memory limit.
* **Evicted** is initiated by the kubelet when the node itself is under resource pressure.

---

### 3. Which command identifies why a Pod was evicted?

```bash
kubectl describe pod <pod-name>
```

The `Message` field typically states whether the eviction was due to `MemoryPressure`, `DiskPressure`, or another resource issue.

---

### 4. What node conditions can trigger eviction?

* `MemoryPressure`
* `DiskPressure`
* `PIDPressure`

---

### 5. Can Kubernetes automatically recreate an evicted Pod?

Yes, if the Pod is managed by a higher-level controller such as a Deployment, ReplicaSet, StatefulSet, or DaemonSet.

---

### 6. How do you verify the node has recovered?

Run:

```bash
kubectl describe node <node-name>
```

Ensure all pressure conditions (`MemoryPressure`, `DiskPressure`, `PIDPressure`) are `False`.

---

### 7. How can you reduce the chances of Pod eviction?

* Configure realistic resource requests and limits.
* Monitor node resource utilization.
* Enable Cluster Autoscaler.
* Manage log growth and ephemeral storage.
* Regularly clean unused container images.
