# 13 - Incident Response & Root Cause Analysis

> **Objective:** Learn how to respond to Kubernetes production incidents using a structured, repeatable SRE process—from detection to recovery, postmortem, and long-term prevention.

---

# Table of Contents

1. What is Incident Response?
2. Incident Lifecycle
3. Incident Severity Levels
4. Golden Signals
5. Incident Response Workflow
6. Evidence Collection
7. Investigation Strategy
8. Root Cause Analysis (RCA)
9. Mitigation vs Resolution
10. Communication During an Incident
11. Verification
12. Postmortem
13. AI-SRE Investigation Flow
14. Incident Checklist
15. Interview Questions

---

# 1. What is Incident Response?

Incident Response is the structured process of:

* Detecting an issue
* Assessing impact
* Investigating root cause
* Restoring service
* Preventing recurrence

A good SRE focuses on **restoring service quickly** while collecting enough evidence to understand what happened.

---

# 2. Incident Lifecycle

```text
Incident Detected
        │
        ▼
Acknowledge
        │
        ▼
Assess Severity
        │
        ▼
Collect Evidence
        │
        ▼
Investigate
        │
        ▼
Mitigate
        │
        ▼
Restore Service
        │
        ▼
Root Cause Analysis
        │
        ▼
Permanent Fix
        │
        ▼
Postmortem
```

---

# 3. Incident Severity Levels

| Severity | Description                                 | Example                                         |
| -------- | ------------------------------------------- | ----------------------------------------------- |
| Sev-1    | Complete outage affecting critical services | API Server unavailable, production cluster down |
| Sev-2    | Major functionality degraded                | Database unavailable, payment failures          |
| Sev-3    | Partial degradation                         | Some Pods failing, increased latency            |
| Sev-4    | Minor issue or cosmetic impact              | Dashboard unavailable, non-critical alert       |

Severity determines:

* Escalation
* Communication frequency
* Response priority
* Resource allocation

---

# 4. Golden Signals

Google's SRE model identifies four key signals:

## Latency

Requests becoming slower.

Example:

* API response time increases from 50 ms to 800 ms.

---

## Traffic

Incoming workload.

Examples:

* Requests per second
* Active users
* Queue depth

---

## Errors

Failed requests.

Examples:

* HTTP 500
* DNS failures
* Timeouts

---

## Saturation

Resource utilization approaching limits.

Examples:

* CPU
* Memory
* Disk
* Network
* Connection pools

---

# 5. Incident Response Workflow

```text
Alert Received
       │
       ▼
Confirm Incident
       │
       ▼
Assess Business Impact
       │
       ▼
Identify Affected Services
       │
       ▼
Collect Evidence
       │
       ▼
Choose Playbook
       │
       ▼
Investigate
       │
       ▼
Mitigate
       │
       ▼
Verify
       │
       ▼
Permanent Fix
```

---

# 6. Evidence Collection

Before changing anything, gather evidence.

## Cluster

```bash
kubectl get nodes

kubectl get pods -A

kubectl get events \
--sort-by=.metadata.creationTimestamp
```

---

## Describe Resources

```bash
kubectl describe pod <pod>

kubectl describe node <node>
```

---

## Logs

```bash
kubectl logs <pod>

kubectl logs <pod> --previous
```

---

## Metrics

```bash
kubectl top pods

kubectl top nodes
```

---

## Control Plane

```bash
kubectl get pods -n kube-system
```

---

## Storage

```bash
kubectl get pvc

kubectl get pv
```

---

## Networking

```bash
kubectl get svc

kubectl get endpoints

kubectl get networkpolicy
```

---

> **Best Practice:** Capture evidence before restarting Pods or deleting resources. Logs, events, and container state may be lost after recreation.

---

# 7. Investigation Strategy

A structured approach avoids jumping to conclusions.

## Step 1

What changed?

Examples:

* New deployment
* Configuration update
* Infrastructure maintenance
* Certificate renewal
* Traffic spike

---

## Step 2

What is failing?

* Application?
* Kubernetes?
* Infrastructure?
* Database?
* External dependency?

---

## Step 3

Determine the failure layer.

| Layer         | Examples                          |
| ------------- | --------------------------------- |
| Application   | Exceptions, crashes, memory leaks |
| Container     | OOMKilled, CrashLoopBackOff       |
| Pod           | Pending, scheduling               |
| Node          | NotReady                          |
| Network       | Service unreachable               |
| Storage       | FailedMount                       |
| Control Plane | API Server, etcd                  |

---

## Step 4

Follow the appropriate playbook.

Examples:

* CrashLoopBackOff
* Pending
* DNS
* Storage
* API Server
* etcd

---

# 8. Root Cause Analysis (RCA)

Avoid treating symptoms as causes.

Example:

```text
Users cannot access application
        │
        ▼
Service unavailable
        │
        ▼
Pods restarting
        │
        ▼
OOMKilled
        │
        ▼
Memory limit too low
        │
        ▼
Deployment configured incorrectly
```

**Root Cause:** Incorrect memory limit.

**Symptom:** Service unavailable.

A useful technique is the **Five Whys**:

1. Why did users receive errors? The Service had no healthy Pods.
2. Why were the Pods unhealthy? They kept restarting.
3. Why were they restarting? They were OOMKilled.
4. Why were they OOMKilled? The memory limit was too low.
5. Why was the limit too low? A deployment configuration change was not properly validated.

---

# 9. Mitigation vs Resolution

## Mitigation

Goal:

Restore service quickly.

Examples:

* Roll back deployment
* Restart failed Pods
* Scale replicas
* Increase memory limit
* Redirect traffic

---

## Resolution

Goal:

Remove the underlying cause.

Examples:

* Fix application bug
* Improve resource sizing
* Repair networking
* Renew certificates
* Replace failing hardware

Mitigation restores service. Resolution prevents recurrence.

---

# 10. Communication During an Incident

Communicate clearly and factually.

Include:

* Current impact
* Affected services
* Severity
* Actions taken
* Next update time

Example:

```text
Incident: Elevated API latency

Severity: Sev-2

Impact:
Customers may experience slower responses.

Current Status:
Investigation in progress.
High CPU observed on application Pods.

Mitigation:
Traffic shifted to healthy replicas.

Next Update:
15 minutes.
```

Avoid speculation until evidence supports a conclusion.

---

# 11. Verification

After mitigation or resolution:

Verify infrastructure:

```bash
kubectl get nodes
```

Verify workloads:

```bash
kubectl get pods -A
```

Verify application:

```bash
curl https://service/health
```

Verify metrics:

* CPU normalized
* Memory stable
* Error rate reduced
* Latency restored

Verify logs:

No new critical errors.

Continue monitoring to ensure the issue does not recur.

---

# 12. Postmortem

A good postmortem is **blameless** and focuses on improving systems and processes.

## Incident Summary

* Date
* Duration
* Impact
* Severity

---

## Timeline

| Time  | Event                         |
| ----- | ----------------------------- |
| 10:02 | Alert triggered               |
| 10:04 | On-call engineer acknowledged |
| 10:12 | Root cause identified         |
| 10:20 | Mitigation applied            |
| 10:35 | Service restored              |
| 11:10 | Permanent fix deployed        |

---

## Root Cause

Describe the underlying technical cause.

---

## Contributing Factors

Examples:

* Missing monitoring
* Incorrect configuration
* Insufficient testing
* Capacity limits

---

## Lessons Learned

Examples:

* Improve deployment validation.
* Add Prometheus alert.
* Increase test coverage.
* Document recovery steps.

---

## Action Items

| Task             | Owner         | Due Date   | Status      |
| ---------------- | ------------- | ---------- | ----------- |
| Add memory alert | Platform Team | 2026-08-01 | Open        |
| Create runbook   | SRE Team      | 2026-07-25 | In Progress |
| Add load test    | Dev Team      | 2026-08-10 | Open        |

---

# 13. AI-SRE Investigation Flow

```text
Alert
   │
   ▼
Collect Metrics
   │
   ▼
Collect Logs
   │
   ▼
Collect Events
   │
   ▼
Classify Incident
   │
   ├── CrashLoopBackOff
   ├── OOMKilled
   ├── Pending
   ├── DNS
   ├── Network
   ├── Storage
   ├── API Server
   └── etcd
          │
          ▼
Evidence Builder
          │
          ▼
LLM Root Cause Analysis
          │
          ▼
Confidence Score
          │
          ▼
Suggested Fixes
          │
          ▼
Human Approval
          │
          ▼
Automated Remediation
          │
          ▼
Verification
          │
          ▼
Incident Summary
```

---

# 14. Incident Checklist

## Detection

* [ ] Alert received
* [ ] Incident acknowledged
* [ ] Severity assigned

---

## Evidence

* [ ] Pods
* [ ] Nodes
* [ ] Events
* [ ] Logs
* [ ] Metrics
* [ ] Network
* [ ] Storage
* [ ] Control Plane

---

## Investigation

* [ ] Recent changes reviewed
* [ ] Root cause identified
* [ ] Impact understood

---

## Mitigation

* [ ] Service restored
* [ ] Users verified
* [ ] Monitoring confirms recovery

---

## Recovery

* [ ] Permanent fix implemented
* [ ] Documentation updated
* [ ] Postmortem completed
* [ ] Action items assigned

---

# 15. Interview Questions

### 1. What is the first thing you should do during a production incident?

Acknowledge the incident, assess its impact, and begin collecting evidence before making disruptive changes.

---

### 2. Why shouldn't you restart Pods immediately?

Restarting Pods can destroy valuable evidence such as logs, events, and container state that may be needed to determine the root cause.

---

### 3. What is the difference between mitigation and resolution?

* **Mitigation** restores service quickly.
* **Resolution** removes the underlying cause to prevent recurrence.

---

### 4. What are Google's Four Golden Signals?

* Latency
* Traffic
* Errors
* Saturation

---

### 5. What is a blameless postmortem?

A review focused on improving systems and processes rather than assigning blame to individuals.

---

### 6. Why is evidence collection important?

Evidence provides the factual basis for identifying the true root cause and avoids conclusions based on assumptions.

---

### 7. How would your AI-SRE agent assist during an incident?

It would:

* Collect logs, metrics, events, and Kubernetes resource information.
* Classify the incident.
* Correlate evidence.
* Suggest likely root causes with confidence scores.
* Recommend remediation steps.
* Present findings for human approval before automation.

---

### 8. What defines a successful incident response?

* Rapid service restoration.
* Accurate identification of the root cause.
* Effective communication.
* Verification of recovery.
* Action items implemented to reduce the likelihood of recurrence.
