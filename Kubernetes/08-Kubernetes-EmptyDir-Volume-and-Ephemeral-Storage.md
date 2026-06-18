# Kubernetes EmptyDir Volume and Ephemeral Storage

## Overview

Applications running inside Kubernetes Pods often need storage.

Examples:

```text
Application Logs
Cache Files
Temporary Files
Shared Files Between Containers
Generated Reports
Processing Data
```

By default, containers use their own filesystem.

However, container storage is temporary.

If the container is recreated:

```text
Container Deleted
      |
      v
Data Lost
```

Kubernetes solves this problem using:

```text
Volumes
```

Volumes allow containers to store and share data.

---

# What is a Kubernetes Volume?

A Volume is a storage mechanism that allows data to exist independently of an individual container.

Architecture:

```text
Pod
 |
 +----------------------+
 |                      |
 |   Container-1        |
 |   Container-2        |
 |                      |
 +----------+-----------+
            |
            v
         Volume
```

Both containers can access the same storage.

---

# Why Do We Need Volumes?

Suppose a Pod contains two containers:

```text
Container-1
Container-2
```

Container-1 creates:

```text
report.txt
```

Container-2 needs to read:

```text
report.txt
```

Without a Volume:

```text
Container-1 Filesystem
         |
         X
         |
Container-2 Cannot Access
```

Data sharing becomes impossible.

---

# Storage Problems Before Volumes

## Problem #1 – No Shared Storage

```text
Container-1
     |
     v
Creates File
     |
     X
     |
Container-2 Cannot Access
```

---

## Problem #2 – Container Restart

Suppose application crashes.

```text
Container Deleted
      |
      v
Container Recreated
```

Data stored inside container filesystem may disappear.

---

## Problem #3 – Temporary Application Data

Applications often require:

```text
Logs
Cache
Processing Files
Temporary Output
```

A mechanism is needed to store these files.

---

# Kubernetes Solution

Kubernetes introduced:

```text
Volumes
```

Volumes provide:

```text
Data Sharing
Storage
Temporary Persistence
Container Independence
```

---

# Types of Kubernetes Volumes

Kubernetes supports many volume types:

```text
emptyDir
hostPath
Persistent Volume (PV)
Persistent Volume Claim (PVC)
AWS EBS
AWS EFS
NFS
ConfigMap
Secret
```

The first volume type usually learned is:

```text
emptyDir
```

---

# What is EmptyDir?

EmptyDir is the simplest volume type in Kubernetes.

When a Pod starts:

```text
Empty Directory Created
```

When the Pod is deleted:

```text
Empty Directory Deleted
```

The volume exists only while the Pod exists.

---

# Why Is It Called EmptyDir?

When Kubernetes creates the Pod:

```text
Volume Starts Empty
```

Example:

```text
/myvolume
```

Initially:

```text
No Files
No Folders
```

Containers create data inside it.

---

# EmptyDir Lifecycle

```text
Pod Created
     |
     v
emptyDir Created
     |
     v
Containers Use Volume
     |
     v
Pod Deleted
     |
     v
emptyDir Deleted
```

---

# What is Ephemeral Storage?

The word:

```text
Ephemeral
```

means:

```text
Temporary
Short-Lived
Non-Permanent
```

EmptyDir is considered:

```text
Ephemeral Storage
```

because its lifetime is tied to the Pod.

---

# Important Rule

```text
Pod Exists
     |
     v
emptyDir Exists
```

```text
Pod Deleted
     |
     v
emptyDir Deleted
```

---

# EmptyDir Architecture

```text
Pod
 |
 +--------------------------------+
 |                                |
 |      Container-1               |
 |            |                   |
 |            |                   |
 |            v                   |
 |                            Container-2
 |                                ^
 |                                |
 +---------------+----------------+
                 |
                 v
             emptyDir
```

Both containers access the same storage.

---

# How EmptyDir Works

Step 1:

```text
Pod Starts
```

---

Step 2:

```text
emptyDir Volume Created
```

---

Step 3:

```text
Containers Mount Volume
```

---

Step 4:

```text
Containers Read And Write Data
```

---

Step 5:

```text
Pod Deleted
```

---

Step 6:

```text
Volume Deleted
```

---

# Real-World Use Cases

## Sharing Files Between Containers

```text
Container-1
     |
     v
Creates File
     |
     v
emptyDir
     |
     v
Container-2 Reads File
```

---

## Temporary Cache

Applications generate cache files.

```text
Cache Data
```

stored inside EmptyDir.

---

## Log Collection

Application writes logs.

Log collector reads logs.

Both use same EmptyDir volume.

---

## Processing Jobs

```text
Input File
     |
     v
Processing
     |
     v
Output File
```

Temporary files stored in EmptyDir.

---

# Practical Lab

## Objective

Create:

```text
Deployment
      |
      v
Pod
      |
      +--------------------+
      |                    |
      v                    v
   cont1               cont2
      \                /
       \              /
        \            /
          emptyDir
```

Both containers will share data.

---

# Step 1: Create Deployment YAML

## File: k8s-vol.yml

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: deploy1

spec:
  replicas: 1

  selector:
    matchLabels:
      app: demo

  template:
    metadata:
      labels:
        app: demo

    spec:

      volumes:
      - name: myvolume
        emptyDir: {}

      containers:

      - name: cont1
        image: ubuntu

        command:
        - "/bin/bash"
        - "-c"
        - "while true; do echo Hello world; sleep 5; done"

        volumeMounts:
        - name: myvolume
          mountPath: "/mycont1"

      - name: cont2
        image: ubuntu

        command:
        - "/bin/bash"
        - "-c"
        - "while true; do echo Hello world; sleep 5; done"

        volumeMounts:
        - name: myvolume
          mountPath: "/mycont2"
```

---

# Understanding the YAML

## Volume Definition

```yaml
volumes:
- name: myvolume
  emptyDir: {}
```

Creates an EmptyDir volume.

---

## Mounting Volume

Container-1:

```yaml
volumeMounts:
- name: myvolume
  mountPath: /mycont1
```

Container-2:

```yaml
volumeMounts:
- name: myvolume
  mountPath: /mycont2
```

Both containers access the same storage.

---

# Step 2: Create Deployment

```bash
kubectl create -f k8s-vol.yml
```

Expected:

```text
deployment.apps/deploy1 created
```

---

# Step 3: Verify Deployment

```bash
kubectl get deploy
```

Expected:

```text
NAME      READY
deploy1   1/1
```

---

# Step 4: Verify Pod

```bash
kubectl get pods
```

Example:

```text
deploy1-7d6db774b7-kpmng
```

---

# Step 5: Enter Container-1

```bash
kubectl exec -it deploy1-7d6db774b7-kpmng -c cont1 -- /bin/bash
```

Create file:

```bash
cd /mycont1

echo "Hello Kubernetes" > test.txt

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

# Step 6: Enter Container-2

```bash
kubectl exec -it deploy1-7d6db774b7-kpmng -c cont2 -- /bin/bash
```

Check:

```bash
cd /mycont2

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
Hello Kubernetes
```

This proves:

```text
Both Containers Share Same Volume
```

---

# Container Restart Test

Suppose one container crashes.

```text
Container Restart
```

The Pod still exists.

Therefore:

```text
emptyDir Still Exists
```

Data remains available.

---

# Pod Deletion Test

Delete Pod:

```bash
kubectl delete pod <pod-name>
```

Example:

```bash
kubectl delete pod deploy1-7d6db774b7-kpmng
```

Deployment creates a new Pod.

---

# Verify Data Again

Enter new Pod:

```bash
kubectl exec -it <new-pod-name> -c cont1 -- /bin/bash
```

Check:

```bash
ls /mycont1
```

Expected:

```text
No Files
```

Why?

```text
Old Pod Deleted
      |
      v
Old emptyDir Deleted
      |
      v
New Pod Created
      |
      v
New emptyDir Created
```

---

# Why EmptyDir is NOT Persistent Storage

Persistent storage means:

```text
Data Survives Pod Deletion
```

EmptyDir does not satisfy this requirement.

Example:

```text
Pod Deleted
      |
      v
Volume Deleted
      |
      v
Data Lost
```

Therefore:

```text
emptyDir = Ephemeral Storage
```

not:

```text
Persistent Storage
```

---

# Advantages of EmptyDir

## Simple Configuration

```text
Easy To Create
Easy To Understand
```

---

## Shared Storage

Multiple containers can share data.

---

## Fast Performance

Uses node storage.

---

## Temporary Data Storage

Suitable for:

```text
Cache
Logs
Processing Data
Temporary Files
```

---

## Survives Container Restart

As long as Pod exists.

---

# Disadvantages of EmptyDir

## Data Lost When Pod Is Deleted

```text
Pod Deleted
      |
      v
Data Lost
```

---

## Not Suitable For Databases

Databases require persistent storage.

---

## Not Suitable For Critical Data

Important files should not be stored in EmptyDir.

---

## Pod Lifetime Dependency

Volume exists only while Pod exists.

---

# EmptyDir vs Container Filesystem

| Feature                     | Container Storage | emptyDir |
| --------------------------- | ----------------- | -------- |
| Shared Between Containers   | No                | Yes      |
| Survives Container Restart  | No                | Yes      |
| Survives Pod Deletion       | No                | No       |
| Suitable For Temporary Data | Limited           | Yes      |

---

# EmptyDir vs HostPath

| Feature                    | emptyDir  | hostPath        |
| -------------------------- | --------- | --------------- |
| Storage Location           | Pod       | Node            |
| Shared Between Containers  | Yes       | Yes             |
| Survives Container Restart | Yes       | Yes             |
| Survives Pod Deletion      | No        | Yes             |
| Type                       | Ephemeral | Node Persistent |
|                            |           |                 |

---

# Useful Commands

View Deployments:

```bash
kubectl get deploy
```

View Pods:

```bash
kubectl get pods
```

Describe Deployment:

```bash
kubectl describe deploy deploy1
```

Describe Pod:

```bash
kubectl describe pod <pod-name>
```

Enter Container:

```bash
kubectl exec -it <pod-name> -c cont1 -- /bin/bash
```

Delete Deployment:

```bash
kubectl delete deploy deploy1
```

---

# Interview Questions

### What is a Kubernetes Volume?

A storage mechanism used by containers to store and share data.

---

### What is EmptyDir?

A temporary volume created when a Pod starts and deleted when the Pod is removed.

---

### Why is EmptyDir called Ephemeral Storage?

Because the volume exists only during the Pod lifetime.

---

### Can multiple containers share EmptyDir?

Yes.

---

### Does EmptyDir survive container restart?

Yes.

---

### Does EmptyDir survive Pod deletion?

No.

---

### Is EmptyDir suitable for databases?

No.

---

### What is the main use case of EmptyDir?

Sharing temporary data between containers inside the same Pod.

---

# Summary

```text
Kubernetes Volumes
       |
       +--> emptyDir
               |
               +--> Shared Storage
               |
               +--> Temporary Storage
               |
               +--> Survives Container Restart
               |
               +--> Deleted With Pod
               |
               +--> Ephemeral Storage
```

## Workflow

```text
Create Deployment
       |
       v
Create emptyDir Volume
       |
       v
Mount Into Containers
       |
       v
Create File In cont1
       |
       v
Read File In cont2
       |
       v
Verify Shared Storage
       |
       v
Delete Pod
       |
       v
emptyDir Deleted
       |
       v
Data Lost
```

### Next Topic

```text
HostPath Volume
       |
       v
Node-Level Storage
       |
       v
Data Can Survive Pod Recreation
       |
       v
Foundation For Persistent Storage Concepts
```
