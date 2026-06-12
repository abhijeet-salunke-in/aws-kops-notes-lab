# Kubernetes DaemonSet

## Overview

So far we have learned:

```text
Pod
 ↓
ReplicationController
 ↓
ReplicaSet
 ↓
Deployment
```

Each Kubernetes object solves a specific problem.

However, there are some workloads where we do not want a fixed number of Pods.

Instead, we want:

```text
One Pod Running On Every Node
```

This requirement is solved by **DaemonSet**.

---

# Problem Statement

Suppose a Kubernetes cluster contains:

```text
1 Control Plane
1 Worker Node
```

Architecture:

```text
Cluster
│
├── Control Plane
└── Worker Node
```

Now imagine we need:

```text
Log Collection Agent

OR

Monitoring Agent

OR

Security Agent
```

to run on every node.

Examples:

```text
Fluentd
Filebeat
Prometheus Node Exporter
Datadog Agent
Falco
```

Using a Deployment is not ideal because Deployment focuses on:

```text
Fixed Number Of Replicas
```

Example:

```yaml
replicas: 2
```

Kubernetes may schedule Pods anywhere.

We need a mechanism that guarantees:

```text
One Pod On Every Node
```

---

# What is a DaemonSet?

A DaemonSet is a Kubernetes workload object that ensures one Pod runs on every node in the cluster.

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
Create Pod On Node-1

Create Pod On Node-2

Create Pod On Node-3

...
```

Problems:

```text
Manual Work

Hard To Manage

Error Prone

Not Scalable
```

---

# Problems Solved By DaemonSet

### Automatic Node Coverage

```text
One Pod Per Node
```

---

### Self-Healing

```text
Pod Deleted
      ↓
DaemonSet Creates New Pod
```

---

### Automatic Node Scaling

```text
New Node Added
      ↓
DaemonSet Creates Pod
```

---

### Monitoring

Examples:

```text
Prometheus Node Exporter

Datadog Agent
```

---

### Log Collection

Examples:

```text
Fluentd

Filebeat
```

---

### Security Monitoring

Examples:

```text
Falco
```

---

# DaemonSet Architecture

Suppose cluster contains:

```text
1 Control Plane
2 Worker Nodes
```

```text
DaemonSet
      |
      +----------------------+
      |                      |
      v                      v

Worker Node-1          Worker Node-2
      |                      |
      v                      v

Daemon Pod            Daemon Pod
```

DaemonSet maintains:

```text
One Pod On Each Node
```

---

# Deployment vs DaemonSet

| Feature                    | Deployment    | DaemonSet  |
| -------------------------- | ------------- | ---------- |
| Fixed Replicas             | Yes           | No         |
| One Pod Per Node           | No            | Yes        |
| Self-Healing               | Yes           | Yes        |
| Scaling                    | Manual / Auto | Node Based |
| Used For Applications      | Yes           | Usually No |
| Used For Monitoring Agents | No            | Yes        |

---

# Practical Lab

## Cluster Used During Practice

Initially:

```text
1 Control Plane

1 Worker Node
```

During the lab, an additional worker node was added.

DaemonSet automatically created another Pod.

This demonstrates:

```text
Automatic Node Coverage
```

---

# Architecture Used

```text
DaemonSet
     |
     +-------------+
     |             |
     v             v

Node-1         Node-2
  |               |
  v               v

Pod             Pod
```

---

# Step 1: Create DaemonSet YAML

## File: daemon-demo.yml

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

# Understanding the YAML

## apiVersion

```yaml
apiVersion: apps/v1
```

DaemonSet belongs to:

```text
Apps API Group
```

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

Used to identify Pods managed by DaemonSet.

---

## template

```yaml
template:
```

Defines the Pod configuration.

---

## image

```yaml
image: aarushtechnologies/portfolio:v1
```

Container image used by all DaemonSet Pods.

---

# Step 2: Create DaemonSet

Execute:

```bash
kubectl create -f daemon-demo.yml
```

Expected:

```text
daemonset.apps/daemon-port-deploy created
```

---

# Common Error During Practice

While creating YAML:

```text
error converting YAML to JSON

yaml: line xx: could not find expected ':'
```

Reason:

```text
Incorrect YAML Indentation
```

Fix:

```text
Verify Spaces And Alignment Carefully
```

Then execute again:

```bash
kubectl create -f daemon-demo.yml
```

---

# Step 3: Verify DaemonSet

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

READY = Healthy Pods
```

---

# Step 4: Verify Pods

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

# Step 5: Verify Pod Placement

Execute:

```bash
kubectl get pods -o wide
```

Example:

```text
NAME                        NODE

daemon-port-deploy-rnlnz    Worker-Node-1

daemon-port-deploy-t78z9    Worker-Node-2
```

Observation:

```text
One Pod Running On Each Node
```

---

# Step 6: Verify Nodes

Execute:

```bash
kubectl get nodes
```

Example:

```text
NAME                     ROLES

control-plane-node       control-plane

worker-node-1            node

worker-node-2            node
```

---

# Understanding What Happened

Suppose cluster contains:

```text
2 Worker Nodes
```

DaemonSet creates:

```text
2 Pods
```

Suppose cluster grows to:

```text
5 Worker Nodes
```

DaemonSet automatically creates:

```text
5 Pods
```

No manual scaling required.

---

# Self-Healing Demonstration

View Pods:

```bash
kubectl get pods
```

Delete one Pod:

```bash
kubectl delete pod <pod-name>
```

Example:

```bash
kubectl delete pod daemon-port-deploy-rnlnz
```

Verify again:

```bash
kubectl get pods
```

Result:

```text
Deleted Pod Recreated Automatically
```

This proves:

```text
Self-Healing
```

---

# Automatic Node Addition Demonstration

Current Cluster:

```text
1 Control Plane

1 Worker Node
```

DaemonSet creates:

```text
1 Pod
```

Add another Worker Node.

Check:

```bash
kubectl get nodes
```

Now:

```text
1 Control Plane

2 Worker Nodes
```

Verify Pods:

```bash
kubectl get pods
```

Result:

```text
2 DaemonSet Pods
```

DaemonSet automatically creates a Pod on the new node.

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
kubectl delete -f daemon-demo.yml
```

OR

```bash
kubectl delete ds daemon-port-deploy
```

---

Verify:

```bash
kubectl get ds

kubectl get pods
```

Everything should be removed.

---

# Deployment vs ReplicaSet vs DaemonSet

| Feature             | ReplicaSet | Deployment | DaemonSet  |
| ------------------- | ---------- | ---------- | ---------- |
| Self-Healing        | Yes        | Yes        | Yes        |
| Fixed Replicas      | Yes        | Yes        | No         |
| Rolling Updates     | No         | Yes        | Limited    |
| Rollback            | No         | Yes        | No         |
| One Pod Per Node    | No         | No         | Yes        |
| Application Hosting | Yes        | Yes        | Usually No |
| Monitoring Agents   | No         | No         | Yes        |

---

# Interview Questions

### What is a DaemonSet?

A Kubernetes object that ensures one Pod runs on every node.

---

### Why is DaemonSet used?

To automatically run Pods on all nodes.

---

### What happens when a new node joins the cluster?

DaemonSet automatically creates a Pod on the new node.

---

### Does DaemonSet require replicas?

```text
No
```

DaemonSet automatically determines Pod count based on node count.

---

### What is the main use case of DaemonSet?

```text
Monitoring Agents

Log Collection Agents

Security Agents
```

---

### How is DaemonSet different from Deployment?

Deployment manages a fixed number of Pods.

DaemonSet ensures one Pod runs on every node.

---

# Summary

```text
DaemonSet
      |
      +--> One Pod Per Node
      |
      +--> Automatic Node Coverage
      |
      +--> Self-Healing
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
Node Removed
        |
        v
Associated Pod Removed
        |
        v
Cluster Remains Consistent
```
