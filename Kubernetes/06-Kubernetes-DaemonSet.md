# Kubernetes DaemonSet

## Topic Overview

Previously we learned:

```text
Pod
 ↓
Replication Controller
 ↓
ReplicaSet
 ↓
Deployment
```

Each Kubernetes object was created to solve a specific problem.

However, there are some workloads where we do not want:

```text
Fixed Number Of Pods
```

Instead, we want:

```text
One Pod Running On Every Node
```

This requirement is solved using:

```text
DaemonSet
```

---

# Cluster Used During Practice

Initially the cluster contained:

```text
1 Control Plane
1 Worker Node
```

Architecture:

```text
Kubernetes Cluster

├── Control Plane
└── Worker Node-1
```

During the practical session:

```text
One More Worker Node
```

was added.

Final Architecture:

```text
Kubernetes Cluster

├── Control Plane
├── Worker Node-1
└── Worker Node-2
```

This setup was used to demonstrate DaemonSet behavior.

---

# Problem Statement

Suppose we need:

```text
Monitoring Agent

OR

Log Collection Agent

OR

Security Agent
```

to run on every node.

Examples:

```text
Prometheus Node Exporter

Fluentd

Filebeat

Datadog Agent

Falco
```

Using Deployment is not ideal because Deployment only guarantees:

```text
Desired Number Of Pods
```

Example:

```yaml
replicas: 2
```

Pods may run anywhere.

Example:

```text
Node-1 → 2 Pods

Node-2 → 0 Pods
```

This does not solve our requirement.

We need:

```text
Exactly One Pod On Every Node
```

---

# What is DaemonSet?

A DaemonSet is a Kubernetes workload object that ensures:

```text
One Pod Runs On Every Node
```

Whenever:

```text
New Node Added
```

DaemonSet automatically creates a Pod on that node.

Whenever:

```text
Node Removed
```

DaemonSet automatically removes the associated Pod.

---

# Why Do We Need DaemonSet?

Without DaemonSet:

```text
Manually Create Pod On Node-1

Manually Create Pod On Node-2

Manually Create Pod On Node-3
```

Problems:

```text
Manual Work

Difficult Management

Not Scalable

Error Prone
```

DaemonSet automates this process.

---

# Problems Solved By DaemonSet

## 1. Automatic Node Coverage

```text
One Pod Per Node
```

No manual creation required.

---

## 2. Self-Healing

If a DaemonSet Pod crashes:

```text
Pod Deleted
      ↓
DaemonSet Detects
      ↓
New Pod Created
```

---

## 3. Automatic Node Scaling

When a new node joins:

```text
New Node Added
      ↓
DaemonSet Creates New Pod
```

Automatically.

---

## 4. Monitoring

Examples:

```text
Prometheus Node Exporter

Datadog Agent
```

---

## 5. Log Collection

Examples:

```text
Fluentd

Filebeat
```

---

## 6. Security Monitoring

Examples:

```text
Falco
```

---

# DaemonSet Architecture

Final cluster used during practice:

```text
Control Plane
Worker Node-1
Worker Node-2
```

DaemonSet Pods:

```text
Control Plane
      X

Worker Node-1
      |
      +---- Daemon Pod

Worker Node-2
      |
      +---- Daemon Pod
```

Observation:

```text
2 Worker Nodes

2 DaemonSet Pods
```

Exactly one Pod per worker node.

---

# Deployment vs DaemonSet

| Feature             | Deployment    | DaemonSet  |
| ------------------- | ------------- | ---------- |
| Fixed Replicas      | Yes           | No         |
| One Pod Per Node    | No            | Yes        |
| Self-Healing        | Yes           | Yes        |
| Scaling             | Replica Based | Node Based |
| Rollback            | Yes           | Limited    |
| Application Hosting | Yes           | Usually No |
| Monitoring Agents   | No            | Yes        |
| Log Collection      | No            | Yes        |

---

# Practical Lab

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

Output:

```text
control-plane

nodes-ap-south-1a
```

---

# Step 4: Add New Worker Node

Edit worker node instance group:

```bash
kops edit ig nodes-ap-south-1a
```

Change:

```yaml
minSize: 1
maxSize: 1
```

To:

```yaml
minSize: 2
maxSize: 2
```

Save and Exit.

---

# Step 5: Update Cluster

```bash
kops update cluster --yes
```

---

# Step 6: Perform Rolling Update

```bash
kops rolling-update cluster --yes
```

Wait until the new node is created.

---

# Step 7: Verify Nodes

```bash
kubectl get nodes
```

Expected:

```text
control-plane

worker-node-1

worker-node-2
```

---

# Step 8: Create Namespace

File:

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: devops
```

Apply:

```bash
kubectl create -f namespace.yml
```

Verify:

```bash
kubectl get ns
```

---

# Step 9: Create DaemonSet YAML

File:

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

## selector

```yaml
selector:
  matchLabels:
    app: portfolio
```

Used to identify Pods managed by DaemonSet.

---

## template

```yaml
template:
```

Defines Pod configuration.

---

## image

```yaml
image: aarushtechnologies/portfolio:v1
```

Container image used by all DaemonSet Pods.

---

# Step 10: Create DaemonSet

```bash
kubectl create -f Daemon-demo.yml
```

Expected:

```text
daemonset.apps/daemon-port-deploy created
```

---

# Step 11: Verify DaemonSet

```bash
kubectl get ds
```

Example:

```text
NAME                  DESIRED   CURRENT   READY

daemon-port-deploy    2         2         2
```

Meaning:

```text
DESIRED = Required Pods

CURRENT = Existing Pods

READY = Healthy Pods
```

---

# Step 12: Verify Pods

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

# Step 13: Verify Pod Placement

```bash
kubectl get pods -o wide
```

Example:

```text
daemon-port-deploy-rnlnz
Node = Worker Node-1

daemon-port-deploy-t78z9
Node = Worker Node-2
```

Observation:

```text
One Pod Running On Each Worker Node
```

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

Verify:

```bash
kubectl get pods
```

Result:

```text
New Pod Automatically Created
```

This proves:

```text
DaemonSet Self-Healing
```

---

# Automatic Node Addition Demonstration

Before:

```text
1 Control Plane

1 Worker Node
```

DaemonSet Pods:

```text
1
```

After adding a worker node:

```text
1 Control Plane

2 Worker Nodes
```

DaemonSet Pods:

```text
2
```

Observation:

```text
DaemonSet Automatically Created
A Pod On The New Node
```

---

# Real World Use Cases

## Fluentd

```text
Log Collection
```

---

## Filebeat

```text
Log Shipping
```

---

## Prometheus Node Exporter

```text
Node Monitoring
```

---

## Datadog Agent

```text
Infrastructure Monitoring
```

---

## Falco

```text
Runtime Security Monitoring
```

---

# Useful Commands

## View DaemonSets

```bash
kubectl get ds
```

---

## Detailed Information

```bash
kubectl describe ds daemon-port-deploy
```

---

## View Pods

```bash
kubectl get pods
```

---

## View Pod Placement

```bash
kubectl get pods -o wide
```

---

## View Nodes

```bash
kubectl get nodes
```

---

# Cleanup

Delete DaemonSet:

```bash
kubectl delete -f Daemon-demo.yml
```

OR

```bash
kubectl delete ds daemon-port-deploy
```

Verify:

```bash
kubectl get ds

kubectl get pods
```

Everything should be removed.

---

# Interview Questions

### What is a DaemonSet?

A Kubernetes object that ensures one Pod runs on every node.

---

### Why is DaemonSet used?

To automatically run Pods on all nodes.

---

### What happens when a new node joins the cluster?

DaemonSet automatically creates a Pod on that node.

---

### Does DaemonSet require replicas?

```text
No
```

DaemonSet automatically determines the number of Pods based on the number of nodes.

---

### What is the main use case of DaemonSet?

```text
Monitoring Agents

Log Collection Agents

Security Agents
```

---

### Difference Between Deployment and DaemonSet

Deployment:

```text
Maintains Fixed Number Of Pods
```

DaemonSet:

```text
Maintains One Pod Per Node
```

---

# Summary

```text
DaemonSet
      |
      +--> One Pod Per Node
      |
      +--> Self-Healing
      |
      +--> Automatic Node Coverage
      |
      +--> Monitoring Agents
      |
      +--> Log Collection Agents
      |
      +--> Security Agents
```

## Workflow

```text
Create DaemonSet
        |
        v
DaemonSet Creates Pod On Every Node
        |
        v
New Node Added
        |
        v
New Pod Created Automatically
        |
        v
Pod Deleted
        |
        v
DaemonSet Recreates Pod
        |
        v
Cluster Remains Consistent
```

This version matches **exactly what happened in your class**:

* Started with **1 control plane + 1 worker node**
* Added **1 new worker node using KOPS**
* Verified nodes
* Created Namespace
* Created DaemonSet
* Observed one Pod on each worker node
* Learned automatic node coverage and self-healing.
