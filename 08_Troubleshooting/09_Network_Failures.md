# 09 - Network Failures

> **Objective:** Learn how to investigate, identify, remediate, and verify network connectivity issues within a Kubernetes cluster, including Pod-to-Pod, Pod-to-Service, and external connectivity problems.

---

# Table of Contents

1. What are Network Failures?
2. Kubernetes Networking Architecture
3. Packet Flow
4. Common Causes
5. Symptoms
6. Investigation Workflow
7. Root Cause Analysis Commands
8. Common Root Cause Scenarios
9. Linux & Network-Level Investigation
10. Prometheus Metrics
11. Fixes
12. Verification
13. Prevention
14. AI-SRE Investigation Flow
15. Interview Questions

---

# 1. What are Network Failures?

A network failure occurs when two components in a Kubernetes cluster cannot communicate, even though they may both be healthy.

Examples include:

* Pod cannot reach another Pod.
* Pod cannot access a Service.
* Service cannot reach backend Pods.
* External clients cannot reach the application.
* Pods cannot access the Internet.

Unlike DNS issues, **the hostname resolves successfully, but the connection itself fails**.

---

# 2. Kubernetes Networking Architecture

```text
Client
   │
   ▼
Ingress / LoadBalancer
   │
   ▼
Service (ClusterIP)
   │
   ▼
kube-proxy (iptables/IPVS)
   │
   ▼
Endpoints
   │
   ▼
Pod Network (CNI)
   │
   ▼
Application Pod
```

Major networking components:

* CoreDNS
* kube-proxy
* CNI Plugin (Calico, Flannel, Cilium, Weave, etc.)
* Services
* Endpoints
* NetworkPolicies

---

# 3. Packet Flow

Suppose a frontend Pod calls:

```text
http://backend:8080
```

Flow:

```text
Frontend Pod
      │
      ▼
CoreDNS
      │
      ▼
Service IP
      │
      ▼
kube-proxy
      │
      ▼
Endpoint
      │
      ▼
Backend Pod
```

A failure at any stage breaks communication.

---

# 4. Common Causes

## Service Problems

* Wrong selector
* Wrong targetPort
* Missing Service

---

## Endpoint Problems

* No matching Pods
* Pods NotReady

---

## CNI Problems

* Calico failure
* Flannel failure
* Cilium failure

---

## kube-proxy Problems

* iptables rules missing
* IPVS issues

---

## NetworkPolicy

* Ingress denied
* Egress denied

---

## Application

* Wrong listening port
* Application bound only to `127.0.0.1`

---

## Infrastructure

* Firewall
* Routing
* Cloud security groups

---

# 5. Symptoms

Application log

```text
connection refused
```

or

```text
i/o timeout
```

or

```text
no route to host
```

Pods remain

```text
Running
```

making diagnosis more difficult.

---

# 6. Investigation Workflow

```text
Connection Failure
        │
        ▼
Check Pod
        │
        ▼
Check Service
        │
        ▼
Check Endpoints
        │
        ▼
Check DNS
        │
        ▼
Check NetworkPolicy
        │
        ▼
Check kube-proxy
        │
        ▼
Check CNI
        │
        ▼
Check Node Networking
        │
        ▼
Fix
        │
        ▼
Verify
```

---

# 7. Root Cause Analysis Commands

## Step 1 – Verify Pods

```bash
kubectl get pods -o wide
```

Ensure both Pods are Running.

---

## Step 2 – Verify Service

```bash
kubectl get svc
```

Example

```text
NAME

backend

TYPE

ClusterIP

CLUSTER-IP

10.96.34.25
```

---

## Step 3 – Verify Endpoints

```bash
kubectl get endpoints backend
```

Expected

```text
backend

10.244.0.18:8080
```

If

```text
<none>
```

then the Service selector is incorrect or Pods are not Ready.

---

## Step 4 – Test Connectivity

Create a temporary Pod

```bash
kubectl run net-test \
--image=busybox \
-it --rm
```

Inside

```bash
wget http://backend:8080

nc -vz backend 8080
```

---

## Step 5 – Verify NetworkPolicy

```bash
kubectl get networkpolicy
```

Describe

```bash
kubectl describe networkpolicy
```

Ensure ingress/egress rules allow traffic.

---

## Step 6 – Check kube-proxy

```bash
kubectl get pods -n kube-system
```

Look for

```text
kube-proxy
```

Expected

```text
Running
```

---

## Step 7 – Check CNI

```bash
kubectl get pods -n kube-system
```

Look for

* calico-node
* cilium
* flannel

---

## Step 8 – Verify Listening Port

Inside the application container

```bash
kubectl exec -it backend -- netstat -tulpn
```

or

```bash
ss -lnt
```

Expected

```text
0.0.0.0:8080
```

Not

```text
127.0.0.1:8080
```

---

# 8. Root Cause Scenarios

---

## Scenario 1 – Empty Endpoints

```bash
kubectl get endpoints backend
```

Output

```text
<none>
```

Root Cause

Wrong Service selector.

---

## Scenario 2 – Wrong targetPort

Service

```yaml
targetPort: 80
```

Application

```text
8080
```

Connection refused.

---

## Scenario 3 – NetworkPolicy

Traffic blocked.

Pods healthy.

DNS healthy.

Connections timeout.

---

## Scenario 4 – Application Listening on Localhost

```text
127.0.0.1:8080
```

instead of

```text
0.0.0.0:8080
```

Pods cannot reach the application.

---

## Scenario 5 – kube-proxy Failure

Service exists.

Endpoints exist.

Pods healthy.

Service IP unreachable.

Root Cause

kube-proxy not programming iptables/IPVS rules.

---

## Scenario 6 – CNI Failure

Pods on different nodes cannot communicate.

Node-local communication still works.

---

## Scenario 7 – Firewall

Nodes cannot reach each other.

Cloud firewall blocks Pod CIDR.

---

# 9. Linux & Network-Level Investigation

View interfaces

```bash
ip addr
```

Routes

```bash
ip route
```

Listening ports

```bash
ss -lnt
```

iptables

```bash
iptables -L
```

Connectivity

```bash
ping <pod-ip>
```

Trace route

```bash
traceroute <pod-ip>
```

Check kube-proxy logs

```bash
kubectl logs -n kube-system daemonset/kube-proxy
```

---

# 10. Prometheus Metrics

Useful metrics:

```promql
up
```

```promql
container_network_receive_bytes_total
```

```promql
container_network_transmit_bytes_total
```

```promql
container_network_receive_errors_total
```

```promql
container_network_transmit_errors_total
```

If using Cilium:

```promql
cilium_drop_count_total
```

If using Calico:

```promql
felix_active_local_endpoints
```

---

# 11. Fixes

Possible remediations

* Correct Service selector
* Correct targetPort
* Fix application bind address
* Restart kube-proxy
* Restore CNI
* Update NetworkPolicy
* Open firewall ports
* Fix routing

Restart kube-proxy

```bash
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

---

# 12. Verification

Verify Service

```bash
kubectl get endpoints
```

Verify connectivity

```bash
wget http://backend:8080
```

Verify application

```bash
curl http://backend:8080/health
```

Verify NetworkPolicy

Application communication succeeds.

---

# 13. Prevention

* Use readiness probes.
* Validate Service selectors.
* Test NetworkPolicies before deployment.
* Monitor kube-proxy.
* Monitor CNI health.
* Standardize application ports.
* Monitor network latency.

---

# 14. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── Pods
        ├── Services
        ├── Endpoints
        ├── NetworkPolicies
        ├── kube-proxy
        ├── CNI
        ├── Application Logs
        ├── Connectivity Tests
        └── Metrics
                │
                ▼
Network Failure Playbook
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

### 1. What is the difference between a DNS issue and a network issue?

* **DNS issue:** The hostname cannot be resolved.
* **Network issue:** The hostname resolves successfully, but the connection fails.

---

### 2. Which command tells you whether a Service has healthy backend Pods?

```bash
kubectl get endpoints <service-name>
```

---

### 3. What happens if a Service has no Endpoints?

Traffic reaches the Service but cannot be forwarded to any Pods.

---

### 4. How do you verify that an application is listening on the expected interface?

```bash
ss -lnt
```

or

```bash
netstat -tulpn
```

Ensure it listens on `0.0.0.0` rather than only `127.0.0.1`.

---

### 5. Which Kubernetes component programs Service routing rules?

**kube-proxy**, using iptables or IPVS.

---

### 6. Which Kubernetes component provides Pod networking?

A **Container Network Interface (CNI)** plugin such as Calico, Cilium, Flannel, or Weave.

---

### 7. How do you verify a network issue is resolved?

* Services have populated Endpoints.
* Connectivity tests (`curl`, `wget`, `nc`) succeed.
* Applications communicate normally.
* No network-related errors appear in application logs.
