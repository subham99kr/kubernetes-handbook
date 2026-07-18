# 12 - etcd Failures

> **Objective:** Learn how to investigate, identify, remediate, and recover from **etcd failures**, including quorum loss, database corruption, disk issues, certificate problems, and disaster recovery.

---

# Table of Contents

1. What is etcd?
2. etcd Architecture
3. How Kubernetes Uses etcd
4. Common Causes
5. Symptoms
6. Investigation Workflow
7. Root Cause Analysis Commands
8. Common Root Cause Scenarios
9. Linux & etcd Investigation
10. Prometheus Metrics
11. Backup & Disaster Recovery
12. Fixes
13. Verification
14. Prevention
15. AI-SRE Investigation Flow
16. Interview Questions

---

# 1. What is etcd?

**etcd** is Kubernetes' distributed key-value database.

It stores the **entire cluster state**, including:

* Nodes
* Pods
* Deployments
* Services
* ConfigMaps
* Secrets
* PersistentVolumes
* RBAC objects
* Leases
* Events (short-lived)

Without etcd, the API Server has nowhere to persist or retrieve cluster state.

---

# 2. etcd Architecture

```text
                 kube-apiserver
                       │
                       ▼
                ┌────────────┐
                │    etcd    │
                └────────────┘
               /      │       \
              /       │        \
      Member-1   Member-2   Member-3
```

Production clusters typically use **3 or 5 etcd members**.

A majority (**quorum**) must be available for reads and writes.

---

# 3. How Kubernetes Uses etcd

```text
kubectl apply
      │
      ▼
API Server
      │
      ▼
Validate Request
      │
      ▼
Store Object in etcd
      │
      ▼
Watch Notification
      │
      ▼
Controllers & Scheduler
```

Every Kubernetes object is ultimately stored in etcd.

---

# 4. Common Causes

## Quorum Loss

Multiple etcd members unavailable.

---

## Disk Full

Database cannot accept writes.

---

## Disk Latency

Slow SSD/HDD causes request timeouts.

---

## Certificate Expiration

Mutual TLS authentication fails.

---

## Database Corruption

Improper shutdown or hardware failure.

---

## Network Partition

Members cannot communicate.

---

## Resource Exhaustion

CPU starvation

Memory pressure

I/O bottlenecks

---

# 5. Symptoms

Typical API Server errors:

```text
etcdserver: request timed out
```

```text
etcdserver: leader changed
```

```text
context deadline exceeded
```

```text
transport is closing
```

```text
failed to commit proposal
```

Users may notice:

* `kubectl` hangs
* New Pods not created
* Resource updates fail
* Controllers stop reconciling

---

# 6. Investigation Workflow

```text
API Server Errors
        │
        ▼
Check etcd Pods
        │
        ▼
Check Leader
        │
        ▼
Check Health
        │
        ▼
Check Logs
        │
        ▼
Check Disk
        │
        ▼
Check Certificates
        │
        ▼
Check Quorum
        │
        ▼
Restore if Needed
        │
        ▼
Verify
```

---

# 7. Root Cause Analysis Commands

## Step 1 – Check etcd Pods

```bash
kubectl get pods -n kube-system
```

Look for:

```text
etcd-control-plane
```

---

## Step 2 – Check Health

```bash
ETCDCTL_API=3 etcdctl \
endpoint health
```

Expected

```text
healthy
```

---

## Step 3 – Check Cluster Members

```bash
ETCDCTL_API=3 etcdctl \
member list
```

Verify:

* Leader exists
* Followers connected

---

## Step 4 – Check Endpoint Status

```bash
ETCDCTL_API=3 etcdctl \
endpoint status --write-out=table
```

Review:

* Leader
* Database size
* Raft term
* Raft index

---

## Step 5 – View Logs

```bash
kubectl logs -n kube-system etcd-<node>
```

Look for:

* corruption
* timeout
* election
* leader change

---

## Step 6 – Check Events

```bash
kubectl get events \
--sort-by=.metadata.creationTimestamp
```

---

# 8. Common Root Cause Scenarios

---

## Scenario 1 – Quorum Lost

Three-member cluster

Two members offline.

Result

Cluster unavailable because a majority is lost.

---

## Scenario 2 – Disk Full

```bash
df -h
```

Output

```text
Filesystem

100% Used
```

etcd cannot persist writes.

---

## Scenario 3 – Leader Election Storm

Logs

```text
leader changed
```

frequently.

Possible causes:

* Network instability
* High latency
* CPU starvation

---

## Scenario 4 – Certificate Expired

Logs

```text
x509:

certificate has expired
```

Members cannot establish secure connections.

---

## Scenario 5 – Database Corruption

Logs

```text
database corruption detected
```

Snapshot restoration may be required.

---

## Scenario 6 – Slow Storage

Metrics show:

* fsync latency increasing
* commit latency increasing

API Server begins timing out.

---

# 9. Linux & etcd Investigation

Disk

```bash
df -h
```

Disk performance

```bash
iostat -x
```

Processes

```bash
top
```

Listening ports

```bash
ss -lnt | grep 2379
```

View logs

```bash
journalctl -u kubelet
```

Check manifest

```bash
ls /etc/kubernetes/manifests
```

---

# 10. Prometheus Metrics

Useful metrics:

```promql
etcd_server_has_leader
```

```promql
etcd_server_leader_changes_seen_total
```

```promql
etcd_disk_backend_commit_duration_seconds_bucket
```

```promql
etcd_disk_wal_fsync_duration_seconds_bucket
```

```promql
etcd_mvcc_db_total_size_in_bytes
```

```promql
etcd_network_peer_round_trip_time_seconds
```

Alert if:

* Leader changes frequently.
* Commit latency rises sharply.
* Database size grows unexpectedly.
* No leader is present.

---

# 11. Backup & Disaster Recovery

## Create Snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot save backup.db
```

---

## Verify Snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot status backup.db
```

---

## Restore Snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot restore backup.db
```

Always verify the restore directory and update the etcd static Pod manifest if restoring to a new data directory.

---

# 12. Fixes

Possible remediations

* Restore failed etcd member.
* Recover quorum.
* Free disk space.
* Replace failed disk.
* Renew certificates.
* Restore from snapshot.
* Reduce storage latency.
* Restart failed members (only after identifying the cause).

> **Important:** Avoid restarting multiple etcd members simultaneously in a production cluster, as this can worsen outages or lead to quorum loss.

---

# 13. Verification

Verify

```bash
ETCDCTL_API=3 etcdctl endpoint health
```

Expected

```text
healthy
```

Verify leader

```bash
ETCDCTL_API=3 etcdctl endpoint status
```

Confirm:

* Leader present
* Followers synchronized

Verify Kubernetes

```bash
kubectl get nodes
kubectl get pods -A
```

Ensure API operations succeed without timeout errors.

---

# 14. Prevention

* Schedule automatic snapshots.
* Use SSD/NVMe storage.
* Monitor disk latency.
* Monitor database growth.
* Keep an odd number of members (3 or 5).
* Renew certificates before expiry.
* Never run a production cluster with a single etcd member.
* Test snapshot restoration regularly.

---

# 15. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── etcd Health
        ├── Leader Status
        ├── Member Status
        ├── Logs
        ├── Disk
        ├── Certificates
        ├── Metrics
        └── API Server Errors
                │
                ▼
etcd Playbook
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

# 16. Interview Questions

### 1. What is etcd?

A distributed, strongly consistent key-value store that holds Kubernetes cluster state.

---

### 2. What is quorum?

Quorum is the **majority of etcd members** required for the cluster to make progress. A 3-member cluster needs at least 2 healthy members.

---

### 3. Why is an odd number of etcd members recommended?

An odd number maximizes fault tolerance while minimizing unnecessary consensus overhead. Common deployments use **3** or **5** members.

---

### 4. What happens if etcd is unavailable?

The API Server cannot reliably read or write cluster state. Existing workloads typically continue running, but most control plane operations fail or become read-only depending on the failure.

---

### 5. Which command checks etcd health?

```bash
ETCDCTL_API=3 etcdctl endpoint health
```

---

### 6. What is the safest recovery method after catastrophic data loss?

Restore from a **verified etcd snapshot**, then confirm the restored cluster state and member health.

---

### 7. Which metrics are the most important?

* `etcd_server_has_leader`
* `etcd_server_leader_changes_seen_total`
* `etcd_disk_backend_commit_duration_seconds`
* `etcd_disk_wal_fsync_duration_seconds`
* `etcd_mvcc_db_total_size_in_bytes`

---

### 8. How do you verify the issue is resolved?

* All endpoints report `healthy`.
* A stable leader is elected.
* `kubectl` operations succeed.
* API Server logs no longer show etcd-related errors.
* No repeated leader elections or timeout errors are observed.
