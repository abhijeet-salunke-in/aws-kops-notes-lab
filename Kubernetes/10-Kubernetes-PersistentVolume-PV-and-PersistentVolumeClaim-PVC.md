# Kubernetes PersistentVolume (PV) and PersistentVolumeClaim (PVC)

## Overview

In the previous storage topics we learned:

### EmptyDir

```text
Pod Deleted
     |
     v
Volume Deleted
     |
     v
Data Lost
```

EmptyDir provides:

```text
Shared Storage
Temporary Storage
Ephemeral Storage
```

but does not provide persistence.

---

### HostPath

```text
Node Filesystem
      |
      v
Pod Uses Data
```

HostPath provides:

```text
Node-Level Persistence
```

However it has a limitation:

```text
Node-1 Failure
       |
       v
Data Unavailable
```

HostPath is tied to a specific node.

---

# The Real Production Storage Problem

Modern applications require:

```text
Databases
Jenkins
SonarQube
GitLab
Prometheus
Grafana
ERP Applications
Banking Applications
E-Commerce Platforms
```

These applications cannot afford data loss.

Example:

```text
MySQL Pod Deleted
```

Should NOT mean:

```text
Database Deleted
```

---

# Kubernetes Solution

Kubernetes introduced:

```text
PersistentVolume (PV)

PersistentVolumeClaim (PVC)
```

These provide:

```text
Persistent Storage
Storage Abstraction
Cloud Storage Integration
Production-Ready Storage
```

---

# What is a Persistent Volume (PV)?

A Persistent Volume is a storage resource available inside a Kubernetes cluster.

Think of it as:

```text
Storage Supply
```

Examples:

```text
AWS EBS
AWS EFS
Azure Disk
Azure File Share
NFS
Ceph
SAN Storage
```

Kubernetes represents these storage systems as:

```text
Persistent Volume
```

---

# Real Life Example

Suppose a company purchases:

```text
500 GB Storage
```

from AWS.

Storage Team creates:

```text
AWS EBS Volume
```

Kubernetes converts it into:

```text
Persistent Volume
```

Applications can then consume storage without knowing AWS details.

---

# What is a Persistent Volume Claim (PVC)?

PVC is a request for storage.

Think of it as:

```text
Storage Request Form
```

Example:

Developer needs:

```text
10 GB Storage
```

PVC requests:

```text
10 GB
```

Kubernetes finds a suitable PV and connects them.

---

# Hotel Analogy

Imagine a hotel.

Hotel Rooms:

```text
Room 101
Room 102
Room 103
```

Equivalent to:

```text
Persistent Volumes
```

---

Customer wants a room.

Customer request:

```text
Need One Room
```

Equivalent to:

```text
Persistent Volume Claim
```

---

Result:

```text
Customer Gets Room
```

Equivalent:

```text
PVC Bound To PV
```

---

# Why Kubernetes Uses PV and PVC

Without PV/PVC:

```text
Developer Must Know:

AWS Storage
NFS Configuration
Storage Provisioning
Disk Management
```

Very complicated.

---

Kubernetes separates responsibilities.

Storage Administrator:

```text
Creates Storage
```

Application Developer:

```text
Consumes Storage
```

This separation makes infrastructure easier to manage.

---

# Storage Architecture

```text
AWS EBS Volume
        |
        v

Persistent Volume
        |
        v

Persistent Volume Claim
        |
        v

Deployment
        |
        v

Pod
        |
        v

Containers
```

---

# Complete Storage Flow

```text
Create AWS EBS
        |
        v

Create PV
        |
        v

Create PVC
        |
        v

PVC Binds To PV
        |
        v

Deployment Uses PVC
        |
        v

Pod Uses Storage
```

---

# Understanding PV Lifecycle

```text
Available
     |
     v

Bound
     |
     v

Released
     |
     v

Available / Failed
```

---

## Available

Storage exists but is unused.

---

## Bound

PVC successfully attached.

---

## Released

PVC removed.

---

## Failed

Storage issue occurred.

---

# What is Access Mode?

Access Mode defines how storage can be mounted.

---

## ReadWriteOnce (RWO)

```text
One Node
Read + Write
```

Most AWS EBS volumes use:

```text
ReadWriteOnce
```

---

## ReadOnlyMany (ROX)

```text
Many Nodes
Read Only
```

---

## ReadWriteMany (RWX)

```text
Many Nodes
Read + Write
```

Common with:

```text
EFS
NFS
```

---

# What is Reclaim Policy?

Determines what happens after PVC deletion.

---

## Retain

```text
PVC Deleted
      |
      v
Data Remains
```

---

## Delete

```text
PVC Deleted
      |
      v
Storage Deleted
```

---

## Recycle

```text
PVC Deleted
      |
      v
Data Cleaned
      |
      v
Storage Reusable
```

---

# AWS EBS Setup

Before creating PV, create an EBS volume.

Navigate:

```text
AWS Console
→ EC2
→ Elastic Block Store
→ Volumes
→ Create Volume
```

Example Configuration:

```text
Type: gp3
Size: 10 GB
AZ: ap-south-1a
```

---

# Important Requirement

Worker Node and EBS must be in the same Availability Zone.

Example:

```text
Worker Node:
ap-south-1a
```

EBS:

```text
ap-south-1a
```

Otherwise:

```text
Volume Attachment Fails
```

---

# Step 1: Create Persistent Volume

## File: pv.yml

```yaml
apiVersion: v1
kind: PersistentVolume

metadata:
  name: mypv

spec:
  capacity:
    storage: 10Gi

  accessModes:
  - ReadWriteOnce

  persistentVolumeReclaimPolicy: Recycle

  awsElasticBlockStore:
    volumeID: vol-0a3ebe6522290f9f0
    fsType: ext4
```

---

# Understanding PV YAML

## Capacity

```yaml
capacity:
  storage: 10Gi
```

Storage available:

```text
10 GB
```

---

## Access Mode

```yaml
accessModes:
- ReadWriteOnce
```

Only one node can mount the volume.

---

## Reclaim Policy

```yaml
persistentVolumeReclaimPolicy: Recycle
```

Controls storage cleanup.

---

## AWS EBS Mapping

```yaml
awsElasticBlockStore:
```

Connects Kubernetes PV to AWS EBS.

---

# Create PV

```bash
kubectl create -f pv.yml
```

Verify:

```bash
kubectl get pv
```

Expected:

```text
NAME   CAPACITY   ACCESS MODES   STATUS
mypv   10Gi       RWO            Available
```

---

# Step 2: Create Persistent Volume Claim

## File: pvc.yml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: mypvc

spec:
  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 6Gi
```

---

# Understanding PVC YAML

Developer requests:

```text
6 GB Storage
```

PV provides:

```text
10 GB Storage
```

Kubernetes binds them.

---

# Create PVC

```bash
kubectl create -f pvc.yml
```

Verify:

```bash
kubectl get pvc
```

Expected:

```text
NAME    STATUS   VOLUME
mypvc   Bound    mypv
```

---

Verify PV:

```bash
kubectl get pv
```

Expected:

```text
STATUS: Bound
```

---

# Understanding Binding

```text
PV Capacity  = 10 GB

PVC Request  = 6 GB
```

Since:

```text
10 GB >= 6 GB
```

Binding succeeds.

---

# Step 3: Create Deployment Using PVC

## File: my-deploy-pvc.yml

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: deploy2

spec:
  replicas: 1

  selector:
    matchLabels:
      app: demo2

  template:
    metadata:
      labels:
        app: demo2

    spec:

      volumes:
      - name: myvolume2
        persistentVolumeClaim:
          claimName: mypvc

      containers:

      - name: cont1
        image: ubuntu
        command: ["/bin/bash","-c","while true; do echo Hello world; sleep 5; done"]

        volumeMounts:
        - name: myvolume2
          mountPath: /mycont1

      - name: cont2
        image: ubuntu
        command: ["/bin/bash","-c","while true; do echo Hello world; sleep 5; done"]

        volumeMounts:
        - name: myvolume2
          mountPath: /myconti2
```

---

# Deployment Architecture

```text
AWS EBS
    |
    v

Persistent Volume
    |
    v

Persistent Volume Claim
    |
    v

Deployment
    |
    v

Pod
    |
+-----------+
|           |
v           v

cont1     cont2
```

---

# Create Deployment

```bash
kubectl create -f my-deploy-pvc.yml
```

Verify:

```bash
kubectl get deploy
```

```bash
kubectl get pods
```

---

# Practical Testing

## Step 1: Get Pod Name

```bash
kubectl get pods
```

Example:

```text
deploy2-65dc7d86b9-k5mgh
```

---

## Step 2: Create File From Container 1

```bash
kubectl exec -it deploy2-65dc7d86b9-k5mgh -c cont1 -- /bin/bash
```

Move to mount path:

```bash
cd /mycont1
```

Create file:

```bash
echo "PV PVC Test Successful" > test.txt
```

Verify:

```bash
ls
```

Expected:

```text
test.txt
```

Exit:

```bash
exit
```

---

## Step 3: Verify From Container 2

```bash
kubectl exec -it deploy2-65dc7d86b9-k5mgh -c cont2 -- /bin/bash
```

Check:

```bash
cd /myconti2

ls
```

Expected:

```text
test.txt
```

View content:

```bash
cat test.txt
```

Expected:

```text
PV PVC Test Successful
```

This proves:

```text
Both Containers Share Same Persistent Storage
```

---

# Persistence Verification Test

This is the most important test.

---

## Step 1: Delete Pod

Get Pod:

```bash
kubectl get pods
```

Delete:

```bash
kubectl delete pod <pod-name>
```

Example:

```bash
kubectl delete pod deploy2-65dc7d86b9-k5mgh
```

Deployment automatically creates a new Pod.

---

## Step 2: Verify New Pod

```bash
kubectl get pods
```

Notice:

```text
New Pod Name
```

---

## Step 3: Check Data Again

Enter new Pod:

```bash
kubectl exec -it <new-pod-name> -c cont1 -- /bin/bash
```

Verify:

```bash
cd /mycont1

ls
```

Expected:

```text
test.txt
```

View content:

```bash
cat test.txt
```

Expected:

```text
PV PVC Test Successful
```

---

# Why Data Survived

Because data is stored in:

```text
AWS EBS
```

not inside:

```text
Container
```

and not inside:

```text
Pod
```

Storage exists independently.

---

# Comparison of Storage Types

| Feature                    | EmptyDir | HostPath | PV + PVC |
| -------------------------- | -------- | -------- | -------- |
| Shared Storage             | Yes      | Yes      | Yes      |
| Survives Container Restart | Yes      | Yes      | Yes      |
| Survives Pod Deletion      | No       | Yes      | Yes      |
| Node Dependent             | No       | Yes      | No       |
| Production Ready           | No       | Limited  | Yes      |
| Cloud Integration          | No       | No       | Yes      |

---

# Advantages of PV and PVC

### True Persistence

```text
Pod Deleted
     |
     v
Data Remains
```

---

### Cloud Storage Support

```text
AWS EBS
AWS EFS
Azure Disk
NFS
```

---

### Storage Abstraction

Developers don't need infrastructure details.

---

### Production Ready

Suitable for:

```text
Databases
Jenkins
SonarQube
GitLab
ERP Systems
Enterprise Applications
```

---

### Independent Lifecycle

Storage lifecycle is separate from Pod lifecycle.

---

# Common Troubleshooting

## PVC Stuck in Pending

Check:

```bash
kubectl get pv
```

Possible reason:

```text
No Matching PV Found
```

---

## Volume Not Attaching

Check:

```text
EBS Availability Zone
Worker Node Availability Zone
```

Must match.

---

## Deployment Not Starting

Check:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Volume Mount Errors
PVC Errors
```

---

# Cleanup

Delete Deployment:

```bash
kubectl delete -f my-deploy-pvc.yml
```

Delete PVC:

```bash
kubectl delete -f pvc.yml
```

Delete PV:

```bash
kubectl delete -f pv.yml
```

Delete EBS Volume (AWS Console):

```text
EC2
→ Volumes
→ Select Volume
→ Delete Volume
```

---

# Interview Questions

### What is a Persistent Volume?

A Kubernetes storage resource that provides persistent storage to applications.

---

### What is a Persistent Volume Claim?

A request for storage made by an application.

---

### What is the relationship between PV and PVC?

```text
PV Supplies Storage

PVC Requests Storage
```

---

### What is the status of a successfully connected PV and PVC?

```text
Bound
```

---

### Why are PV and PVC used?

To provide storage independent of Pod lifecycle.

---

### Which AWS service was used in this lab?

```text
Amazon EBS
```

---

### What happens if a Pod is deleted?

Data remains because storage exists in the Persistent Volume.

---

# Summary

```text
Container Storage
       |
       v
Not Persistent

emptyDir
       |
       v
Ephemeral Storage

hostPath
       |
       v
Node-Level Persistence

Persistent Volume
       |
       v
Production Storage

Persistent Volume Claim
       |
       v
Storage Request

AWS EBS
       |
       v
Actual Storage
```

## Workflow

```text
Create EBS Volume
        |
        v
Create PV
        |
        v
Create PVC
        |
        v
PV Bound To PVC
        |
        v
Create Deployment
        |
        v
Create File
        |
        v
Delete Pod
        |
        v
New Pod Created
        |
        v
Data Still Exists
        |
        v
Persistence Verified
```
