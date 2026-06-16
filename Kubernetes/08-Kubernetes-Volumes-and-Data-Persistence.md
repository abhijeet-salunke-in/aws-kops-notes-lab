# Kubernetes Volumes and Data Persistence

## Overview

Containers are designed to be lightweight and temporary.

When a container stops, restarts, crashes, or gets recreated, the data stored inside that container can be lost.

This creates a major challenge for real-world applications because applications need to store:

* User files
* Application logs
* Configuration data
* Reports
* Uploaded documents
* Database files

Without persistent storage, applications become unreliable.

Kubernetes solves this problem using:

```text
Volumes
```

---

# The Problem Before Volumes

Suppose Kubernetes only had Pods and Containers.

```text
Pod
 |
 +-- Container
```

Inside the container:

```text
/data/file.txt
```

Everything works normally.

---

Now imagine:

```text
Container Crashes
```

or

```text
Pod Gets Deleted
```

or

```text
Pod Gets Recreated
```

Result:

```text
file.txt ❌ Lost
```

---

# Why Does This Happen?

Container storage is:

```text
Ephemeral Storage
```

Meaning:

```text
Temporary Storage
```

The lifecycle of container storage is tied to the container itself.

When the container disappears:

```text
Storage Disappears
```

---

# Real World Example

Imagine an E-Commerce application.

Users upload:

```text
Invoices
Product Images
Order Reports
Customer Files
```

Suppose these files are stored inside a container.

```text
Container
   |
   +-- uploads/
```

Now the Pod crashes.

Kubernetes creates a new Pod.

The uploaded files disappear.

This is unacceptable in production environments.

---

# Kubernetes Solution

Kubernetes introduced:

```text
Volumes
```

---

# What is a Kubernetes Volume?

A Volume is a storage mechanism attached to a Pod.

Unlike container storage:

```text
Volume survives container restart
```

and can be shared among multiple containers inside the same Pod.

---

# Simple Analogy

Think of:

```text
Containers = Employees
```

and

```text
Volume = Shared Office Folder
```

Every employee can access the same folder.

---

# Architecture Without Volume

```text
Pod
│
├── Container-1
│     └── Own Files
│
└── Container-2
      └── Own Files
```

Each container has separate storage.

Files cannot be shared.

---

# Architecture With Volume

```text
Pod
│
├── Container-1
│       │
│       ▼
│
├──── Shared Volume ────┐
│                       │
└── Container-2         │
        │               │
        ▼               ▼

     Shared Data
```

Both containers can access the same storage.

---

# Benefits of Volumes

## Data Sharing

Containers can access the same files.

---

## Better Data Persistence

Data survives container restart.

---

## Log Sharing

Application and monitoring containers can access common logs.

---

## Configuration Sharing

Multiple containers can use common configuration files.

---

## Inter-Container Communication

Containers can exchange data through shared files.

---

# Types of Kubernetes Volumes

Common volume types:

```text
emptyDir

hostPath

PersistentVolume (PV)

PersistentVolumeClaim (PVC)

NFS

AWS EBS

AWS EFS

Azure Disk

Ceph
```

---

# Today's Lab Topic

Today's practical focused on:

```text
emptyDir
```

---

# What is emptyDir?

emptyDir creates an empty storage area when a Pod starts.

Lifecycle:

```text
Pod Created
      |
      v
emptyDir Created
```

---

All containers inside the Pod can use it.

---

# Important Behavior

Container Restart:

```text
Data Remains
```

---

Pod Deletion:

```text
Data Lost
```

---

# Lifecycle of emptyDir

```text
Pod Starts
     |
     v

emptyDir Created
     |
     v

Containers Use Data
     |
     v

Container Restart
     |
     v

Data Still Exists
     |
     v

Pod Deleted
     |
     v

Volume Deleted
     |
     v

Data Lost
```

---

# Practical Demonstration

## Goal

Understand:

```text
Without Volume
```

vs

```text
With Volume
```

---

# Step 1: Create Deployment Without Volume

## File: deploy1.yml

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
      containers:

      - name: cont1
        image: ubuntu
        command:
        - /bin/bash
        - -c
        - while true; do echo Hello world; sleep 5; done

      - name: cont2
        image: ubuntu
        command:
        - /bin/bash
        - -c
        - while true; do echo Hello world; sleep 5; done
```

---

# Understanding This YAML

## Deployment

```yaml
kind: Deployment
```

Creates and manages Pods.

---

## Replicas

```yaml
replicas: 1
```

Maintains one Pod.

---

## Containers

```yaml
cont1
cont2
```

Two containers run inside the same Pod.

---

## Command

```yaml
command:
- /bin/bash
- -c
- while true; do echo Hello world; sleep 5; done
```

Keeps Ubuntu containers running continuously.

Without this command:

```text
Container Exits Immediately
```

---

# Step 2: Create Deployment

```bash
kubectl create -f deploy1.yml
```

---

# Step 3: Verify Deployment

```bash
kubectl get deploy
```

```bash
kubectl get pods
```

Expected:

```text
deploy1-xxxxxxxxxx-xxxxx
```

Running.

---

# Step 4: Enter Container 1

```bash
kubectl exec -it <pod-name> -c cont1 -- /bin/bash
```

Example:

```bash
kubectl exec -it deploy1-7d6db774b7-kpmng -c cont1 -- /bin/bash
```

---

# Step 5: Create Data

Inside Container 1:

```bash
mkdir mydir
```

```bash
echo "hello" > mydir/test.txt
```

Verify:

```bash
ls mydir
```

Output:

```text
test.txt
```

---

# Step 6: Exit Container

```bash
exit
```

---

# Step 7: Enter Container 2

```bash
kubectl exec -it <pod-name> -c cont2 -- /bin/bash
```

---

Check:

```bash
ls
```

or

```bash
ls mydir
```

Result:

```text
No Such Directory
```

---

# What Did We Learn?

Without Volumes:

```text
Container-1 Storage

≠

Container-2 Storage
```

Containers have isolated filesystems.

---

# Problem Statement

```text
Data Created By Container-1

Cannot Be Seen By Container-2
```

---

# Kubernetes Solution

Add:

```text
Shared Volume
```

---

# Step 8: Modify Deployment With Volume

Update the same file:

```text
deploy1.yml
```

---

# Deployment With Volume

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
        - /bin/bash
        - -c
        - while true; do echo Hello world; sleep 5; done

        volumeMounts:
        - name: myvolume
          mountPath: /mycont1

      - name: cont2
        image: ubuntu
        command:
        - /bin/bash
        - -c
        - while true; do echo Hello world; sleep 5; done

        volumeMounts:
        - name: myvolume
          mountPath: /mycont2
```

---

# Understanding The Volume Section

## Create Shared Volume

```yaml
volumes:
- name: myvolume
  emptyDir: {}
```

Creates shared storage.

---

## Mount Volume In Container 1

```yaml
volumeMounts:
- name: myvolume
  mountPath: /mycont1
```

Volume appears as:

```text
/mycont1
```

---

## Mount Volume In Container 2

```yaml
volumeMounts:
- name: myvolume
  mountPath: /mycont2
```

Volume appears as:

```text
/mycont2
```

---

# Architecture

```text
                Pod

 ┌─────────────────────────────┐

      Container-1

          /mycont1
              │
              ▼

        Shared Volume

              ▲
              │

          /mycont2

      Container-2

 └─────────────────────────────┘
```

---

# Step 9: Apply Updated YAML

```bash
kubectl apply -f deploy1.yml
```

Deployment recreates Pods.

---

# Verify

```bash
kubectl get pods
```

---

# Step 10: Enter Container 1

```bash
kubectl exec -it <pod-name> -c cont1 -- /bin/bash
```

---

Go to mounted volume:

```bash
cd /mycont1
```

Create file:

```bash
echo "Shared Data" > file1.txt
```

Verify:

```bash
ls
```

Output:

```text
file1.txt
```

Exit.

```bash
exit
```

---

# Step 11: Enter Container 2

```bash
kubectl exec -it <pod-name> -c cont2 -- /bin/bash
```

---

Go to:

```bash
cd /mycont2
```

Check:

```bash
ls
```

Output:

```text
file1.txt
```

---

# What Happened?

Container 1 created:

```text
file1.txt
```

Container 2 immediately accessed:

```text
file1.txt
```

Because both containers use:

```text
Same Shared Volume
```

---

# Important Limitation

This was the final concept explained in class.

Although emptyDir survives:

```text
Container Restart
```

it does NOT survive:

```text
Pod Deletion
```

---

Example:

```bash
kubectl delete pod <pod-name>
```

Result:

```text
Pod Deleted

Volume Deleted

Data Deleted
```

---

# Why Is This Still Not True Persistence?

Because:

```text
Volume Exists Inside Pod
```

When Pod disappears:

```text
Volume Disappears
```

---

# Real Production Storage

For permanent storage Kubernetes uses:

```text
Persistent Volume (PV)

Persistent Volume Claim (PVC)
```

Backed by:

```text
AWS EBS

AWS EFS

NFS

Azure Disk

Google Persistent Disk
```

These survive Pod deletion.

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

Describe Pod:

```bash
kubectl describe pod <pod-name>
```

---

Enter Container 1:

```bash
kubectl exec -it <pod-name> -c cont1 -- /bin/bash
```

---

Enter Container 2:

```bash
kubectl exec -it <pod-name> -c cont2 -- /bin/bash
```

---

Delete Pod:

```bash
kubectl delete pod <pod-name>
```

---

# Interview Questions

### What is a Kubernetes Volume?

A storage mechanism that allows containers to store and share data.

---

### Why are Volumes needed?

Because container storage is temporary and can be lost when containers restart or Pods are recreated.

---

### What is emptyDir?

A temporary volume created when a Pod starts and removed when the Pod is deleted.

---

### Can multiple containers share an emptyDir volume?

Yes.

---

### Does emptyDir survive container restart?

Yes.

---

### Does emptyDir survive Pod deletion?

No.

---

### What is the difference between container storage and volume storage?

```text
Container Storage
      |
      +--> Private
      +--> Temporary

Volume Storage
      |
      +--> Shared
      +--> Better Persistence
```

---

### What is the production-grade solution for persistent storage?

```text
Persistent Volume (PV)

Persistent Volume Claim (PVC)
```

---

# Summary

```text
Container Storage
       |
       +--> Temporary
       +--> Not Shared

Volume
       |
       +--> Shared Across Containers
       +--> Survives Container Restart
       +--> Supports Data Sharing

emptyDir
       |
       +--> Created With Pod
       +--> Deleted With Pod
       +--> Shared Storage
       +--> Temporary Persistence

Next Topic
       |
       v

Persistent Volume (PV)
       |
       v

Persistent Volume Claim (PVC)
```

```
```
