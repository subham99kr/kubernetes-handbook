# 03 - ImagePullBackOff

> **Objective:** Learn how to investigate, identify, remediate, and verify Kubernetes workloads that fail to pull container images.

---

# Table of Contents

1. What is ImagePullBackOff?
2. Image Pull Lifecycle
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

# 1. What is ImagePullBackOff?

**ImagePullBackOff** indicates that Kubernetes failed to download a container image and is waiting before retrying.

Unlike **CrashLoopBackOff**, the container **never starts**.

Typically the sequence is:

```text
Pod Created
      │
      ▼
Container Runtime Tries to Pull Image
      │
      ▼
Image Pull Fails
      │
      ▼
ErrImagePull
      │
      ▼
Retry with Exponential Backoff
      │
      ▼
ImagePullBackOff
```

---

# 2. Image Pull Lifecycle

When a Pod is created:

1. Scheduler assigns the Pod to a node.
2. Kubelet receives the Pod specification.
3. Container runtime (containerd/Docker/CRI-O) checks for the image locally.
4. If the image is absent, it contacts the configured container registry.
5. If the pull succeeds, the container starts.
6. If the pull fails repeatedly, Kubernetes reports **ImagePullBackOff**.

---

# 3. Common Causes

## Registry Issues

* Docker Hub unavailable
* Private registry offline
* Registry DNS failure
* Registry timeout

---

## Authentication Issues

* Missing imagePullSecret
* Expired registry credentials
* Incorrect username/password
* Unauthorized registry access

---

## Image Issues

* Wrong image name
* Typo in repository
* Incorrect image tag
* Image deleted from registry

---

## Network Issues

* Node cannot reach registry
* Firewall blocking HTTPS
* Proxy misconfiguration

---

## Kubernetes Configuration

* Wrong imagePullPolicy
* Incorrect namespace secret
* Wrong ServiceAccount

---

# 4. Symptoms

```bash
kubectl get pods
```

Example:

```text
NAME         READY   STATUS             RESTARTS   AGE
frontend     0/1     ImagePullBackOff   0          3m
```

Notice:

* Restart Count = 0
* Container never started

---

# 5. Investigation Workflow

```text
kubectl get pods
        │
        ▼
kubectl describe pod
        │
        ▼
Read Events
        │
        ▼
Identify Pull Failure
        │
        ▼
Verify Image Name
        │
        ▼
Verify Registry Access
        │
        ▼
Verify imagePullSecrets
        │
        ▼
Verify ServiceAccount
        │
        ▼
Apply Fix
        │
        ▼
Verify Successful Pull
```

---

# 6. Root Cause Analysis Commands

## Step 1 – Check Pod Status

```bash
kubectl get pods
```

Output

```text
NAME         READY   STATUS             RESTARTS
frontend     0/1     ImagePullBackOff   0
```

---

## Step 2 – Describe Pod

```bash
kubectl describe pod frontend
```

Example

```text
Events:

Failed to pull image "myrepo/frontend:v2"

rpc error:

manifest unknown
```

Interpretation

The image tag does not exist.

---

Another example

```text
Failed to pull image

unauthorized:

authentication required
```

Interpretation

Registry authentication failed.

---

Another example

```text
dial tcp

lookup registry.example.com

no such host
```

Interpretation

DNS resolution failure.

---

## Step 3 – Verify Image Name

```bash
kubectl get deployment frontend -o yaml
```

Verify

```yaml
image: myrepo/frontend:v2
```

Look carefully for:

* spelling
* repository
* organization
* image tag

---

## Step 4 – Verify imagePullSecrets

```bash
kubectl get secrets
```

```bash
kubectl describe pod frontend
```

Example

```yaml
imagePullSecrets:
- regcred
```

Verify that **regcred** exists.

---

## Step 5 – Verify Secret

```bash
kubectl describe secret regcred
```

Ensure the registry credentials are valid.

---

## Step 6 – Verify ServiceAccount

```bash
kubectl describe serviceaccount default
```

Confirm the ServiceAccount references the required imagePullSecret.

---

## Step 7 – Test Registry Reachability

Run a temporary pod:

```bash
kubectl run net-test \
--image=busybox \
-it --rm
```

Inside the pod:

```bash
nslookup registry.example.com

wget https://registry.example.com
```

Purpose

Verify:

* DNS
* Network
* HTTPS connectivity

---

# 7. Root Cause Scenarios

---

## Scenario 1 – Wrong Tag

Describe output

```text
manifest unknown
```

Root Cause

Tag does not exist.

Fix

Correct the deployment image.

---

## Scenario 2 – Private Registry Authentication

Describe output

```text
unauthorized
```

Root Cause

Missing credentials.

Fix

```bash
kubectl create secret docker-registry regcred ...
```

Update Deployment

```yaml
imagePullSecrets:
- name: regcred
```

---

## Scenario 3 – Registry Offline

Events

```text
connection refused
```

Root Cause

Registry unavailable.

Verify registry health.

---

## Scenario 4 – DNS Failure

Events

```text
no such host
```

Root Cause

Cluster DNS cannot resolve registry hostname.

Investigate CoreDNS.

---

## Scenario 5 – Wrong Repository Name

Deployment

```yaml
image: frontend:v1
```

Actual image

```text
company/frontend:v1
```

Root Cause

Repository typo.

---

## Scenario 6 – Deleted Image

Events

```text
manifest unknown
```

Image existed yesterday.

Registry cleanup removed it.

---

# 8. Fixes

Common fixes include:

* Correct image name
* Correct image tag
* Push missing image
* Recreate imagePullSecret
* Fix registry credentials
* Restore registry connectivity
* Configure ServiceAccount
* Rollback deployment

Rollback

```bash
kubectl rollout undo deployment frontend
```

---

# 9. Verification

Verify Pod

```bash
kubectl get pods
```

Expected

```text
NAME       READY   STATUS
frontend   1/1     Running
```

---

Verify Events

```bash
kubectl describe pod frontend
```

No further pull failures should appear.

---

Verify Image

```bash
kubectl get pod frontend \
-o=jsonpath="{.spec.containers[*].image}"
```

Confirm the expected image is running.

---

# 10. Prevention

* Use immutable image tags.
* Avoid `latest` in production.
* Validate images in CI/CD.
* Monitor registry availability.
* Rotate registry credentials.
* Replicate images to local registries for critical workloads.
* Test imagePullSecrets before deployment.

---

# 11. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── Pod Status
        ├── Events
        ├── Deployment
        ├── ServiceAccount
        ├── imagePullSecrets
        ├── Registry Configuration
        └── Network Checks
                │
                ▼
ImagePullBackOff Playbook
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

### 1. What is the difference between **ErrImagePull** and **ImagePullBackOff**?

* **ErrImagePull** is the initial image pull failure.
* **ImagePullBackOff** occurs after Kubernetes retries and begins exponential backoff.

---

### 2. Does the container ever start in ImagePullBackOff?

No. The image is never successfully pulled.

---

### 3. Which command gives the most useful information?

```bash
kubectl describe pod <pod-name>
```

The Events section usually contains the exact registry error.

---

### 4. What causes `manifest unknown`?

The requested image tag or repository does not exist.

---

### 5. What causes `unauthorized`?

Registry authentication failure, often due to missing or incorrect `imagePullSecrets`.

---

### 6. How do you verify an imagePullSecret?

```bash
kubectl describe secret <secret-name>
```

Also confirm that the Pod or ServiceAccount references it.

---

### 7. Why should production deployments avoid the `latest` tag?

Because it is mutable and can lead to non-reproducible deployments.

---

### 8. How do you verify the issue is resolved?

* Pod reaches `Running`
* Image is pulled successfully
* No image pull errors appear in Events
* Application containers start normally
