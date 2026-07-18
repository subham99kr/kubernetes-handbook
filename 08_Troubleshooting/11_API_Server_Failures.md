# 11 - API Server Failures

> **Objective:** Learn how to investigate, identify, remediate, and verify Kubernetes API Server failures. The API Server is the front door of Kubernetes—if it becomes unavailable, almost every cluster operation is affected.

---

# Table of Contents

1. What is the Kubernetes API Server?
2. API Server Architecture
3. Request Lifecycle
4. Common Causes
5. Symptoms
6. Investigation Workflow
7. Root Cause Analysis Commands
8. Common Root Cause Scenarios
9. Linux & Control Plane Investigation
10. Prometheus Metrics
11. Fixes
12. Verification
13. Prevention
14. AI-SRE Investigation Flow
15. Interview Questions

---

# 1. What is the Kubernetes API Server?

The **kube-apiserver** is the central component of the Kubernetes control plane.

Almost every Kubernetes operation goes through the API Server:

* kubectl commands
* Controllers
* Scheduler
* Kubelet
* CoreDNS
* Operators
* Admission Controllers

If the API Server is unavailable:

* kubectl stops working
* New Pods cannot be scheduled
* Controllers stop reconciling resources
* Cluster automation pauses
* Existing running Pods usually continue running

---

# 2. API Server Architecture

```text
                   kubectl
                      │
                      ▼
              kube-apiserver
          ┌───────────┼────────────┐
          ▼           ▼            ▼
 Admission      Authentication   Authorization
 Controllers
          │
          ▼
        etcd
          ▲
          │
Scheduler │ Controllers │ Kubelets
```

---

# 3. Request Lifecycle

```text
kubectl apply
        │
        ▼
Authentication
        │
        ▼
Authorization (RBAC)
        │
        ▼
Admission Controllers
        │
        ▼
Validation
        │
        ▼
Persist Object to etcd
        │
        ▼
Return Response
```

Understanding this pipeline helps identify where requests fail.

---

# 4. Common Causes

## API Server Process Failure

* kube-apiserver crashed
* Static Pod terminated
* Misconfigured manifest

---

## etcd Unavailable

API Server cannot read or write cluster state.

---

## Certificate Problems

* Expired certificates
* Invalid CA
* Incorrect client certificates

---

## Resource Exhaustion

* CPU starvation
* Memory pressure
* File descriptor exhaustion

---

## Network Problems

* API Server port blocked
* Load balancer issue (HA clusters)
* DNS resolution failure

---

## Authentication / Authorization

* Invalid kubeconfig
* RBAC denial
* Expired tokens

---

# 5. Symptoms

Typical errors:

```text
Unable to connect to the server
```

```text
The connection to the server was refused
```

```text
context deadline exceeded
```

```text
EOF
```

```text
TLS handshake timeout
```

---

# 6. Investigation Workflow

```text
kubectl Fails
      │
      ▼
Check API Server Health
      │
      ▼
Check Static Pod
      │
      ▼
Check Logs
      │
      ▼
Check etcd
      │
      ▼
Check Certificates
      │
      ▼
Check Resources
      │
      ▼
Check Network
      │
      ▼
Fix
      │
      ▼
Verify
```

---

# 7. Root Cause Analysis Commands

## Step 1 – Verify Cluster Access

```bash
kubectl cluster-info
```

If this fails, the API Server may be unavailable or inaccessible.

---

## Step 2 – Check Component Health

```bash
kubectl get componentstatuses
```

> Note: `componentstatuses` is deprecated in recent Kubernetes versions. Prefer checking the control plane Pods and health endpoints directly.

---

## Step 3 – Check Static Pods

```bash
kubectl get pods -n kube-system
```

Look for:

```text
kube-apiserver
```

Expected:

```text
Running
```

---

## Step 4 – Check Logs

```bash
kubectl logs -n kube-system kube-apiserver-<node>
```

Look for:

* panic
* certificate errors
* etcd connection failures
* authentication failures

---

## Step 5 – Health Endpoint

```bash
curl -k https://<api-server>:6443/livez
```

Expected

```text
ok
```

Readiness endpoint:

```bash
curl -k https://<api-server>:6443/readyz?verbose
```

---

## Step 6 – Check Events

```bash
kubectl get events \
--sort-by=.metadata.creationTimestamp
```

---

# 8. Common Root Cause Scenarios

---

## Scenario 1 – API Server Crash

Logs

```text
panic:
```

Root Cause

Application crash or invalid configuration.

---

## Scenario 2 – etcd Unreachable

Logs

```text
connection refused

dial tcp
```

Root Cause

API Server cannot communicate with etcd.

---

## Scenario 3 – Certificate Expired

Logs

```text
certificate has expired
```

Root Cause

Control plane certificates need renewal.

---

## Scenario 4 – High CPU

```bash
kubectl top pod -n kube-system
```

API Server CPU:

```text
95%
```

Requests begin timing out.

---

## Scenario 5 – Authentication Failure

```text
Unauthorized
```

Possible causes:

* Invalid kubeconfig
* Expired client certificate
* Expired bearer token

---

## Scenario 6 – Port Blocked

```bash
nc -vz <api-server> 6443
```

Fails.

Firewall or network issue.

---

# 9. Linux & Control Plane Investigation

API Server manifest

```bash
ls /etc/kubernetes/manifests
```

---

View logs

```bash
journalctl -u kubelet
```

---

Check listening port

```bash
ss -lnt | grep 6443
```

---

Resource usage

```bash
top
```

---

Disk

```bash
df -h
```

---

Certificates

```bash
kubeadm certs check-expiration
```

---

Container runtime

```bash
crictl ps
```

---

# 10. Prometheus Metrics

Useful metrics:

```promql
apiserver_request_total
```

```promql
apiserver_request_duration_seconds_bucket
```

```promql
apiserver_current_inflight_requests
```

```promql
apiserver_storage_objects
```

```promql
apiserver_admission_controller_admission_duration_seconds
```

```promql
process_resident_memory_bytes
```

---

# 11. Fixes

Possible remediations

* Restart kube-apiserver
* Correct manifest configuration
* Restore etcd connectivity
* Renew certificates
* Free CPU and memory resources
* Open firewall port 6443
* Repair kubeconfig
* Restore control plane networking

---

# 12. Verification

Verify health endpoint

```bash
curl -k https://<api-server>:6443/livez
```

Expected

```text
ok
```

Verify cluster

```bash
kubectl get nodes
```

Expected

```text
All nodes Ready
```

Verify workloads

```bash
kubectl get pods -A
```

Controllers should resume normal reconciliation.

---

# 13. Prevention

* Monitor API Server latency.
* Monitor request rate.
* Alert on failed health checks.
* Renew certificates before expiry.
* Monitor control plane resources.
* Use highly available API Servers in production.
* Back up etcd regularly.

---

# 14. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── API Health
        ├── Static Pods
        ├── Logs
        ├── Events
        ├── etcd Status
        ├── Certificates
        ├── Metrics
        └── Network
                │
                ▼
API Server Playbook
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

# 15. Interview Questions

### 1. What is the role of the Kubernetes API Server?

It is the primary entry point for the Kubernetes control plane. All CRUD operations on Kubernetes resources pass through it.

---

### 2. Do running Pods stop if the API Server goes down?

Generally, **no**. Existing Pods continue running because the kubelet manages containers locally. However, no new scheduling or control plane reconciliation can occur until the API Server is available again.

---

### 3. Which port does the API Server typically listen on?

TCP **6443**.

---

### 4. How do you verify the API Server is healthy?

```bash
curl -k https://<api-server>:6443/livez
curl -k https://<api-server>:6443/readyz?verbose
```

---

### 5. What dependency is most critical for the API Server?

**etcd**, because it stores the cluster's persistent state.

---

### 6. How do you investigate an API Server that won't start?

* Check the static Pod manifest.
* Review kubelet logs.
* Review API Server logs.
* Verify certificates.
* Confirm etcd connectivity.
* Check available CPU, memory, and disk space.

---

### 7. Why is `componentstatuses` not recommended anymore?

It has been deprecated. Modern troubleshooting relies on health endpoints (`/livez`, `/readyz`) and inspecting control plane Pods and logs.

---

### 8. How do you verify the issue is resolved?

* `/livez` and `/readyz` return `ok`.
* `kubectl` commands succeed.
* Nodes report `Ready`.
* Controllers resume reconciliation.
* No new API Server errors appear in the logs.
