# Kubernetes EmptyDir Volumes and Data Persistence

## Overview

Containers are designed to be lightweight and temporary.

By default, data written inside a container is not guaranteed to survive container recreation.

This creates a major challenge for applications that need to store data.

Examples:

```text
Application Logs
Uploaded Files
Temporary Cache
Generated Reports
Shared Files Between Containers
```

To solve this problem, Kubernetes provides a feature called:

```text
Volumes
```

Volumes allow data to be stored and shared independently of the container filesystem.

---

# The Data Persistence Problem

Consider a Pod running an application.

```text
Pod
 |
 +---- Container
```

Inside the container:

```text
/var/log/app.log
```

If the container crashes:

```text
Container Deleted
       |
       v
New Container Created
```

The data inside the container filesystem may be lost.

This is known as:

```text
Ephemeral Storage Problem
```

---

# Why Do We Need Volumes?

Without Volumes:

```text
Application
     |
     v
Writes Data
     |
     v
Container Filesystem
     |
     v
Container Deleted
     |
     v
Data Lost
```

Problems:

```text
No Data Persistence
No File Sharing
No Central Storage
No Stateful Applications
```

---

# Kubernetes Solution

Kubernetes introduced:

```text
Volumes
```

A Volume is a storage mechanism that allows Pods and Containers to store and share data.

---

# What is a Kubernetes Volume?

A Volume is a directory accessible to one or more containers inside a Pod.

Architecture:

```text
Pod
 |
 +------------------------+
 |                        |
 |   Container-1          |
 |   Container-2          |
 |                        |
 +-----------+------------+
             |
             v
         Volume
```

Both containers can access the same data.

---

# Benefits of Volumes

### Data Sharing

```text
Container-1
     |
     v
 Volume
     ^
     |
Container-2
```

---

### Data Persistence During Container Restart

```text
Container Crash
      |
      v
Container Restart
      |
      v
Data Still Available
```

---

### Centralized Storage

Multiple containers can read and write to the same location.

---

### Temporary Application Storage

Useful for:

```text
Cache
Logs
Scratch Data
Processing Files
```

---

# Types of Kubernetes Volumes

Kubernetes supports many volume types.

Examples:

```text
emptyDir
hostPath
Persistent Volume (PV)
Persistent Volume Claim (PVC)
NFS
AWS EBS
AWS EFS
ConfigMap
Secret
```

In this topic we focus on:

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
Directory Deleted
```

The volume exists only for the lifetime of the Pod.

---

# Important Rule

```text
Pod Created
     |
     v
emptyDir Created
```

```text
Pod Deleted
     |
     v
emptyDir Deleted
```

---

# How EmptyDir Works

Step 1:

```text
Pod Starts
```

Step 2:

```text
emptyDir Volume Created
```

Step 3:

```text
Containers Mount Volume
```

Step 4:

```text
Containers Read/Write Data
```

Step 5:

```text
Pod Deleted
```

Step 6:

```text
Volume Deleted
```

---

# EmptyDir Architecture

```text
Pod
 |
 +-------------------------------+
 |                               |
 |   Container-1                 |
 |      |                        |
 |      |                        |
 |      v                        |
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

# Why Use EmptyDir?

Suppose an application contains:

```text
Container-1
```

and

```text
Container-2
```

Container-1 creates files.

Container-2 reads files.

Without a shared volume:

```text
Container-1 Files
```

cannot be accessed by

```text
Container-2
```

EmptyDir solves this problem.

---

# Real-World Use Cases

## Shared Files Between Containers

```text
Frontend Container
      |
      v
emptyDir
      ^
      |
Backend Container
```

---

## Temporary Cache

```text
Cache Files
Session Files
Temporary Processing Data
```

---

## Log Aggregation

Application writes logs.

Log collector reads logs.

Both use the same EmptyDir volume.

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
    +-------------------+
    |                   |
    v                   v
Container-1       Container-2
        \         /
         \       /
          emptyDir
```

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

## Volume Mount

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

Create a file:

```bash
cd /mycont1

echo "Kubernetes Volume Test" > test.txt

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

Check shared data:

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
Kubernetes Volume Test
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

The EmptyDir volume remains available because the Pod still exists.

Data remains accessible.

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

Check:

```bash
kubectl get pods
```

A new Pod appears.

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
Directory Empty
```

Reason:

```text
Old Pod Deleted
Old EmptyDir Deleted
New Pod Created
New EmptyDir Created
```

---

# Advantages of EmptyDir

### Simple Configuration

```text
Easy to Create
Easy to Use
```

---

### Shared Storage

Multiple containers can access the same files.

---

### Fast Performance

Stored on node storage.

---

### Useful for Temporary Data

```text
Cache
Logs
Temporary Files
```

---

# Limitations of EmptyDir

### Data Lost After Pod Deletion

```text
Pod Deleted
     |
     v
Data Deleted
```

---

### Not Suitable for Databases

Databases require persistent storage.

---

### Not Suitable for Long-Term Storage

Data does not survive Pod replacement.

---

### Node Dependent

Volume exists only while Pod exists.

---

# EmptyDir vs Container Filesystem

| Feature                    | Container Storage | emptyDir |
| -------------------------- | ----------------- | -------- |
| Shared Between Containers  | No                | Yes      |
| Survives Container Restart | No                | Yes      |
| Survives Pod Deletion      | No                | No       |
| Easy to Configure          | Yes               | Yes      |

---

# EmptyDir vs HostPath

| Feature                    | emptyDir | hostPath        |
| -------------------------- | -------- | --------------- |
| Storage Location           | Pod      | Node            |
| Shared Containers          | Yes      | Yes             |
| Survives Container Restart | Yes      | Yes             |
| Survives Pod Deletion      | No       | Yes (same node) |
| Production Friendly        | No       | Limited         |

---

# Useful Commands

View Deployments:

```bash
kubectl get deploy
```

---

View Pods:

```bash
kubectl get pods
```

---

Describe Deployment:

```bash
kubectl describe deploy deploy1
```

---

Describe Pod:

```bash
kubectl describe pod <pod-name>
```

---

Enter Container:

```bash
kubectl exec -it <pod-name> -c cont1 -- /bin/bash
```

---

Delete Deployment:

```bash
kubectl delete deploy deploy1
```

---

# Interview Questions

### What is a Kubernetes Volume?

A storage mechanism that allows containers inside a Pod to store and share data.

---

### What is EmptyDir?

A temporary volume created when a Pod starts and deleted when the Pod is removed.

---

### Can multiple containers use the same EmptyDir volume?

Yes.

---

### Does EmptyDir survive container restart?

Yes, as long as the Pod exists.

---

### Does EmptyDir survive Pod deletion?

No.

---

### What is the main use case of EmptyDir?

Sharing temporary data between containers in the same Pod.

---

### Is EmptyDir suitable for databases?

No, because data is lost when the Pod is deleted.

---

# Summary

```text
Kubernetes Volumes
        |
        +--> Data Sharing
        |
        +--> Temporary Storage
        |
        +--> Data Persistence During Container Restart
        |
        v
      emptyDir
        |
        +--> Created With Pod
        |
        +--> Shared Across Containers
        |
        +--> Survives Container Restart
        |
        +--> Deleted With Pod
```

## Workflow

```text
Create Deployment
        |
        v
Create emptyDir Volume
        |
        v
Mount Volume Into Containers
        |
        v
Create File In Container-1
        |
        v
Read Same File In Container-2
        |
        v
Verify Shared Storage
        |
        v
Delete Pod
        |
        v
Volume Deleted
        |
        v
Data Lost
```

---

### Next Topic

```text
HostPath Volume
       |
       v
Store Data On Kubernetes Node
       |
       v
Data Can Survive Pod Recreation
       |
       v
Foundation For Understanding Persistent Storage
```
