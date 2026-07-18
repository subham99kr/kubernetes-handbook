# 08 - DNS Issues

> **Objective:** Learn how to investigate, identify, remediate, and verify DNS resolution failures in Kubernetes.

---

# Table of Contents

1. What are DNS Issues?
2. How Kubernetes DNS Works
3. DNS Resolution Flow
4. Common Causes
5. Symptoms
6. Investigation Workflow
7. Root Cause Analysis Commands
8. Common Root Cause Scenarios
9. Linux & Network-Level Investigation
10. Fixes
11. Verification
12. Prevention
13. AI-SRE Investigation Flow
14. Interview Questions

---

# 1. What are DNS Issues?

DNS (Domain Name System) is responsible for converting service names (e.g., `mysql.default.svc.cluster.local`) into IP addresses.

If DNS fails:

* Applications cannot locate services.
* Database connections fail.
* API calls fail.
* Microservices cannot communicate.

DNS issues often appear as application failures, even though the applications themselves are healthy.

---

# 2. How Kubernetes DNS Works

Every Pod receives a DNS configuration pointing to the cluster DNS service (usually CoreDNS).

Flow:

```text
Application
      │
      ▼
DNS Query
      │
      ▼
CoreDNS Service
      │
      ▼
CoreDNS Pods
      │
      ▼
Kubernetes API
      │
      ▼
Service IP Returned
      │
      ▼
Application Connects
```

---

# 3. DNS Resolution Flow

Example:

Application tries:

```text
mysql.default.svc.cluster.local
```

CoreDNS looks up:

```
Service
Namespace
Cluster Domain
```

Returns:

```text
10.96.34.25
```

Application connects.

---

# 4. Common Causes

## CoreDNS Problems

* CoreDNS Pods crashed
* CoreDNS Pending
* CoreDNS CrashLoopBackOff
* CoreDNS misconfiguration

---

## Service Problems

* Wrong Service name
* Wrong Namespace
* Missing Service

---

## Network Problems

* CNI failure
* NetworkPolicy blocking DNS
* Firewall issues

---

## Endpoint Problems

Service exists

↓

No Endpoints

---

## Application Problems

* Typo in hostname
* Wrong cluster domain

---

# 5. Symptoms

Application Logs

```text
dial tcp:

lookup mysql:

no such host
```

or

```text
Temporary failure in name resolution
```

---

Pod

```bash
kubectl get pods
```

may still show

```text
Running
```

because the application itself is healthy.

---

# 6. Investigation Workflow

```text
Application Error
        │
        ▼
Check Service
        │
        ▼
Check Endpoints
        │
        ▼
Check CoreDNS Pods
        │
        ▼
Test DNS
        │
        ▼
Check CoreDNS Logs
        │
        ▼
Check Network Policies
        │
        ▼
Check CNI
        │
        ▼
Fix
        │
        ▼
Verify
```

---

# 7. Root Cause Analysis Commands

## Step 1 – Verify Service

```bash
kubectl get svc
```

Example

```text
NAME      TYPE

mysql     ClusterIP
```

---

## Step 2 – Verify Endpoints

```bash
kubectl get endpoints mysql
```

Output

```text
NAME

ENDPOINTS

mysql

10.244.0.25:3306
```

If

```text
<none>
```

then the Service has no backing Pods.

---

## Step 3 – Check CoreDNS

```bash
kubectl get pods -n kube-system
```

Look for

```text
coredns
```

Expected

```text
Running
```

---

## Step 4 – Check CoreDNS Logs

```bash
kubectl logs -n kube-system deployment/coredns
```

Look for

* plugin errors
* timeouts
* upstream failures

---

## Step 5 – Launch Debug Pod

```bash
kubectl run dns-test \
--image=busybox \
-it --rm
```

Inside the Pod:

```bash
nslookup kubernetes.default
```

Expected

```text
Server:

10.96.0.10

Address:

10.96.0.10
```

---

Test another Service

```bash
nslookup mysql
```

---

## Step 6 – Check DNS Configuration

```bash
cat /etc/resolv.conf
```

Expected

```text
nameserver 10.96.0.10

search default.svc.cluster.local
```

---

## Step 7 – Check Network Policies

```bash
kubectl get networkpolicy
```

Verify UDP/TCP port 53 is allowed.

---

# 8. Root Cause Scenarios

---

## Scenario 1 – CoreDNS Crash

```text
coredns

CrashLoopBackOff
```

Root Cause

DNS server unavailable.

---

## Scenario 2 – Missing Service

Application

```text
mysql
```

Service

Doesn't exist.

---

## Scenario 3 – Empty Endpoints

```bash
kubectl get endpoints mysql
```

Output

```text
<none>
```

Root Cause

Service selector doesn't match Pods.

---

## Scenario 4 – Wrong Namespace

Application

```text
mysql
```

Actual Service

```text
mysql.database
```

---

## Scenario 5 – Network Policy

DNS traffic blocked.

UDP 53 denied.

---

## Scenario 6 – Wrong Cluster Domain

Application

```text
svc.local
```

Cluster

```text
svc.cluster.local
```

---

# 9. Linux & Network-Level Investigation

Test DNS

```bash
nslookup kubernetes.default
```

---

Alternative

```bash
dig kubernetes.default.svc.cluster.local
```

---

Connectivity

```bash
ping 10.96.0.10
```

---

Check DNS Port

```bash
nc -vz 10.96.0.10 53
```

---

Inspect CoreDNS ConfigMap

```bash
kubectl get configmap coredns \
-n kube-system -o yaml
```

---

# 10. Fixes

Possible remediations

* Restart CoreDNS
* Correct Service selector
* Fix Deployment labels
* Fix hostname
* Correct namespace
* Allow DNS in NetworkPolicy
* Restore CNI
* Repair CoreDNS configuration

Restart CoreDNS

```bash
kubectl rollout restart deployment coredns \
-n kube-system
```

---

# 11. Verification

Verify

```bash
nslookup kubernetes.default
```

Expected

Successful resolution.

Verify

```bash
kubectl get endpoints
```

Endpoints populated.

Verify

Application connects successfully.

---

# 12. Prevention

* Monitor CoreDNS.
* Alert on DNS failures.
* Use readiness probes for CoreDNS.
* Validate Service selectors.
* Test NetworkPolicies.
* Avoid hardcoded IP addresses.
* Monitor DNS latency.

---

# 13. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── Application Logs
        ├── Services
        ├── Endpoints
        ├── CoreDNS Pods
        ├── CoreDNS Logs
        ├── DNS Config
        ├── Network Policies
        └── CNI Status
                │
                ▼
DNS Playbook
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

### 1. Which component provides DNS inside Kubernetes?

**CoreDNS** (or kube-dns in older clusters).

---

### 2. Which command verifies DNS resolution from inside a Pod?

```bash
nslookup kubernetes.default
```

---

### 3. Why can a Service exist but still not work?

Because it may have **no Endpoints**, meaning no Pods match its selector.

---

### 4. How do you check whether CoreDNS is healthy?

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system deployment/coredns
```

---

### 5. Why should you inspect `/etc/resolv.conf` inside a Pod?

It shows the configured DNS server, search domains, and resolver options used by the application.

---

### 6. Which NetworkPolicy ports must typically be allowed for DNS?

* UDP 53
* TCP 53 (for larger DNS responses or fallback)

---

### 7. How do you verify a DNS issue is resolved?

* `nslookup` or `dig` succeeds.
* CoreDNS Pods are healthy.
* Services have populated Endpoints.
* Applications can resolve service names and communicate successfully.
