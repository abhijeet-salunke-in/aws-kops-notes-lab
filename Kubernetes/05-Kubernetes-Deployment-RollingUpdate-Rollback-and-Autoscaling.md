# Kubernetes Deployment, Rolling Updates, Rollbacks and Autoscaling

## Overview

In the previous sections we learned:

```text
Pod
 ↓
ReplicationController
 ↓
ReplicaSet
```

Each step solved problems from the previous one.

---

## Problem with Standalone Pods

Suppose Kubernetes only had Pods.

```text
Pod
 └── nginx
```

If the Pod crashes:

```text
Pod ❌

Application ❌
```

Nobody recreates it.

Example:

```bash
kubectl delete pod nginx-pod
```

Result:

```text
Application Down
```

---

## Problem Solved by ReplicationController

ReplicationController introduced:

```text
Self-Healing
Scaling
High Availability
```

Example:

```yaml
replicas: 3
```

RC ensures:

```text
3 Pods Always Running
```

---

## Problem Solved by ReplicaSet

ReplicaSet improved RC.

It introduced advanced selectors.

RC only supports:

```yaml
selector:
  app: mobile
```

ReplicaSet supports:

```yaml
matchLabels:
```

and

```yaml
matchExpressions:
```

which provide more powerful Pod selection.

---

## New Problem

Suppose your application is running:

```text
aarushtechnologies/app
```

with:

```text
100 Pods
```

Now the company wants:

```text
aarushtechnologies/medical
```

Questions:

```text
How to update all Pods?

How to avoid downtime?

How to rollback if update fails?

How to keep version history?
```

ReplicaSet cannot manage these efficiently.

---

# Kubernetes Deployment

Deployment is a higher-level Kubernetes object.

Architecture:

```text
Deployment
      |
      v
ReplicaSet
      |
      v
Pods
```

Deployment manages ReplicaSets and Pods.

---

# What Problems Does Deployment Solve?

Deployment provides:

✅ Self-Healing

✅ Scaling

✅ High Availability

✅ Rolling Updates

✅ Rollbacks

✅ Version History

✅ Autoscaling Support

---

# Deployment Architecture

```text
Deployment
      |
      v
ReplicaSet
      |
      +-------------+
      |             |
      v             v
    Pod-1         Pod-2
```

---

# Practical Lab

## Scenario

Create:

```text
Deployment
      |
      v
2 Pods
      |
      v
LoadBalancer Service
```

---

# Step 1: Create Deployment YAML

## File: deploy1.yml

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: mob-deploy

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
      - name: mob-deploy-cont
        image: aarushtechnologies/app

        ports:
        - containerPort: 80
```

---

# Understanding Deployment YAML

## apiVersion

```yaml
apiVersion: apps/v1
```

Deployment belongs to the Apps API group.

---

## kind

```yaml
kind: Deployment
```

Creates a Deployment object.

---

## replicas

```yaml
replicas: 2
```

Maintain 2 Pods.

---

## selector

```yaml
selector:
  matchLabels:
    app: mobile
```

Deployment manages Pods having:

```yaml
app: mobile
```

---

## template

Defines Pod configuration.

```yaml
template:
```

Every Pod created by Deployment follows this template.

---

# Step 2: Create Deployment

```bash
kubectl create -f deploy1.yml
```

Expected:

```text
deployment.apps/mob-deploy created
```

---

# Step 3: Verify Deployment

```bash
kubectl get deploy
```

Example:

```text
NAME         READY
mob-deploy   2/2
```

---

# Step 4: Verify ReplicaSet

Deployment automatically creates a ReplicaSet.

```bash
kubectl get rs
```

Example:

```text
NAME
mob-deploy-5894dfbc7
```

---

# Step 5: Verify Pods

```bash
kubectl get pods
```

Example:

```text
mob-deploy-5894dfbc7-sg1cv
mob-deploy-5894dfbc7-z2f2m
```

Both should be:

```text
Running
```

---

# Self-Healing Demonstration

Delete a Pod.

Get Pods:

```bash
kubectl get pods
```

Delete one:

```bash
kubectl delete pod <pod-name>
```

Example:

```bash
kubectl delete pod mob-deploy-5894dfbc7-sg1cv
```

Check again:

```bash
kubectl get pods
```

You will notice:

```text
Deleted Pod Recreated Automatically
```

This proves:

```text
Deployment + ReplicaSet = Self-Healing
```

---

# Exposing Deployment Using LoadBalancer Service

## File: mob-deploy-svc.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mob-deploy-svc

spec:
  type: LoadBalancer

  selector:
    app: mobile

  ports:
  - port: 80
    targetPort: 80
```

---

# Create Service

```bash
kubectl create -f mob-deploy-svc.yml
```

Expected:

```text
service/mob-deploy-svc created
```

---

# Verify Service

```bash
kubectl get svc
```

Example:

```text
NAME             TYPE
mob-deploy-svc   LoadBalancer
```

Wait 2–5 minutes.

You will receive:

```text
EXTERNAL-IP
```

or

```text
AWS ELB DNS Name
```

---

# Test Application

Get Service Details:

```bash
kubectl get svc
```

Example:

```text
a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

Open Browser:

```text
http://a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

Expected:

```text
Application Homepage Loads
```

---

# Rolling Updates

One of the most important features of Deployment.

---

## Current Version

```text
aarushtechnologies/app
```

---

## Update Application Image

```bash
kubectl set image deployment/mob-deploy \
mob-deploy-cont=aarushtechnologies/medical
```

Expected:

```text
deployment.apps/mob-deploy image updated
```

---

# Verify Rolling Update

```bash
kubectl rollout status deployment/mob-deploy
```

Expected:

```text
deployment "mob-deploy" successfully rolled out
```

---

# Verify Pods

```bash
kubectl get pods
```

You will notice:

```text
Old Pods Removed

New Pods Created
```

without downtime.

---

# Deployment Revision History

Every update creates a revision.

View history:

```bash
kubectl rollout history deployment/mob-deploy
```

Example:

```text
REVISION
1
2
```

---

# Perform Another Update

```bash
kubectl set image deployment/mob-deploy \
mob-deploy-cont=aarushtechnologies/portfolio:v1
```

Check history again:

```bash
kubectl rollout history deployment/mob-deploy
```

Example:

```text
REVISION
1
2
3
```

---

# Rollback

Suppose latest version breaks.

Rollback to a previous working revision.

---

## Rollback to Revision 1

```bash
kubectl rollout undo deployment/mob-deploy \
--to-revision=1
```

---

## Rollback to Revision 3

```bash
kubectl rollout undo deployment/mob-deploy \
--to-revision=3
```

---

## Verify Rollback

```bash
kubectl get pods
```

New Pods will be created using the selected revision.

---

# Autoscaling (Horizontal Pod Autoscaler)

Manual scaling:

```bash
kubectl scale deployment mob-deploy --replicas=10
```

works, but production systems need automatic scaling.

---

# What is HPA?

HPA stands for:

```text
Horizontal Pod Autoscaler
```

It automatically increases or decreases Pods based on resource usage.

Common metrics:

```text
CPU Usage

Memory Usage
```

---

# Create Autoscaler

Command used in lab:

```bash
kubectl autoscale deployment/mob-deploy \
--min=5 \
--max=50
```

Expected:

```text
horizontalpodautoscaler.autoscaling/mob-deploy autoscaled
```

---

# Verify HPA

```bash
kubectl get hpa
```

Example:

```text
NAME
mob-deploy
```

---

# Verify Pods

```bash
kubectl get pods
```

Example:

```text
mob-deploy-5894dfbc7-bbvpb
mob-deploy-5894dfbc7-kgwmf
mob-deploy-5894dfbc7-ldqkz
mob-deploy-5894dfbc7-sg1cv
mob-deploy-5894dfbc7-z2f2m
```

Notice:

```text
5 Pods Running
```

because:

```text
Minimum Pods = 5
```

---

# Deployment vs ReplicaSet

| Feature                | ReplicaSet | Deployment |
| ---------------------- | ---------- | ---------- |
| Self-Healing           | Yes        | Yes        |
| Scaling                | Yes        | Yes        |
| High Availability      | Yes        | Yes        |
| Rolling Updates        | No         | Yes        |
| Rollbacks              | No         | Yes        |
| Version History        | No         | Yes        |
| Production Recommended | No         | Yes        |

---

# Useful Commands

## View Deployments

```bash
kubectl get deploy
```

---

## Detailed Deployment Information

```bash
kubectl describe deployment mob-deploy
```

---

## View ReplicaSets

```bash
kubectl get rs
```

---

## View Pods

```bash
kubectl get pods
```

---

## View Services

```bash
kubectl get svc
```

---

## Rollout History

```bash
kubectl rollout history deployment/mob-deploy
```

---

## Rollout Status

```bash
kubectl rollout status deployment/mob-deploy
```

---

## View HPA

```bash
kubectl get hpa
```

---

# Cleanup

Delete HPA:

```bash
kubectl delete hpa mob-deploy
```

---

Delete Service:

```bash
kubectl delete -f mob-deploy-svc.yml
```

---

Delete Deployment:

```bash
kubectl delete -f deploy1.yml
```

OR

```bash
kubectl delete deployment mob-deploy
```

---

Verify:

```bash
kubectl get deploy

kubectl get rs

kubectl get pods

kubectl get svc

kubectl get hpa
```

Everything should be removed.

---

# Evolution of Kubernetes Controllers

```text
Pod
│
├─ Problem:
│   No Self-Healing
│   No Scaling
│
▼

ReplicationController
│
├─ Solves:
│   Self-Healing
│   Scaling
│
├─ Problem:
│   Weak Selectors
│
▼

ReplicaSet
│
├─ Solves:
│   Better Selectors
│   Self-Healing
│   Scaling
│
├─ Problem:
│   No Rolling Updates
│   No Rollbacks
│
▼

Deployment
│
├─ Solves:
│   Rolling Updates
│   Rollbacks
│   Version History
│   Self-Healing
│   Scaling
│   Autoscaling
│
▼

Modern Kubernetes
```

---

# Interview Questions

### What is a Deployment?

A Deployment is a Kubernetes object that manages ReplicaSets and Pods.

---

### Why is Deployment preferred over ReplicaSet?

Because Deployment supports:

```text
Rolling Updates
Rollbacks
Version History
```

---

### How do you update a Deployment image?

```bash
kubectl set image deployment/mob-deploy \
mob-deploy-cont=<new-image>
```

---

### How do you check Deployment history?

```bash
kubectl rollout history deployment/mob-deploy
```

---

### How do you rollback a Deployment?

```bash
kubectl rollout undo deployment/mob-deploy --to-revision=1
```

---

### What is HPA?

Horizontal Pod Autoscaler automatically scales Pods based on resource utilization.

---

# Summary

```text
Deployment
      |
      +--> Manages ReplicaSets
      |
      +--> Self-Healing
      |
      +--> Scaling
      |
      +--> Rolling Updates
      |
      +--> Rollbacks
      |
      +--> Version History
      |
      +--> Autoscaling (HPA)
```

```text
Create Deployment
       |
       v
Deployment Creates ReplicaSet
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
Update Application
       |
       v
Rolling Update
       |
       v
Rollback if Needed
       |
       v
Autoscale Automatically
```
