# Kubernetes ReplicaSet (RS) and Scaling

## Introduction

In the previous topic, we learned about ReplicationController (RC).

ReplicationController solved several important problems:

- Self-Healing
- High Availability
- Scaling
- Desired State Management

However, RC had an important limitation.

It only supported simple label selectors.

To overcome this limitation, Kubernetes introduced:

```text
ReplicaSet (RS)
```

ReplicaSet is considered the next generation of ReplicationController.

Today, ReplicaSet is the standard controller used internally by Deployments.

---

# Evolution from Pod to ReplicaSet

Understanding ReplicaSet becomes easier when we understand how Kubernetes evolved.

---

# Stage 1: Standalone Pods

Suppose Kubernetes only had Pods.

```text
Pod
 └── nginx
```

Everything works fine until the Pod fails.

Example:

```bash
kubectl delete pod nginx-pod
```

Result:

```text
Pod Deleted
Application Down
```

Nobody recreates it automatically.

---

# Problems with Standalone Pods

## Problem 1: No Self-Healing

```text
Pod Crash
    ↓
Application Down
```

---

## Problem 2: No High Availability

Single Pod:

```text
1 Pod
```

If it fails:

```text
Application Down
```

---

## Problem 3: Manual Scaling

Need:

```text
10 Pods
```

Must create:

```text
Pod1
Pod2
Pod3
...
Pod10
```

Manually.

---

# Kubernetes Team's First Solution

To solve these problems Kubernetes introduced:

```text
ReplicationController (RC)
```

RC continuously checks:

```text
Desired Pods
vs
Actual Pods
```

Example:

```yaml
replicas: 3
```

If one Pod dies:

```text
Desired = 3
Actual = 2
```

RC creates a new Pod automatically.

---

# Problems Solved by RC

## Self-Healing

```text
Pod Dies
    ↓
RC Creates New Pod
```

---

## Scaling

Need:

```text
5 Pods
```

Update:

```yaml
replicas: 5
```

RC creates additional Pods.

---

## High Availability

Instead of:

```text
1 Pod
```

Run:

```text
3 Pods
```

Application survives failures.

---

# Why Kubernetes Didn't Stop at RC

RC had a major limitation.

Its selector capability was very basic.

Example:

```yaml
selector:
  app: mobile
```

Meaning:

```text
app = mobile
```

Only exact matching.

---

# RC Selector Limitations

RC cannot perform:

```text
nginx OR apache
```

---

RC cannot perform:

```text
NOT mysql
```

---

RC cannot perform:

```text
frontend AND production
```

Advanced applications needed more flexible selection mechanisms.

---

# Kubernetes Team's Next Solution

To solve selector limitations Kubernetes introduced:

```text
ReplicaSet
```

Think of ReplicaSet as:

```text
ReplicationController V2
```

---

# What is ReplicaSet?

ReplicaSet is a Kubernetes controller responsible for maintaining a specified number of Pod replicas.

Its responsibilities include:

- Self-Healing
- High Availability
- Scaling
- Advanced Label Selection

ReplicaSet continuously ensures:

```text
Desired Pods = Actual Pods
```

---

# How ReplicaSet Works

Suppose:

```yaml
replicas: 2
```

Current state:

```text
Pod-1 Running
Pod-2 Running
```

---

Suppose Pod-2 crashes:

```text
Pod-1 Running
Pod-2 Deleted
```

Now:

```text
Desired = 2
Actual = 1
```

ReplicaSet immediately creates:

```text
Pod-3 Running
```

Result:

```text
2 Pods Running Again
```

---

# Benefits of ReplicaSet

## Self-Healing

```text
Pod Deleted
      ↓
ReplicaSet Creates New Pod
```

---

## High Availability

Multiple Pods increase application availability.

---

## Scaling

ReplicaSet can increase or decrease Pod count.

---

## Desired State Management

ReplicaSet continuously ensures actual state matches desired state.

---

# ReplicaSet Architecture

```text
ReplicaSet
      |
      |
      +----------------+
      |                |
      v                v
    Pod-1           Pod-2
```

ReplicaSet continuously monitors and manages Pods.

---

# ReplicaSet Components

## replicas

Defines how many Pods should run.

Example:

```yaml
replicas: 2
```

---

## selector

Used to identify Pods managed by ReplicaSet.

Example:

```yaml
selector:
  matchLabels:
    app: mobile
```

---

## template

Defines Pod configuration.

Example:

```yaml
template:
  metadata:
    labels:
      app: mobile
```

---

# Advanced Selectors in ReplicaSet

ReplicaSet introduced:

```yaml
matchLabels
```

and

```yaml
matchExpressions
```

---

## Example: matchLabels

```yaml
selector:
  matchLabels:
    app: mobile
```

Matches:

```yaml
app: mobile
```

---

## Example: matchExpressions

```yaml
matchExpressions:
- key: app
  operator: In
  values:
  - nginx
  - apache
```

Result:

```text
nginx  ✅
apache ✅
mysql  ❌
```

This was not possible with ReplicationController.

---

# Practical Lab

## Objective

Create:

```text
ReplicaSet
     |
     +------+
     |      |
     v      v
   Pod1   Pod2
```

and expose the application using a LoadBalancer Service.

---

# Step 1: Create ReplicaSet YAML

## File: my-rs.yml

```yaml
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: myrs

spec:
  replicas: 2

  selector:
    matchLabels:
      app: mobile

  template:
    metadata:
      labels:
        app: mobile

    spec:
      containers:
      - name: mob-cont
        image: aarushtechnologies/app

        ports:
        - containerPort: 80
```

---

# YAML Explanation

## API Version

```yaml
apiVersion: apps/v1
```

ReplicaSet belongs to the Apps API group.

---

## Kind

```yaml
kind: ReplicaSet
```

Creates a ReplicaSet object.

---

## Metadata

```yaml
metadata:
  name: myrs
```

ReplicaSet name:

```text
myrs
```

---

## Replicas

```yaml
replicas: 2
```

Maintain two Pods.

---

## Selector

```yaml
selector:
  matchLabels:
    app: mobile
```

Identifies Pods managed by ReplicaSet.

---

## Template

Defines Pod configuration.

Every Pod created by ReplicaSet follows this template.

---

# Step 2: Create ReplicaSet

```bash
kubectl create -f my-rs.yml
```

Expected:

```text
replicaset.apps/myrs created
```

---

# Step 3: Verify ReplicaSet

```bash
kubectl get rs
```

Expected:

```text
NAME   DESIRED   CURRENT   READY
myrs   2         2         2
```

Meaning:

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
myrs-jx5kt
myrs-lf7hp
```

Both Pods should be:

```text
Running
```

---

# Self-Healing Demonstration

## Step 1: View Pods

```bash
kubectl get pods
```

---

## Step 2: Delete a Pod

Example:

```bash
kubectl delete pod myrs-jx5kt
```

Expected:

```text
pod deleted
```

---

## Step 3: Verify Again

```bash
kubectl get pods
```

Example:

```text
myrs-lf7hp
myrs-r8w4t
```

Notice:

```text
Deleted Pod Recreated Automatically
```

This proves:

```text
Self-Healing
```

---

# Exposing ReplicaSet Using Service

Pods require a Service for stable access.

---

# Step 1: Create Service YAML

## File: myrs-svc.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mobile-svc

spec:
  type: LoadBalancer

  selector:
    app: mobile

  ports:
  - port: 80
    targetPort: 80
```

---

# Service YAML Explanation

## Type

```yaml
type: LoadBalancer
```

Creates AWS Elastic Load Balancer.

---

## Selector

```yaml
selector:
  app: mobile
```

Matches Pods managed by ReplicaSet.

---

## Port Mapping

```yaml
port: 80
targetPort: 80
```

Traffic received on port 80 forwards to container port 80.

---

# Step 2: Create Service

```bash
kubectl create -f myrs-svc.yml
```

Expected:

```text
service/mobile-svc created
```

---

# Step 3: Verify Service

```bash
kubectl get svc
```

Example:

```text
NAME         TYPE           CLUSTER-IP
mobile-svc   LoadBalancer   100.69.20.84
```

Initially:

```text
EXTERNAL-IP
<pending>
```

Wait a few minutes.

---

# Step 4: Verify Load Balancer

```bash
kubectl get svc
```

Example:

```text
mobile-svc

LoadBalancer

100.69.20.84

a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

---

# Step 5: Verify in AWS

Navigate:

```text
AWS Console
→ EC2
→ Load Balancers
```

You should see a new AWS Load Balancer.

---

# Step 6: Test Application

Copy:

```text
EXTERNAL-IP
```

Example:

```text
a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

Open:

```text
http://a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

Expected:

```text
Application Homepage Loads
```

---

# Scaling ReplicaSet

Scaling means changing the number of Pod replicas.

---

# Scale Up

Current:

```text
2 Pods
```

Increase to:

```text
5 Pods
```

Command:

```bash
kubectl scale rs myrs --replicas=5
```

Verify:

```bash
kubectl get rs
```

Expected:

```text
NAME   DESIRED   CURRENT   READY
myrs   5         5         5
```

---

Check Pods:

```bash
kubectl get pods
```

You should now see:

```text
5 Running Pods
```

---

# Scale Down

Current:

```text
5 Pods
```

Reduce to:

```text
2 Pods
```

Command:

```bash
kubectl scale rs myrs --replicas=2
```

Verify:

```bash
kubectl get pods
```

Expected:

```text
2 Running Pods
```

---

# Generic Scaling Command

```bash
kubectl scale rs <replicaset-name> --replicas=<number>
```

Example:

```bash
kubectl scale rs myrs --replicas=10
```

---

# Orphan Deletion Demonstration

One concept practiced during the lab:

---

## Normal Deletion

```bash
kubectl delete rs myrs
```

Result:

```text
ReplicaSet Deleted
Pods Deleted
```

---

## Orphan Deletion

```bash
kubectl delete replicaset myrs --cascade=orphan
```

Result:

```text
ReplicaSet Deleted
Pods Remain Running
```

Useful when Pods must continue running after ReplicaSet removal.

---

# Limitations of ReplicaSet

ReplicaSet successfully provides:

- Self-Healing
- Scaling
- High Availability

However, ReplicaSet does not manage:

- Rolling Updates
- Rollbacks
- Version History
- Deployment Strategies

---

## Example

Current image:

```text
nginx:1.25
```

Need:

```text
nginx:1.26
```

ReplicaSet can maintain Pods but does not provide elegant upgrade management.

---

# Why Deployment Was Introduced

Kubernetes introduced:

```text
Deployment
```

Think of Deployment as:

```text
Manager of ReplicaSets
```

Deployment provides:

- Rolling Updates
- Rollbacks
- Version History
- Deployment Strategies
- Self-Healing
- Scaling

---

# Evolution of Kubernetes Controllers

```text
Pod
│
├─ Problem:
│   No self-healing
│   No scaling
│
▼

ReplicationController
│
├─ Solves:
│   Self-healing
│   Scaling
│
├─ Problem:
│   Simple selectors
│
▼

ReplicaSet
│
├─ Solves:
│   Better selectors
│   Self-healing
│   Scaling
│
├─ Problem:
│   No rolling updates
│   No rollback
│
▼

Deployment
│
├─ Solves:
│   Rolling updates
│   Rollbacks
│   Version history
│
▼

Modern Kubernetes
```

---

# Useful Commands

View ReplicaSets:

```bash
kubectl get rs
```

---

Describe ReplicaSet:

```bash
kubectl describe rs myrs
```

---

View Pods:

```bash
kubectl get pods
```

---

View Services:

```bash
kubectl get svc
```

---

View Endpoints:

```bash
kubectl get endpoints
```

---

List Kubernetes Resources:

```bash
kubectl api-resources
```

---

# Cleanup

Delete Service:

```bash
kubectl delete -f myrs-svc.yml
```

Delete ReplicaSet:

```bash
kubectl delete -f my-rs.yml
```

OR

```bash
kubectl delete rs myrs
```

Verify:

```bash
kubectl get rs
kubectl get pods
kubectl get svc
```

---

# Interview Questions

### What is a ReplicaSet?

A Kubernetes controller that maintains a desired number of Pod replicas.

---

### Why was ReplicaSet introduced?

To overcome the selector limitations of ReplicationController.

---

### What is matchLabels?

A selector used to identify Pods based on labels.

---

### What is matchExpressions?

An advanced selector that supports complex matching conditions.

---

### What happens if a Pod managed by ReplicaSet is deleted?

ReplicaSet automatically creates a replacement Pod.

---

### How do you scale a ReplicaSet?

```bash
kubectl scale rs myrs --replicas=5
```

---

### Why is ReplicaSet rarely created directly in production?

Because Deployments manage ReplicaSets automatically.

---

### What is the relationship between Deployment and ReplicaSet?

Deployment manages ReplicaSets, and ReplicaSets manage Pods.

---

# Summary

```text
ReplicaSet
      |
      +--> Maintains Desired Number of Pods
      |
      +--> Provides Self-Healing
      |
      +--> Supports Scaling
      |
      +--> Improves High Availability
      |
      +--> Supports Advanced Selectors
```

## Workflow

```text
Create ReplicaSet
        |
        v
ReplicaSet Creates Pods
        |
        v
Create Service
        |
        v
AWS LoadBalancer Created
        |
        v
Application Accessible
        |
        v
Scale Pods Up/Down
        |
        v
ReplicaSet Maintains Desired State
```
