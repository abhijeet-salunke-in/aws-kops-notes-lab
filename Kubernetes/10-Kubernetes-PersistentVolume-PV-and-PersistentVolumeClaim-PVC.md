# Kubernetes Persistent Volumes (PV) and Persistent Volume Claims (PVC)

## Introduction

In Kubernetes, containers are **ephemeral** by nature. This means when a container or pod is deleted, all data stored inside the container filesystem is lost.

This behavior is acceptable for stateless applications but becomes a major problem for applications that need to store data permanently such as:

* Databases (MySQL, PostgreSQL, MongoDB)
* Logging Applications
* File Storage Systems
* CMS Applications
* ERP Applications
* E-commerce Applications
* User Upload Services

To solve this problem Kubernetes provides:

1. Persistent Volume (PV)
2. Persistent Volume Claim (PVC)

---

# Why Do We Need Persistent Storage?

Imagine you have a Notes Application.

User creates:

```text
Note 1
Note 2
Note 3
```

Data is stored inside container.

If pod gets deleted:

```bash
kubectl delete pod notes-app
```

Deployment creates a new pod.

But:

```text
All notes are lost.
```

This is because container storage is temporary.

---

# Evolution of Kubernetes Storage

## Stage 1 : Container Filesystem

Data stored inside container.

```text
Container
 └── notes.db
```

Problem:

```text
Container deleted = Data deleted
```

---

## Stage 2 : emptyDir

```yaml
volumes:
- name: data
  emptyDir: {}
```

### How It Works

```text
Pod Created
    ↓
emptyDir Created
    ↓
Containers Share Data
```

### Advantages

* Fast
* Shared between containers

### Disadvantages

When pod dies:

```text
Pod Deleted
    ↓
emptyDir Deleted
    ↓
Data Lost
```

---

## Stage 3 : hostPath

```yaml
hostPath:
  path: /database
```

### How It Works

```text
Node
 └── /database
      └── notes.db
```

Pod mounts:

```text
/database
      ↓
/app/data
```

### Advantages

Data survives pod deletion.

### Disadvantages

Data exists only on one node.

Example:

```text
Worker-1
 └── /database
```

If Worker-1 crashes:

```text
Data Lost
```

If pod moves to Worker-2:

```text
No Data
```

---

# Real Problem with hostPath

Consider:

```text
Worker-1
  └── Data

Worker-2
  └── Empty
```

Deployment recreates pod on Worker-2.

Result:

```text
Application starts
Data missing
```

This is not production-ready.

---

# Solution : Persistent Volume (PV)

Instead of storing data on node disk:

```text
Store data externally.
```

Examples:

* AWS EBS
* AWS EFS
* Azure Disk
* Azure Files
* NFS
* Ceph
* SAN Storage

---

# Real Life Example

Imagine a company office.

### Storage Room

```text
Persistent Volume (PV)
```

### Employees

```text
Pods
```

Employees do not directly access storage.

They request storage through management.

Management provides access.

---

# What is Persistent Volume (PV)?

PV is an actual storage resource available in cluster.

Example:

```text
AWS EBS Volume
5 GB
```

PV Definition:

```yaml
kind: PersistentVolume
```

Example:

```text
PV
 └── 5 GB EBS
```

---

# What is PVC?

PVC means:

```text
Persistent Volume Claim
```

PVC is a request for storage.

Example:

```text
I need 2GB Storage
```

PVC does not create storage.

PVC requests storage.

---

# Relationship Between PV and PVC

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
AWS EBS
```

---

# Real Life Analogy

Hotel Example

### Hotel Rooms

```text
Persistent Volumes
```

### Customer

```text
PVC
```

Customer says:

```text
I need one room.
```

Hotel assigns room.

Customer does not care which room.

Same happens in Kubernetes.

---

# PV vs PVC

| PV               | PVC                    |
| ---------------- | ---------------------- |
| Actual Storage   | Storage Request        |
| Created by Admin | Created by Developer   |
| Represents Disk  | Represents Requirement |
| AWS EBS          | Request for 5GB        |

---

# Storage Architecture

```text
Application
    ↓
Container
    ↓
Pod
    ↓
PVC
    ↓
PV
    ↓
AWS EBS
```

---

# Our AWS Architecture

```text
Control Plane
       ↓
Worker Node
       ↓
Pod
       ↓
PVC
       ↓
PV
       ↓
AWS EBS Volume
```

---

# Manual PV Creation

Create EBS Volume first.

AWS Console

```text
EC2
 ↓
Volumes
 ↓
Create Volume
```

Example:

```text
Size = 5GB
Type = gp3
AZ = ap-south-1a
```

---

# Create Persistent Volume

## pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv

spec:
  capacity:
    storage: 5Gi

  accessModes:
  - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-xxxxxxxx
```

Apply:

```bash
kubectl apply -f pv.yaml
```

Verify:

```bash
kubectl get pv
```

---

# Create PVC

## pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: ebs-pvc

spec:
  storageClassName: ""

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 2Gi
```

Apply:

```bash
kubectl apply -f pvc.yaml
```

Verify:

```bash
kubectl get pvc
```

Expected:

```text
STATUS = Bound
```

---

# Create Deployment

## deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: notes-app

spec:
  replicas: 1

  selector:
    matchLabels:
      app: notes

  template:
    metadata:
      labels:
        app: notes

    spec:
      containers:
      - name: notes-cont
        image: abhisalunke16/notes-app:v2

        ports:
        - containerPort: 5000

        volumeMounts:
        - name: notes-storage
          mountPath: /app/data

      volumes:
      - name: notes-storage
        persistentVolumeClaim:
          claimName: ebs-pvc
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

---

# Verification

Enter Pod

```bash
kubectl exec -it deploy/notes-app -- sh
```

Create file

```bash
echo "Hello EBS" > /app/data/test.txt
```

Delete Pod

```bash
kubectl delete pod -l app=notes
```

Deployment creates new pod.

Check file:

```bash
cat /app/data/test.txt
```

Output:

```text
Hello EBS
```

Persistence Verified.

---

# Problem We Faced During Lab

Pod stuck at:

```text
ContainerCreating
```

Command:

```bash
kubectl describe pod PODNAME
```

Error:

```text
FailedAttachVolume
```

---

# Root Cause

AWS denied attaching EBS.

Error:

```text
UnauthorizedOperation
```

Detailed Error:

```text
ec2:AttachVolume
```

Kubernetes tried:

```text
Attach EBS to Worker Node
```

AWS replied:

```text
Permission Denied
```

---

# Why Did This Happen?

Our Kops IAM Roles lacked permissions.

Role:

```text
masters.abhi.k8s.local
```

and

```text
nodes.abhi.k8s.local
```

did not have:

```text
AmazonEBSCSIDriverPolicy
```

---

# How We Fixed It

Attach policy:

```bash
aws iam attach-role-policy \
--role-name masters.abhi.k8s.local \
--policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

---

Attach policy:

```bash
aws iam attach-role-policy \
--role-name nodes.abhi.k8s.local \
--policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

Verify:

```bash
aws iam list-attached-role-policies \
--role-name masters.abhi.k8s.local
```

```bash
aws iam list-attached-role-policies \
--role-name nodes.abhi.k8s.local
```

---

Restart Deployment

```bash
kubectl rollout restart deployment notes-app
```

Result:

```text
Pod Running
```

---

# Troubleshooting Guide

## 1. Check PV

```bash
kubectl get pv
```

Expected:

```text
STATUS = Bound
```

---

## 2. Check PVC

```bash
kubectl get pvc
```

Expected:

```text
STATUS = Bound
```

---

## 3. Check Pod

```bash
kubectl get pods
```

If:

```text
ContainerCreating
```

Investigate further.

---

## 4. Describe Pod

```bash
kubectl describe pod PODNAME
```

Look for:

```text
Events
```

---

## 5. Check Volume Attachments

```bash
kubectl get volumeattachment
```

Expected:

```text
ATTACHED = true
```

---

## 6. Check CSI Controller Logs

```bash
kubectl logs -n kube-system deployment/ebs-csi-controller \
-c ebs-plugin
```

Useful for:

```text
Permission Issues
Attachment Issues
Volume Issues
```

---

## 7. Check CSI Node Logs

```bash
kubectl logs -n kube-system daemonset/ebs-csi-node \
-c ebs-plugin
```

Useful for:

```text
Mount Failures
Node Side Errors
```

---

## 8. Check AWS Volume

```bash
aws ec2 describe-volumes \
--region ap-south-1
```

Verify:

```text
Volume Exists
Volume Available
```

---

# Interview Questions

### What is PV?

A storage resource available in cluster.

---

### What is PVC?

A request for storage.

---

### Difference Between PV and PVC?

PV = Actual Storage

PVC = Request for Storage

---

### Why not use hostPath?

Because data exists on only one node.

---

### Why use EBS?

External persistent storage.

---

### What happens when Pod is deleted?

Pod deleted.

PVC survives.

PV survives.

EBS survives.

New pod mounts same data.

---

### What happens if Worker Node crashes?

With hostPath:

```text
Data Lost
```

With EBS:

```text
Data Safe
```

because EBS is external to node.

---

# Summary

```text
emptyDir
  ↓
Pod Deleted
  ↓
Data Lost

hostPath
  ↓
Pod Deleted
  ↓
Data Safe

hostPath
  ↓
Node Crash
  ↓
Data Lost

PV + PVC + EBS
  ↓
Pod Deleted
  ↓
Data Safe

PV + PVC + EBS
  ↓
Node Crash
  ↓
Data Safe
```
