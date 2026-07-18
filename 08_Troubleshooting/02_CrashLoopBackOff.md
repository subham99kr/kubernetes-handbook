# 02 - CrashLoopBackOff

> **Objective:** Learn how to investigate, identify the root cause, remediate, and verify a Kubernetes pod stuck in the **CrashLoopBackOff** state.

---

# Table of Contents

1. What is CrashLoopBackOff?
2. How CrashLoopBackOff Works
3. Common Causes
4. Symptoms
5. Investigation Workflow
6. Commands for Root Cause Analysis
7. Understanding Exit Codes
8. Root Cause Scenarios
9. Fixes
10. Verification
11. Prevention
12. AI-SRE Investigation Flow
13. Interview Questions

---

# 1. What is CrashLoopBackOff?

A **CrashLoopBackOff** is a Kubernetes pod state indicating that a container repeatedly starts, crashes, and Kubernetes is delaying subsequent restart attempts using an exponential backoff algorithm.

It **does not** indicate the actual cause of the failure.

Instead, it indicates:

> Kubernetes attempted to restart the application several times, but it continues to fail.

---

## Example

```text
Application Starts
        │
        ▼
Application Crashes
        │
        ▼
RestartPolicy = Always
        │
        ▼
Kubernetes Restarts Container
        │
        ▼
Application Crashes Again
        │
        ▼
Restart Delay Increases
        │
        ▼
CrashLoopBackOff
```

---

# 2. How CrashLoopBackOff Works

Kubernetes follows this restart process:

1. Start the container.
2. Monitor its execution.
3. If it exits unexpectedly:

   * Restart it.
4. If repeated crashes occur:

   * Wait before restarting.
5. Increase the waiting time after each failure (Exponential Backoff).

Example:

| Restart |       Delay |
| ------- | ----------: |
| 1       | Immediately |
| 2       |  10 seconds |
| 3       |  20 seconds |
| 4       |  40 seconds |
| 5       |  80 seconds |
| ...     |         ... |

The purpose is to prevent continuous restart storms.

---

# 3. Common Causes

CrashLoopBackOff is **not** a root cause.

It is usually caused by one of the following:

## Application Issues

* Unhandled exception
* Segmentation fault
* Incorrect startup arguments
* Missing dependency
* Syntax error
* Runtime error

---

## Configuration Issues

* Missing ConfigMap
* Missing Secret
* Wrong environment variables
* Invalid configuration

---

## Infrastructure Issues

* Database unavailable
* DNS resolution failure
* Network connectivity problems

---

## Resource Issues

* Out Of Memory
* CPU starvation
* Storage unavailable

---

## Container Issues

* Wrong ENTRYPOINT
* Wrong CMD
* Missing executable
* Incorrect working directory

---

## Health Check Issues

* Liveness Probe failures
* Startup Probe failures
* Incorrect probe path
* Probe timeout

---

# 4. Symptoms

## Check Pods

```bash
kubectl get pods
```

Example

```text
NAME                           READY   STATUS             RESTARTS   AGE
frontend-7b5dd8b9c9-xpm4n      0/1     CrashLoopBackOff   9          6m
```

Interpretation

* Pod exists.
* Container starts.
* Container crashes.
* Kubernetes is delaying restarts.

---

# 5. Investigation Workflow

```text
kubectl get pods
        │
        ▼
kubectl describe pod
        │
        ▼
Check Events
        │
        ▼
Check Container State
        │
        ▼
Check Exit Code
        │
        ▼
View Previous Logs
        │
        ▼
Inspect Deployment
        │
        ▼
Inspect ConfigMaps
        │
        ▼
Inspect Secrets
        │
        ▼
Identify Root Cause
        │
        ▼
Apply Fix
        │
        ▼
Verify Recovery
```

---

# 6. Commands for Root Cause Analysis

---

## Step 1 – List Pods

```bash
kubectl get pods
```

Example

```text
NAME                        READY   STATUS             RESTARTS
frontend                    0/1     CrashLoopBackOff   12
```

Purpose

* Identify affected pods.
* Observe restart count.
* Confirm CrashLoopBackOff.

---

## Step 2 – Describe Pod

```bash
kubectl describe pod frontend
```

Important Sections

```text
State:
  Waiting
    Reason: CrashLoopBackOff

Last State:
  Terminated
    Reason: Error
    Exit Code: 1

Restart Count: 12
```

Interpretation

### State

Current container status.

### Last State

Previous execution.

Usually contains the actual reason.

### Exit Code

Most important clue.

### Restart Count

Higher values indicate repeated failures.

---

## Step 3 – Check Events

At the bottom:

```text
Events:

Back-off restarting failed container
```

Events help determine whether failures are due to:

* Scheduling
* Image Pull
* Mount failures
* Probe failures
* Resource exhaustion

---

## Step 4 – View Logs

```bash
kubectl logs frontend
```

If the pod has already restarted:

```bash
kubectl logs frontend --previous
```

Example

```text
Database connection failed.

Unable to connect to PostgreSQL.

Exiting...
```

Root Cause

Application cannot connect to database.

---

## Step 5 – Inspect Deployment

```bash
kubectl describe deployment frontend
```

Verify

* Image
* Environment Variables
* Resource Limits
* Volumes
* Secrets
* ConfigMaps

---

## Step 6 – Check Environment Variables

```bash
kubectl exec -it frontend -- env
```

Verify expected values exist.

---

## Step 7 – Check ConfigMaps

```bash
kubectl get configmap
```

```bash
kubectl describe configmap app-config
```

Example

```text
DATABASE_HOST=mysql
```

---

## Step 8 – Check Secrets

```bash
kubectl get secrets
```

```bash
kubectl describe secret db-secret
```

Verify required secret exists.

---

## Step 9 – Rollout History

```bash
kubectl rollout history deployment frontend
```

Useful after recent deployments.

---

# 7. Understanding Exit Codes

## Exit Code 0

Application completed successfully.

Usually not expected for web servers.

---

## Exit Code 1

Generic application error.

Most common.

Investigate logs.

---

## Exit Code 127

Executable not found.

Example

```Dockerfile
ENTRYPOINT ["server"]
```

but

```text
server
```

does not exist.

---

## Exit Code 126

Permission denied.

Usually executable lacks permissions.

---

## Exit Code 137

Container killed due to OOM.

Investigate memory usage.

---

## Exit Code 139

Segmentation fault.

Often native code or C/C++ libraries.

---

## Exit Code 143

Container terminated gracefully.

Usually during rolling updates.

---

# 8. Root Cause Scenarios

---

## Scenario 1 – Missing ConfigMap

Logs

```text
Configuration file not found
```

Describe

```text
MountVolume.SetUp failed
```

Root Cause

ConfigMap missing.

Fix

```bash
kubectl apply -f configmap.yaml
```

---

## Scenario 2 – Database Down

Logs

```text
Connection refused
```

Verify

```bash
kubectl get svc
```

Check

```bash
kubectl exec -it frontend -- nslookup mysql
```

Root Cause

Database unavailable.

---

## Scenario 3 – Wrong Environment Variables

Logs

```text
DATABASE_HOST is empty
```

Verify

```bash
kubectl describe deployment frontend
```

Root Cause

Incorrect deployment configuration.

---

## Scenario 4 – Liveness Probe Failure

Describe

```text
Liveness probe failed

HTTP probe failed

500 Internal Server Error
```

Root Cause

Application healthy after 60 seconds but probe starts after 5 seconds.

Fix

Increase

```yaml
initialDelaySeconds
```

---

## Scenario 5 – Wrong Image Command

Logs

```text
exec: app: not found
```

Exit Code

```text
127
```

Root Cause

Incorrect ENTRYPOINT.

---

## Scenario 6 – OOMKilled

Describe

```text
Last State:

Reason: OOMKilled

Exit Code: 137
```

Root Cause

Application exceeded memory limit.

Increase memory or optimize application.

---

# 9. Fixes

Common fixes include:

* Correct ConfigMap
* Correct Secret
* Update Deployment
* Increase Memory
* Fix ENTRYPOINT
* Fix Docker Image
* Correct Probe Configuration
* Rollback Deployment

Rollback

```bash
kubectl rollout undo deployment frontend
```

Restart Deployment

```bash
kubectl rollout restart deployment frontend
```

---

# 10. Verification

After applying fixes:

## Check Pods

```bash
kubectl get pods
```

Expected

```text
NAME              READY   STATUS
frontend          1/1     Running
```

---

## Check Restart Count

```bash
kubectl describe pod frontend
```

Restart count should stop increasing.

---

## Verify Logs

```bash
kubectl logs frontend
```

Should not contain startup errors.

---

## Verify Readiness

```bash
kubectl get pods
```

```text
READY

1/1
```

---

## Verify Rollout

```bash
kubectl rollout status deployment frontend
```

Expected

```text
deployment "frontend" successfully rolled out
```

---

# 11. Prevention

* Use readiness and startup probes.
* Validate ConfigMaps before deployment.
* Validate Secrets.
* Implement proper logging.
* Monitor restart count.
* Set resource requests and limits.
* Use CI/CD validation.
* Perform health checks before production rollout.
* Implement canary deployments.

---

# 12. AI-SRE Investigation Flow

Your AI-SRE agent investigates CrashLoopBackOff as follows:

```text
Incident Detected
        │
        ▼
Evidence Collection
        │
        ├── Pods
        ├── Deployment
        ├── Events
        ├── Logs
        ├── ConfigMaps
        ├── Secrets
        └── Metrics
                │
                ▼
Incident Classifier
                │
                ▼
CrashLoopBackOff Playbook
                │
                ▼
Adaptive Evidence Collection
                │
                ▼
LLM Root Cause Analysis
                │
                ▼
Risk Assessment
                │
                ▼
Approval Workflow
                │
                ▼
Execute Remediation
                │
                ▼
Verification
                │
                ▼
Incident Summary
```

---

# 13. Interview Questions

### 1. What is CrashLoopBackOff?

A Kubernetes state where a container repeatedly crashes and Kubernetes delays restart attempts using exponential backoff.

---

### 2. Is CrashLoopBackOff the root cause?

No.

It is only a symptom.

---

### 3. Which command do you run first?

```bash
kubectl get pods
```

---

### 4. Which command usually provides the root cause?

```bash
kubectl logs --previous
```

---

### 5. Why use `kubectl describe pod`?

To inspect:

* Events
* Exit codes
* Restart count
* Probe failures
* Container state

---

### 6. What does Exit Code 137 mean?

The container was killed because it exceeded its memory limit (OOMKilled).

---

### 7. When should you use `kubectl logs --previous`?

When the container has already restarted and you need logs from the previous failed instance.

---

### 8. How do you rollback a bad deployment?

```bash
kubectl rollout undo deployment <deployment-name>
```

---

### 9. How do you verify the issue is resolved?

* Pod status is `Running`
* `READY` shows expected containers
* Restart count stops increasing
* Logs contain no startup errors
* Rollout completes successfully

---

### 10. What is the difference between `Error` and `CrashLoopBackOff`?

* **Error** indicates the container terminated unexpectedly.
* **CrashLoopBackOff** indicates Kubernetes has detected repeated crashes and is delaying further restart attempts.
