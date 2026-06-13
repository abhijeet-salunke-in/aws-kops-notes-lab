# Kubernetes DaemonSet

## Lab Overview

Before starting this lab, the cluster contained:

```text
1 Control Plane Node
1 Worker Node
```

Cluster Architecture:

```text
Cluster
│
├── Control Plane
└── Worker Node
```

During this lab:

* One additional Worker Node was added.
* A DaemonSet was created.
* DaemonSet automatically created one Pod on each Worker Node.

---

# What is a DaemonSet?

A DaemonSet is a Kubernetes workload object that ensures a Pod runs on every node in the cluster.

Whenever a new node joins the cluster:

```text
New Node Added
      ↓
DaemonSet Creates Pod Automatically
```

Whenever a node is removed:

```text
Node Removed
      ↓
Associated Pod Removed
```

DaemonSet automatically keeps node-level Pods running across the cluster.

---

# Why Do We Need DaemonSet?

Suppose we need:

```text
Monitoring Agent
Log Collection Agent
Security Agent
```

running on every node.

Examples:

```text
Prometheus Node Exporter
Fluentd
Filebeat
Datadog Agent
Falco
```

Using Deployment is not suitable because Deployment focuses on:

```text
Fixed Number Of Replicas
```

Example:

```yaml
replicas: 2
```

Deployment only guarantees:

```text
2 Pods
```

It does not guarantee:

```text
1 Pod Per Node
```

DaemonSet solves this problem.

---

# Problem Solved By DaemonSet

## Automatic Node Coverage

Instead of manually creating Pods on every node:

```text
Node-1 → Pod
Node-2 → Pod
Node-3 → Pod
```

DaemonSet automatically does it.

---

## Self-Healing

If a DaemonSet Pod crashes or is deleted:

```text
Pod Deleted
      ↓
DaemonSet Creates New Pod
```

---

## Automatic Scaling

When a new node joins:

```text
New Node
      ↓
New Pod Automatically Created
```

No manual action required.

---

# Cluster Used In This Lab

Initial Cluster:

```text
Control Plane
Worker-1
```

After Adding One Node:

```text
Control Plane
Worker-1
Worker-2
```

---

# Step 1: Configure AWS

```bash
aws configure
```

Provide:

```text
AWS Access Key
AWS Secret Key
Region
Output Format
```

Verify:

```bash
aws sts get-caller-identity
```

---

# Step 2: Configure KOPS State Store

```bash
export KOPS_STATE_STORE=s3://abhis.kops.v1
```

Verify:

```bash
echo $KOPS_STATE_STORE
```

Expected:

```text
s3://abhis.kops.v1
```

---

# Step 3: Check Existing Instance Groups

```bash
kops get ig --name=abhi.k8s.local
```

Example:

```text
NAME
control-plane-ap-south-1a
nodes-ap-south-1a
```

---

# Step 4: Add One More Worker Node

Edit worker node Instance Group:

```bash
kops edit ig nodes-ap-south-1a
```

Current:

```yaml
minSize: 1
maxSize: 1
```

Change to:

```yaml
minSize: 2
maxSize: 2
```

Save and exit.

---

# Step 5: Update Cluster

```bash
kops update cluster --name=abhi.k8s.local --yes
```

Expected:

```text
Cluster Changes Applied
```

---

# Step 6: Apply Rolling Update

```bash
kops rolling-update cluster --yes
```

Wait until update completes.

---

# Step 7: Verify Nodes

```bash
kubectl get nodes
```

Example:

```text
NAME                     STATUS
control-plane-node       Ready
worker-node-1            Ready
worker-node-2            Ready
```

Now the cluster contains:

```text
1 Control Plane
2 Worker Nodes
```

---

# Step 8: Create DaemonSet YAML

File:

```text
Daemon-demo.yml
```

Contents:

```yaml
apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: daemon-port-deploy

spec:
  selector:
    matchLabels:
      app: portfolio

  template:
    metadata:
      labels:
        app: portfolio

    spec:
      containers:
      - name: port-cont
        image: aarushtechnologies/portfolio:v1

        ports:
        - containerPort: 80
```

---

# Understanding The YAML

## apiVersion

```yaml
apiVersion: apps/v1
```

DaemonSet belongs to the Apps API Group.

---

## kind

```yaml
kind: DaemonSet
```

Creates a DaemonSet object.

---

## selector

```yaml
selector:
  matchLabels:
    app: portfolio
```

Used to identify Pods managed by the DaemonSet.

---

## template

```yaml
template:
```

Defines how Pods should be created.

---

## image

```yaml
image: aarushtechnologies/portfolio:v1
```

Container image used by every Pod.

---

# Step 9: Create DaemonSet

```bash
kubectl create -f Daemon-demo.yml
```

Expected:

```text
daemonset.apps/daemon-port-deploy created
```

---

# Step 10: Verify DaemonSet

```bash
kubectl get ds
```

Example:

```text
NAME                 DESIRED   CURRENT   READY
daemon-port-deploy   2         2         2
```

Explanation:

```text
DESIRED = Required Pods

CURRENT = Existing Pods

READY = Running Pods
```

Since there are:

```text
2 Worker Nodes
```

DaemonSet creates:

```text
2 Pods
```

---

# Step 11: Verify Pods

```bash
kubectl get pods
```

Example:

```text
daemon-port-deploy-rnlnz
daemon-port-deploy-t78z9
```

Status:

```text
Running
```

---

# Step 12: Verify Pod Placement

```bash
kubectl get pods -o wide
```

Example:

```text
NAME                        NODE

daemon-port-deploy-rnlnz    worker-node-1

daemon-port-deploy-t78z9    worker-node-2
```

Observation:

```text
One Pod Running On Each Worker Node
```

---

# Step 13: Verify Nodes

```bash
kubectl get nodes
```

Example:

```text
NAME
control-plane-node
worker-node-1
worker-node-2
```

---

# What Happened Internally?

DaemonSet checked:

```text
Worker Nodes = 2
```

Therefore:

```text
Required Pods = 2
```

DaemonSet automatically created:

```text
Pod-1 On Worker-1

Pod-2 On Worker-2
```

No replicas field was required.

---

# Self-Healing Demonstration

View Pods:

```bash
kubectl get pods
```

Delete one Pod:

```bash
kubectl delete pod daemon-port-deploy-rnlnz
```

Check again:

```bash
kubectl get pods
```

Result:

```text
New Pod Created Automatically
```

This proves:

```text
DaemonSet Self-Healing
```

---

# Automatic Node Addition Demonstration

Current Cluster:

```text
Control Plane
Worker-1
Worker-2
```

Add another Worker Node.

Verify:

```bash
kubectl get nodes
```

Example:

```text
Control Plane
Worker-1
Worker-2
Worker-3
```

Now check Pods:

```bash
kubectl get pods -o wide
```

Result:

```text
Worker-1 → Pod

Worker-2 → Pod

Worker-3 → Pod
```

DaemonSet automatically created a Pod on the new Worker Node.

---

# Useful Commands

View DaemonSets:

```bash
kubectl get ds
```

---

Describe DaemonSet:

```bash
kubectl describe ds daemon-port-deploy
```

---

View Pods:

```bash
kubectl get pods
```

---

View Pod Placement:

```bash
kubectl get pods -o wide
```

---

View Nodes:

```bash
kubectl get nodes
```

---

Delete Pod:

```bash
kubectl delete pod <pod-name>
```

---

Delete DaemonSet:

```bash
kubectl delete ds daemon-port-deploy
```

---

# Deployment vs DaemonSet

| Feature                 | Deployment | DaemonSet  |
| ----------------------- | ---------- | ---------- |
| Fixed Replicas          | Yes        | No         |
| One Pod Per Node        | No         | Yes        |
| Self-Healing            | Yes        | Yes        |
| Automatic Node Coverage | No         | Yes        |
| Application Hosting     | Yes        | Usually No |
| Monitoring Agents       | No         | Yes        |
| Log Collection Agents   | No         | Yes        |

---

# Interview Questions

### What is a DaemonSet?

A Kubernetes object that ensures one Pod runs on every node.

---

### Why is DaemonSet used?

To automatically run Pods on all nodes.

---

### Does DaemonSet require replicas?

```text
No
```

DaemonSet automatically determines the number of Pods based on the number of nodes.

---

### What happens when a new node joins the cluster?

```text
DaemonSet Automatically Creates A Pod
```

on that node.

---

### What is the main use case of DaemonSet?

```text
Monitoring Agents

Log Collection Agents

Security Agents
```

---

# Summary

```text
DaemonSet
      │
      ├── One Pod Per Node
      ├── Automatic Node Coverage
      ├── Self-Healing
      ├── Monitoring Agents
      ├── Log Collection Agents
      └── Security Agents
```

Workflow:

```text
Create DaemonSet
        │
        ▼
DaemonSet Creates Pod On Every Node
        │
        ▼
New Node Added
        │
        ▼
New Pod Created Automatically
        │
        ▼
Node Removed
        │
        ▼
Associated Pod Removed
```

This version matches your actual class flow: **started with 1 control-plane + 1 worker node, added a second worker node using KOPS, then created the DaemonSet and verified one pod per worker node.**
