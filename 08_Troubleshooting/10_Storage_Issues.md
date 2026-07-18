# 10 - Storage Issues

> **Objective:** Learn how to investigate, identify, remediate, and verify Kubernetes storage-related issues involving Persistent Volumes (PV), Persistent Volume Claims (PVC), StorageClasses, CSI drivers, and volume mounts.

---

# Table of Contents

1. What are Storage Issues?
2. Kubernetes Storage Architecture
3. Volume Provisioning Lifecycle
4. Common Causes
5. Symptoms
6. Investigation Workflow
7. Root Cause Analysis Commands
8. Common Root Cause Scenarios
9. Linux & Storage-Level Investigation
10. Prometheus Metrics
11. Fixes
12. Verification
13. Prevention
14. AI-SRE Investigation Flow
15. Interview Questions

---

# 1. What are Storage Issues?

Storage issues occur when Kubernetes cannot provision, attach, mount, or access persistent storage required by an application.

Typical symptoms include:

* Pod stuck in `Pending`
* Pod stuck in `ContainerCreating`
* Volume mount failures
* Read-only filesystem
* Data unavailable
* Database startup failures

Storage issues usually involve one or more of:

* PersistentVolume (PV)
* PersistentVolumeClaim (PVC)
* StorageClass
* CSI Driver
* Node storage subsystem

---

# 2. Kubernetes Storage Architecture

```text
Application
      │
      ▼
Pod
      │
      ▼
PersistentVolumeClaim (PVC)
      │
      ▼
PersistentVolume (PV)
      │
      ▼
StorageClass
      │
      ▼
CSI Driver
      │
      ▼
Storage Backend
(NFS / EBS / Azure Disk / GCE PD / Ceph / Local Disk)
```

---

# 3. Volume Provisioning Lifecycle

```text
PVC Created
      │
      ▼
Scheduler Requests Storage
      │
      ▼
StorageClass Selected
      │
      ▼
CSI Driver Creates Volume
      │
      ▼
PV Created
      │
      ▼
PVC Bound
      │
      ▼
Volume Attached
      │
      ▼
Mounted into Pod
```

Failure at any stage prevents the application from accessing storage.

---

# 4. Common Causes

## PVC Issues

* PVC Pending
* Wrong access mode
* Requested storage too large

---

## PV Issues

* No matching PV
* PV already bound
* Incorrect reclaim policy

---

## StorageClass

* StorageClass missing
* Wrong provisioner
* Incorrect parameters

---

## CSI Driver

* CSI Pods crashed
* CSI Driver unavailable
* Node plugin failure

---

## Volume Mount

* Permission denied
* Read-only filesystem
* Mount timeout

---

## Node

* Disk full
* Device unavailable
* Mount path missing

---

# 5. Symptoms

Example

```bash
kubectl get pods
```

Output

```text
NAME         READY   STATUS               RESTARTS
mysql        0/1     ContainerCreating    0
```

PVC

```bash
kubectl get pvc
```

Output

```text
NAME          STATUS

mysql-data    Pending
```

---

# 6. Investigation Workflow

```text
Pod Pending / ContainerCreating
              │
              ▼
Check PVC
              │
              ▼
Check PV
              │
              ▼
Check StorageClass
              │
              ▼
Check CSI Driver
              │
              ▼
Check Events
              │
              ▼
Check Node Mount
              │
              ▼
Fix
              │
              ▼
Verify
```

---

# 7. Root Cause Analysis Commands

## Step 1 – Check Pods

```bash
kubectl get pods
```

---

## Step 2 – Check PVC

```bash
kubectl get pvc
```

Example

```text
NAME          STATUS

mysql-data    Pending
```

A `Pending` PVC indicates that storage has not yet been allocated.

---

## Step 3 – Describe PVC

```bash
kubectl describe pvc mysql-data
```

Example

```text
Events

waiting for first consumer

or

no persistent volumes available
```

---

## Step 4 – Check PersistentVolumes

```bash
kubectl get pv
```

Expected

```text
NAME

STATUS

CAPACITY

ACCESS MODES

pv-01

Bound

20Gi

RWO
```

---

## Step 5 – Check StorageClasses

```bash
kubectl get storageclass
```

Example

```text
NAME

PROVISIONER

standard

ebs.csi.aws.com
```

Verify that the StorageClass referenced by the PVC exists.

---

## Step 6 – Check CSI Driver

```bash
kubectl get pods -n kube-system
```

Look for

* ebs-csi-controller
* ebs-csi-node
* csi-provisioner
* csi-attacher

All should be `Running`.

---

## Step 7 – Check Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Typical storage-related messages include:

* FailedAttachVolume
* FailedMount
* VolumeAttachment timeout
* Multi-Attach error

---

## Step 8 – Describe Pod

```bash
kubectl describe pod mysql
```

Example

```text
Warning

FailedMount

Unable to attach or mount volumes
```

---

# 8. Root Cause Scenarios

---

## Scenario 1 – PVC Pending

PVC

```text
Pending
```

Root Cause

No available PersistentVolume or dynamic provisioning failed.

---

## Scenario 2 – StorageClass Missing

PVC

```yaml
storageClassName: fast-storage
```

Cluster

```text
No StorageClass named fast-storage
```

---

## Scenario 3 – CSI Driver Failure

CSI Pods

```text
CrashLoopBackOff
```

Volumes cannot be provisioned or attached.

---

## Scenario 4 – Multi-Attach Error

Event

```text
Multi-Attach error
```

A ReadWriteOnce (RWO) volume is already attached to another node.

---

## Scenario 5 – Read-only Filesystem

Application

```text
Read-only file system
```

Volume mounted as read-only or underlying storage issue.

---

## Scenario 6 – Permission Denied

Logs

```text
Permission denied
```

Possible causes:

* Incorrect `fsGroup`
* Incorrect file ownership
* SELinux/AppArmor restrictions

---

## Scenario 7 – Node Disk Full

Node

```bash
df -h
```

Output

```text
Filesystem

100% Used
```

No space remains to mount or write data.

---

# 9. Linux & Storage-Level Investigation

Disk usage

```bash
df -h
```

Block devices

```bash
lsblk
```

Mounted filesystems

```bash
mount
```

Kernel logs

```bash
dmesg | grep -i mount
```

CSI logs

```bash
kubectl logs -n kube-system <csi-controller-pod>
```

---

# 10. Prometheus Metrics

Useful metrics include:

```promql
kube_persistentvolume_status_phase
```

```promql
kube_persistentvolumeclaim_status_phase
```

```promql
kubelet_volume_stats_available_bytes
```

```promql
kubelet_volume_stats_used_bytes
```

```promql
kubelet_volume_stats_capacity_bytes
```

```promql
node_filesystem_avail_bytes
```

```promql
node_filesystem_usage_bytes
```

---

# 11. Fixes

Possible remediations

* Create missing PersistentVolumes.
* Correct StorageClass names.
* Restart CSI controller.
* Expand storage if supported.
* Correct access modes.
* Fix file permissions.
* Free node disk space.
* Recreate failed volume attachments.

Example

```bash
kubectl rollout restart deployment ebs-csi-controller \
-n kube-system
```

---

# 12. Verification

Verify PVC

```bash
kubectl get pvc
```

Expected

```text
NAME

STATUS

mysql-data

Bound
```

Verify PV

```bash
kubectl get pv
```

Status should be `Bound`.

Verify Pod

```bash
kubectl get pods
```

Expected

```text
mysql

Running
```

Verify application

Confirm the application can read and write data successfully.

---

# 13. Prevention

* Use dynamic provisioning where possible.
* Monitor PVC utilization.
* Monitor node disk usage.
* Keep CSI drivers updated.
* Test backup and restore procedures.
* Select appropriate access modes.
* Use StorageClasses consistently.

---

# 14. AI-SRE Investigation Flow

```text
Incident Detected
        │
        ▼
Collect Evidence
        │
        ├── Pods
        ├── PVC
        ├── PV
        ├── StorageClass
        ├── CSI Driver
        ├── Events
        ├── Node Disk
        └── Application Logs
                │
                ▼
Storage Playbook
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

### 1. What is the difference between a PV and a PVC?

* **PersistentVolume (PV):** The actual storage resource in the cluster.
* **PersistentVolumeClaim (PVC):** A request by a Pod for storage.

---

### 2. What happens if a PVC remains Pending?

The Pod cannot attach storage and may remain in `Pending` or `ContainerCreating`.

---

### 3. What is a StorageClass?

A StorageClass defines **how** storage should be dynamically provisioned, including the CSI provisioner and storage parameters.

---

### 4. What is a CSI Driver?

The **Container Storage Interface (CSI)** driver allows Kubernetes to provision, attach, detach, and mount storage from different storage providers.

---

### 5. What causes a Multi-Attach error?

A volume with **ReadWriteOnce (RWO)** access mode is already attached to another node and cannot be attached simultaneously.

---

### 6. Which commands are most useful when investigating storage issues?

```bash
kubectl get pvc
kubectl describe pvc <name>
kubectl get pv
kubectl get storageclass
kubectl describe pod <pod-name>
kubectl get events
```

---

### 7. How do you verify a storage issue is resolved?

* PVC status is `Bound`.
* PV status is `Bound`.
* Pod transitions to `Running`.
* Application can successfully read from and write to persistent storage.
