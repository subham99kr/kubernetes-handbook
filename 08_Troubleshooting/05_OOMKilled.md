# 05 - OOMKilled

> **Objective:** Learn how to investigate, identify, remediate, and verify Kubernetes containers terminated due to **Out of Memory (OOM)** conditions.

---

# Table of Contents

1. What is OOMKilled?
2. Linux OOM Killer
3. Memory Requests vs Limits
4. QoS Classes
5. Common Causes
6. Symptoms
7. Investigation Workflow
8. Root Cause Analysis Commands
9. Common Root Cause Scenarios
10. Fixes
11. Verification
12. Prevention
13. AI-SRE Investigation Flow
14. Interview Questions

---

# 1. What is OOMKilled?

An **OOMKilled** event occurs when a container exceeds its configured **memory limit** and the Linux kernel terminates the process to protect the node.

Unlike CPU, **memory cannot be throttled**. If a process tries to use more memory than allowed, it is killed immediately.

The Pod may then restart depending on its restart policy, often resulting in a **CrashLoopBackOff**.

---

# 2. Linux OOM Killer

The Linux kernel continuously monitors available memory.

When memory becomes critically low:

1. The kernel calculates an **OOM score** for running processes.
2. The process most suitable for termination is selected.
3. The kernel sends **SIGKILL (signal 9)**.
4. Kubernetes reports the container as **OOMKilled**.

Flow:

```text
Container Starts
        │
        ▼
Memory Usage Increases
        │
        ▼
Memory Limit Exceeded
        │
        ▼
Linux OOM Killer Invoked
        │
        ▼
SIGKILL Sent
        │
        ▼
Container Terminated
        │
        ▼
Exit Code 137
        │
        ▼
Pod Restart
```

---

# 3. Memory Requests vs Limits

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

**Request**

* Guaranteed memory for scheduling.
* Used by the Kubernetes scheduler.

**Limit**

* Maximum memory the container can consume.
* Exceeding this limit results in **OOMKilled**.

**Important:** A Pod can be scheduled successfully based on its request but still be OOMKilled later if it exceeds its limit.

---

# 4. QoS Classes

Kubernetes assigns each Pod a **Quality of Service (QoS)** class:

### Guaranteed

Requests = Limits for CPU and Memory.

Highest priority during memory pressure.

---

### Burstable

Requests < Limits.

Most common configuration.

---

### BestEffort

No requests or limits defined.

First to be terminated during node memory pressure.

Check QoS:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
QoS Class: Burstable
```

---

# 5. Common Causes

## Application

* Memory leak
* Infinite cache growth
* Loading huge datasets
* Unbounded threads

---

## JVM Applications

* Heap size larger than memory limit
* Incorrect `-Xmx` configuration

---

## Python Applications

* Large Pandas DataFrames
* Loading entire files into memory
* TensorFlow/PyTorch models too large

---

## Node.js

* Heap exhaustion
* Large JSON processing

---

## Infrastructure

* Memory limit too low
* Node memory pressure
* Too many containers

---

# 6. Symptoms

```bash
kubectl get pods
```

Output

```text
NAME         READY   STATUS             RESTARTS
frontend     0/1     CrashLoopBackOff   8
```

Describe Pod

```bash
kubectl describe pod frontend
```

Example

```text
Last State:

Terminated

Reason: OOMKilled

Exit Code: 137
```

This is the definitive indication that the process was terminated due to memory exhaustion.

---

# 7. Investigation Workflow

```text
kubectl get pods
        │
        ▼
kubectl describe pod
        │
        ▼
Confirm OOMKilled
        │
        ▼
Check Memory Limits
        │
        ▼
Check Resource Usage
        │
        ▼
Review Application Logs
        │
        ▼
Identify Memory Leak or Spike
        │
        ▼
Apply Fix
        │
        ▼
Verify Stability
```

---

# 8. Root Cause Analysis Commands

## Step 1 – Check Pod Status

```bash
kubectl get pods
```

---

## Step 2 – Describe Pod

```bash
kubectl describe pod frontend
```

Look for:

```text
Last State:

Reason: OOMKilled

Exit Code: 137
```

---

## Step 3 – View Logs

```bash
kubectl logs frontend --previous
```

The previous logs often show what the application was doing before it was killed.

---

## Step 4 – Check Resource Configuration

```bash
kubectl describe deployment frontend
```

Review:

```yaml
resources:
  requests:
    memory: 256Mi
  limits:
    memory: 512Mi
```

---

## Step 5 – Check Live Memory Usage

```bash
kubectl top pod
```

Example

```text
NAME         CPU   MEMORY
frontend     50m   498Mi
```

Memory is approaching the configured limit.

---

## Step 6 – Check Node Memory

```bash
kubectl top nodes
```

Determine whether the node itself is under memory pressure.

---

# 9. Root Cause Scenarios

## Scenario 1 – Memory Leak

Symptoms

* Memory usage continuously increases.
* Restarts occur after several hours.

Root Cause

Application leak.

Fix

Resolve the memory leak in the application.

---

## Scenario 2 – Memory Limit Too Small

Limit

```yaml
memory: 256Mi
```

Application requires:

```text
600Mi
```

Root Cause

Incorrect limit configuration.

Fix

Increase the memory limit.

---

## Scenario 3 – JVM Heap Too Large

Deployment

```bash
-Xmx2G
```

Container Limit

```yaml
memory: 1Gi
```

Root Cause

Heap exceeds container limit.

Fix

Reduce `-Xmx`.

---

## Scenario 4 – Large File Processing

Application loads a 4 GB CSV file into memory.

Root Cause

Application design.

Fix

Use streaming or chunked processing.

---

## Scenario 5 – Node Memory Pressure

Node

```bash
kubectl describe node
```

Shows:

```text
MemoryPressure=True
```

Root Cause

The node itself is under heavy memory pressure.

---

# 10. Fixes

Possible remediations:

* Increase memory limits.
* Optimize the application.
* Fix memory leaks.
* Reduce cache sizes.
* Configure JVM heap correctly.
* Stream large files.
* Scale horizontally.
* Add more nodes if node memory is exhausted.

---

# 11. Verification

Verify the Pod:

```bash
kubectl get pods
```

Expected

```text
NAME       READY   STATUS
frontend   1/1     Running
```

Check restart count:

```bash
kubectl describe pod frontend
```

Ensure the restart count is no longer increasing.

Monitor usage:

```bash
kubectl top pod
```

Memory should remain below the configured limit.

---

# 12. Prevention

* Always configure requests and limits.
* Monitor memory usage.
* Use Prometheus alerts for high memory consumption.
* Profile applications for leaks.
* Tune JVM heap sizes.
* Use Horizontal Pod Autoscaler when appropriate.
* Perform load testing before production deployments.

---

# 13. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── Pod Status
        ├── Container State
        ├── Exit Code
        ├── Previous Logs
        ├── Deployment
        ├── Resource Limits
        ├── Pod Metrics
        └── Node Metrics
                │
                ▼
OOMKilled Playbook
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

### 1. What does OOMKilled mean?

The Linux kernel terminated the container because it exceeded its memory limit.

---

### 2. Why is the exit code 137?

Exit code **137 = 128 + 9**, where **9** is the **SIGKILL** signal sent by the Linux kernel.

---

### 3. What is the difference between requests and limits?

* **Requests** are used for scheduling.
* **Limits** define the maximum resources a container may consume.

---

### 4. Can a Pod be OOMKilled even if the node has free memory?

Yes. A container exceeding its **own memory limit** will be killed even if the node has additional free memory.

---

### 5. Which QoS class is terminated first during node memory pressure?

**BestEffort**, followed by **Burstable**, while **Guaranteed** Pods are protected as much as possible.

---

### 6. Which command confirms an OOM kill?

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Reason: OOMKilled
Exit Code: 137
```

---

### 7. How do you verify the issue is resolved?

* Pod remains in the `Running` state.
* Restart count no longer increases.
* Memory usage stays below the configured limit.
* No further `OOMKilled` events appear.
