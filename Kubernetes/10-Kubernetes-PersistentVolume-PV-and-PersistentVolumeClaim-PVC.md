# Kubernetes PersistentVolume (PV) and PersistentVolumeClaim (PVC)

## Overview

In previous topics we learned:

### emptyDir

```text
Container Restart
     |
     v
Data Exists
```

But:

```text
Pod Deleted
     |
     v
Data Deleted
```

---

### hostPath

```text
Pod Deleted
     |
     v
Data Still Exists
```

because data is stored on the Node.

However:

```text
Node Failure
     |
     v
Data Unavailable
```

and

```text
Pod Scheduled To Another Node
     |
     v
Data Not Accessible
```

This makes hostPath unsuitable for most production workloads.

---

# The Real Problem

Modern applications need:

```text
Databases

Application Uploads

Reports

Logs

Backups

Customer Data

Configuration Data
```

Requirements:

```text
Data Must Survive Pod Deletion

Data Must Survive Container Restart

Storage Must Be Separate From Pods

Storage Must Be Reusable

Storage Must Work Across Application Lifecycles
```

---

# Kubernetes Solution

Kubernetes introduced:

```text
Persistent Volume (PV)

Persistent Volume Claim (PVC)
```

---

# Real World Analogy

Imagine a company office.

Storage Team:

```text
Creates Storage
```

Developers:

```text
Need Storage
```

Instead of giving developers direct AWS access:

```text
AWS Storage
     |
     v
Persistent Volume
     |
     v
Persistent Volume Claim
     |
     v
Application
```

Storage and applications become independent.

---

# Why Not Directly Mount EBS Into Pods?

Without Kubernetes abstraction:

```text
Application
     |
     v
AWS EBS
```

Developers would need knowledge of:

```text
AWS

Azure

GCP

Storage Configuration

Volume Attachment
```

Not practical.

---

Kubernetes introduces abstraction:

```text
Application
     |
     v
PVC
     |
     v
PV
     |
     v
Storage
```

Applications only request storage.

Kubernetes handles the rest.

---

# What is a Persistent Volume (PV)?

Persistent Volume is a storage resource inside Kubernetes.

Think:

```text
Storage Supply
```

Example:

```text
AWS EBS Volume

10 GB
```

represented as:

```text
PersistentVolume
```

inside Kubernetes.

---

# What is a Persistent Volume Claim (PVC)?

PVC is a request for storage.

Think:

```text
Storage Demand
```

Example:

```text
Application Needs:

6 GB Storage
```

PVC requests storage from Kubernetes.

---

# Hotel Example

Hotel:

```text
100 Rooms Available
```

Equivalent:

```text
Persistent Volume
```

---

Customer:

```text
Needs Room
```

Equivalent:

```text
Persistent Volume Claim
```

---

Result:

```text
Room Allocated
```

Equivalent:

```text
PVC Bound To PV
```

---

# PV and PVC Architecture

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

# Storage Flow

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

Containers Access Storage
```

---

# Understanding the Storage Lifecycle

Without PV/PVC:

```text
Pod Deleted
     |
     v
Storage Deleted
```

---

With PV/PVC:

```text
Pod Deleted
     |
     v
PVC Exists
     |
     v
PV Exists
     |
     v
EBS Exists
```

Data survives.

---

# AWS EBS Creation (Lab Setup)

Before creating PV:

Navigate:

```text
AWS Console

→ EC2

→ Elastic Block Store

→ Volumes

→ Create Volume
```

---

## Volume Configuration

Example:

```text
Volume Type: gp3

Size: 12 GB

Availability Zone:
ap-south-1a
```

---

# Important Requirement

The Kubernetes worker node and EBS volume must be in the same Availability Zone.

Example:

```text
Worker Node:
ap-south-1a
```

Then:

```text
EBS Volume:
ap-south-1a
```

Otherwise:

```text
Volume Cannot Attach
```

---

# Create EBS Volume

Example:

```text
12 GB gp3 Volume
```

After creation:

```text
Copy Volume ID
```

Example:

```text
vol-0f807626fa5b8dafc
```

This ID is used in PV YAML.

---

# What is Access Mode?

Access mode defines how storage can be mounted.

Common options:

```text
ReadWriteOnce (RWO)

ReadOnlyMany (ROX)

ReadWriteMany (RWX)
```

---

## ReadWriteOnce (RWO)

Meaning:

```text
One Node Can Read And Write
```

Most AWS EBS volumes support:

```text
ReadWriteOnce
```

---

# What is Reclaim Policy?

Defines what happens after PVC deletion.

Options:

```text
Retain

Delete

Recycle
```

---

## Retain

```text
Keep Data
```

Storage remains.

---

## Delete

```text
Delete Storage
```

Volume removed.

---

## Recycle

```text
Clean Data
```

Storage becomes reusable.

---

# Practical Lab

## Step 1: Create Persistent Volume

### File: pv.yml

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
    volumeID: vol-0f807626fa5b8dafc
    fsType: ext4
```

---

# Understanding PV YAML Line By Line

## API Version

```yaml
apiVersion: v1
```

Uses Kubernetes core API.

---

## Kind

```yaml
kind: PersistentVolume
```

Creates a Persistent Volume object.

---

## Metadata

```yaml
metadata:
  name: mypv
```

PV name:

```text
mypv
```

---

## Spec

```yaml
spec:
```

Defines PV configuration.

---

## Capacity

```yaml
capacity:
  storage: 10Gi
```

Available storage:

```text
10 GB
```

---

## Access Modes

```yaml
accessModes:
  - ReadWriteOnce
```

Only one node can mount volume in read/write mode.

---

## Reclaim Policy

```yaml
persistentVolumeReclaimPolicy: Recycle
```

After PVC deletion:

```text
Storage Cleaned
Ready For Reuse
```

---

## AWS EBS Section

```yaml
awsElasticBlockStore:
```

Tells Kubernetes:

```text
Use AWS EBS Volume
```

---

## Volume ID

```yaml
volumeID: vol-0f807626fa5b8dafc
```

Actual AWS EBS volume.

---

## Filesystem Type

```yaml
fsType: ext4
```

Linux filesystem used by volume.

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

# PV States

## Available

```text
Waiting For PVC
```

---

## Bound

```text
PVC Connected
```

---

## Released

```text
PVC Deleted
```

---

## Failed

```text
Storage Problem
```

---

# Step 2: Create Persistent Volume Claim

### File: pvc.yml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: mypvc

spec:

  storageClassName: ""

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 6Gi
```

---

# Understanding PVC YAML Line By Line

## API Version

```yaml
apiVersion: v1
```

Core Kubernetes API.

---

## Kind

```yaml
kind: PersistentVolumeClaim
```

Creates PVC object.

---

## Metadata

```yaml
metadata:
  name: mypvc
```

PVC name.

---

## Storage Class Name

```yaml
storageClassName: ""
```

Important.

Means:

```text
Do NOT Use Dynamic Provisioning

Use Existing PV
```

This is required for your lab setup.

---

## Access Mode

```yaml
accessModes:
  - ReadWriteOnce
```

Must match PV access mode.

---

## Requested Storage

```yaml
storage: 6Gi
```

Application requests:

```text
6 GB
```

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

# Verify Binding

```bash
kubectl get pv
```

Expected:

```text
mypv
Bound
```

---

# Binding Logic

PV:

```text
10 GB
```

PVC:

```text
6 GB
```

Result:

```text
10 >= 6
```

Binding successful.

---

# Step 3: Create Deployment Using PVC

### File: my-deploy-pvc.yml

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
          mountPath: /mycont2
```

---

# Understanding Deployment Storage Flow

```text
Container
     |
     v
Volume Mount
     |
     v
PVC
     |
     v
PV
     |
     v
AWS EBS
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

# Step 4: Test Shared Storage

Get Pod Name:

```bash
kubectl get pods
```

Example:

```text
deploy2-7d6db774b7-kpmng
```

---

Enter Container 1:

```bash
kubectl exec -it deploy2-7d6db774b7-kpmng -c cont1 -- /bin/bash
```

Create file:

```bash
cd /mycont1

echo "Hello PV PVC" > test.txt

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

Enter Container 2:

```bash
kubectl exec -it deploy2-7d6db774b7-kpmng -c cont2 -- /bin/bash
```

Check:

```bash
ls /mycont2
```

Expected:

```text
test.txt
```

View:

```bash
cat /mycont2/test.txt
```

Expected:

```text
Hello PV PVC
```

---

# Step 5: Verify Persistence

Delete Pod:

```bash
kubectl get pods
```

```bash
kubectl delete pod <pod-name>
```

Deployment creates new Pod.

---

Get New Pod:

```bash
kubectl get pods
```

---

Enter Container:

```bash
kubectl exec -it <new-pod-name> -c cont1 -- /bin/bash
```

Check:

```bash
ls /mycont1
```

Expected:

```text
test.txt
```

Data still exists.

---

# Why Data Survives

Because:

```text
Data Stored On AWS EBS
```

not inside:

```text
Container

Pod

Node
```

---

# Advantages of PV and PVC

### Persistent Storage

Data survives Pod lifecycle.

---

### Cloud Integration

Supports:

```text
AWS EBS

AWS EFS

Azure Disk

Google Persistent Disk

NFS
```

---

### Decoupled Storage

Applications do not know storage details.

---

### Production Ready

Used for:

```text
Databases

Jenkins

Prometheus

Grafana

Applications

Enterprise Systems
```

---

# HostPath vs PV/PVC

| Feature          | HostPath | PV/PVC |
| ---------------- | -------- | ------ |
| Data Persistence | Yes      | Yes    |
| Node Dependent   | Yes      | No     |
| Cloud Storage    | No       | Yes    |
| Production Ready | No       | Yes    |
| Scalable         | No       | Yes    |

---

# Interview Questions

### What is a Persistent Volume?

A cluster storage resource managed by Kubernetes.

---

### What is a Persistent Volume Claim?

A request for storage by applications.

---

### What happens when PVC finds a matching PV?

```text
Bound State
```

---

### Why is storageClassName set to empty?

```text
Use Existing Static PV
```

---

### Which AWS service was used in this lab?

```text
Amazon EBS
```

---

### What is the relationship between PV and PVC?

```text
PV Supplies Storage

PVC Requests Storage
```

---

# Summary

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

Result:

Persistent Storage
Independent Of Pod Lifecycle
Production Ready
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

PVC Bound To PV
        |
        v

Create Deployment
        |
        v

Mount Storage
        |
        v

Create Data
        |
        v

Delete Pod
        |
        v

New Pod Created
        |
        v

Data Still Exists
```

### Next Topic

```text
StorageClass
      |
      v
Dynamic Provisioning
      |
      v
Automatic PV Creation
```
