# Kubernetes HostPath Volume and Node Level Persistence (on node)

## Overview

In the previous topic, we learned about:

```text
emptyDir Volume
```

EmptyDir allows containers inside the same Pod to share data.

However:

```text
Pod Deleted
     |
     v
emptyDir Deleted
     |
     v
Data Lost
```

This creates a problem.

Many applications require data to survive beyond container restarts and Pod recreation.

Examples:

```text
Application Logs
Configuration Files
Cache Files
Reports
Temporary Databases
Shared Files
```

Kubernetes provides another volume type called:

```text
hostPath
```

which stores data directly on the Kubernetes Node.

---

# Understanding the Storage Problem

## Container Filesystem

Suppose a container creates a file:

```text
/var/data/report.txt
```

Architecture:

```text
Container
     |
     v
Container Filesystem
```

Problem:

```text
Container Deleted
     |
     v
Filesystem Deleted
     |
     v
Data Lost
```

---

## EmptyDir Improvement

EmptyDir solved container-level storage problems.

```text
Pod
 |
 +------------+
 |            |
 | Container1 |
 | Container2 |
 +-----+------+
       |
       v
    emptyDir
```

Advantages:

```text
Shared Storage
Container Restart Safe
```

Problem:

```text
Pod Deleted
     |
     v
emptyDir Deleted
     |
     v
Data Lost
```

---

# Kubernetes Solution

Kubernetes introduced:

```text
hostPath Volume
```

Instead of storing data inside the Pod:

```text
Pod
 |
 v
Node Filesystem
```

Data is stored on the Kubernetes worker node.

---

# What is HostPath?

HostPath is a Kubernetes volume type that mounts a directory from the Node's filesystem into a Pod.

In simple words:

```text
Node Folder
     |
     v
Mounted Inside Container
```

Example:

Node Directory:

```text
/mydata
```

Container Mount:

```text
/mycont1
```

Both point to the same location.

---

# HostPath Architecture

```text
Kubernetes Node
|
+-----------------------------------+
|                                   |
|   /mydata                         |
|                                   |
+----------------+------------------+
                 |
                 |
                 v

          Kubernetes Pod
                 |
      +----------+----------+
      |                     |
      v                     v
   cont1                 cont2
      |                     |
      +----------+----------+
                 |
                 v
             hostPath
```

Both containers access the same node directory.

---

# Why Do We Need HostPath?

Suppose:

```text
Application Generates Logs
```

Without HostPath:

```text
Pod Deleted
     |
     v
Logs Deleted
```

With HostPath:

```text
Logs Stored On Node
     |
     v
Pod Deleted
     |
     v
Logs Still Exist
```

---

# HostPath Lifecycle

## Pod Creation

```text
Pod Starts
```

Kubernetes mounts:

```text
/mydata
```

into containers.

---

## Container Usage

Containers:

```text
Read Files
Write Files
Modify Files
```

inside mounted directory.

---

## Pod Deletion

```text
Pod Deleted
```

But:

```text
Node Directory Still Exists
```

Therefore:

```text
Data Still Exists
```

---

# HostPath vs EmptyDir

## EmptyDir

```text
Pod Created
     |
     v
Volume Created
     |
     v
Pod Deleted
     |
     v
Volume Deleted
```

Data Lost.

---

## HostPath

```text
Node Directory Exists
     |
     v
Pod Uses Directory
     |
     v
Pod Deleted
     |
     v
Directory Remains
```

Data Preserved.

---

# Important Limitation

HostPath stores data on a specific node.

Example:

```text
Node-1
 |
 +--> /mydata
```

If Pod moves to:

```text
Node-2
```

then:

```text
Node-2
 |
 +--> /mydata
```

may be empty.

Therefore:

```text
HostPath Is Node Dependent
```

---

# Practical Lab

## Architecture

```text
Deployment
      |
      v
      Pod
      |
+-------------+
|             |
v             v

cont1      cont2
  \          /
   \        /
    \      /
     \    /
      \  /
       \/
   hostPath
      |
      v
    /mydata
```

---

# Step 1: Create Deployment YAML

## File: hostpath-vol.yml

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
        hostPath:
          path: "/mydata"

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

# Understanding the YAML

## Volume Section

```yaml
volumes:
- name: myvolume2
  hostPath:
    path: "/mydata"
```

Meaning:

```text
Node Directory = /mydata
```

Kubernetes will mount this path into containers.

---

## Container 1 Mount

```yaml
volumeMounts:
- name: myvolume2
  mountPath: /mycont1
```

Container sees:

```text
/mycont1
```

---

## Container 2 Mount

```yaml
volumeMounts:
- name: myvolume2
  mountPath: /mycont2
```

Container sees:

```text
/mycont2
```

Both point to:

```text
Node Directory /mydata
```

---

# Step 2: Create Deployment

```bash
kubectl create -f hostpath-vol.yml
```

Expected:

```text
deployment.apps/deploy2 created
```

---

# Step 3: Verify Deployment

```bash
kubectl get deploy
```

Expected:

```text
NAME      READY
deploy2   1/1
```

---

# Step 4: Verify Pod

```bash
kubectl get pods
```

Example:

```text
deploy2-7d6db774b7-kpmng
```

---

# Step 5: Enter Container 1

```bash
kubectl exec -it deploy2-7d6db774b7-kpmng -c cont1 -- /bin/bash
```

Move to mount path:

```bash
cd /mycont1
```

Create file:

```bash
echo "Hello HostPath" > test.txt
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

# Step 6: Enter Container 2

```bash
kubectl exec -it deploy2-7d6db774b7-kpmng -c cont2 -- /bin/bash
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
Hello HostPath
```

This proves:

```text
Both Containers Access Same Node Storage
```

---

# Step 7: Delete Pod

Find Pod:

```bash
kubectl get pods
```

Delete:

```bash
kubectl delete pod <pod-name>
```

Example:

```bash
kubectl delete pod deploy2-7d6db774b7-kpmng
```

Deployment creates a new Pod automatically.

---

# Step 8: Verify Data Persistence

Get new Pod:

```bash
kubectl get pods
```

Enter Container:

```bash
kubectl exec -it <new-pod-name> -c cont1 -- /bin/bash
```

Check:

```bash
cd /mycont1

ls
```

Expected:

```text
test.txt
```

Data still exists.

Reason:

```text
Data Stored On Node
Not Inside Pod
```

---

# Advantages of HostPath

## Simple Configuration

Easy to understand and configure.

---

## Node-Level Storage

Data survives Pod recreation.

---

## Shared Storage

Multiple containers can share files.

---

## Useful For Testing

Commonly used in labs and learning environments.

---

## Log Collection

Applications can store logs on the node.

---

# Disadvantages of HostPath

## Node Dependency

Data exists only on one node.

---

## Not Portable

Pod moved to another node may not find data.

---

## Security Risk

Containers gain access to node filesystem.

---

## Not Recommended For Production

Usually replaced by:

```text
Persistent Volumes
Persistent Volume Claims
AWS EBS
AWS EFS
NFS
CSI Drivers
```

---

# EmptyDir vs HostPath

| Feature                    | EmptyDir | HostPath |
| -------------------------- | -------- | -------- |
| Shared Between Containers  | Yes      | Yes      |
| Survives Container Restart | Yes      | Yes      |
| Survives Pod Deletion      | No       | Yes      |
| Storage Location           | Pod      | Node     |
| Node Dependent             | No       | Yes      |
| Production Friendly        | Limited  | Rarely   |

---

# HostPath vs Persistent Volumes

| Feature           | HostPath | Persistent Volume |
| ----------------- | -------- | ----------------- |
| Node Specific     | Yes      | No                |
| Production Use    | Rare     | Common            |
| Cloud Integration | No       | Yes               |
| Scalability       | Limited  | High              |
| Reliability       | Low      | High              |

---

# Real World Usage

HostPath is commonly used for:

```text
Learning Kubernetes
Testing Labs
Local Development
Log Collection
Node Diagnostics
Monitoring Agents
```

Examples:

```text
Prometheus Node Exporter
Log Collectors
System Monitoring Tools
```

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
kubectl describe deploy deploy2
```

Enter Container:

```bash
kubectl exec -it <pod-name> -c cont1 -- /bin/bash
```

Delete Deployment:

```bash
kubectl delete deploy deploy2
```

---

# Interview Questions

### What is HostPath?

A volume type that mounts a directory from the Kubernetes node into a Pod.

---

### Where is HostPath data stored?

On the worker node filesystem.

---

### Does HostPath survive Pod deletion?

Yes.

---

### Is HostPath node dependent?

Yes.

---

### Can multiple containers share a HostPath volume?

Yes.

---

### Why is HostPath not preferred in production?

Because it is tied to a single node and creates portability and security concerns.

---

### What is the modern production alternative?

```text
Persistent Volumes (PV)
Persistent Volume Claims (PVC)
AWS EBS
AWS EFS
```

---

# Summary

```text
Kubernetes Volumes
       |
       +--> emptyDir
       |      |
       |      +--> Temporary
       |      +--> Deleted With Pod
       |
       +--> hostPath
              |
              +--> Node Storage
              +--> Survives Pod Recreation
              +--> Shared Between Containers
              +--> Node Dependent
```

## Workflow

```text
Create Deployment
       |
       v
Create hostPath Volume
       |
       v
Mount Volume Into Containers
       |
       v
Create File In cont1
       |
       v
Read Same File In cont2
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
Node Storage Persistence Verified
```

### Next Topic

```text
Persistent Volume (PV)
        |
        v
Persistent Volume Claim (PVC)
        |
        v
Production Grade Storage
        |
        v
Cloud Storage Integration
```
