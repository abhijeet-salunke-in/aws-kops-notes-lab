# Recommended File Name

```text
13-Kubernetes/03-Kubernetes-ReplicationController-and-Scaling.md
```

---

# Kubernetes ReplicationController (RC) and Scaling

## Overview

A Pod is the smallest deployable unit in Kubernetes.

Problem:

If a Pod crashes, gets deleted, or the node fails:

```text
Application Becomes Unavailable
```

Kubernetes solves this problem using a **ReplicationController (RC)**.

A ReplicationController ensures that a specified number of Pod replicas are always running.

---

# What is a ReplicationController?

A ReplicationController continuously monitors Pods and ensures the desired number of replicas are running.

Example:

```text
Desired Replicas = 3
```

Current State:

```text
Pod-1 Running
Pod-2 Running
Pod-3 Running
```

If Pod-2 crashes:

```text
Pod-1 Running
Pod-2 Deleted
Pod-3 Running
```

ReplicationController automatically creates:

```text
Pod-4 Running
```

Result:

```text
3 Pods Running Again
```

---

# Why Do We Need ReplicationController?

Without RC:

```text
Pod Crashes
    |
    v
Application Down
```

With RC:

```text
Pod Crashes
    |
    v
RC Detects Failure
    |
    v
New Pod Created Automatically
```

Benefits:

* High Availability
* Self-Healing
* Scaling Support
* Load Distribution
* Fault Tolerance

---

# ReplicationController Architecture

```text
ReplicationController
        |
        |
        +----------------+
        |                |
        v                v
      Pod-1           Pod-2
                         |
                         v
                       Pod-3
```

RC continuously maintains the desired state.

---

# Important Components of RC

## replicas

Defines how many Pods should run.

Example:

```yaml
replicas: 3
```

---

## selector

Used to identify Pods managed by RC.

Example:

```yaml
selector:
  app: mobile
```

---

## template

Defines the Pod configuration.

Example:

```yaml
template:
  metadata:
    labels:
      app: mobile
```

---

# Practical Lab: Create ReplicationController

## Architecture

```text
ReplicationController
         |
         v
      3 Pods
         |
         v
Application Container
```

---

# Step 1: Create ReplicationController YAML

## File: my-rc.yml

```yaml
apiVersion: v1
kind: ReplicationController

metadata:
  name: myrc

spec:
  replicas: 3

  selector:
    app: mobile

  template:
    metadata:
      labels:
        app: mobile

    spec:
      containers:
      - name: mobile-container
        image: aarushtechnologies/app

        ports:
        - containerPort: 80
```

---

# Step 2: Create ReplicationController

```bash
kubectl create -f my-rc.yml
```

Expected:

```text
replicationcontroller/myrc created
```

---

# Step 3: Verify RC

```bash
kubectl get rc
```

Example:

```text
NAME   DESIRED   CURRENT   READY
myrc   3         3         3
```

---

# Step 4: Verify Pods

```bash
kubectl get pods
```

Example:

```text
myrc-4zmqf
myrc-7js84
myrc-ttscq
```

All Pods should be:

```text
Running
```

---

# Self-Healing Demonstration

## Step 1: Delete a Pod

Get Pods:

```bash
kubectl get pods
```

Delete one Pod:

```bash
kubectl delete pod <pod-name>
```

Example:

```bash
kubectl delete pod myrc-4zmqf
```

---

## Step 2: Verify Again

```bash
kubectl get pods
```

You will notice:

```text
Old Pod Deleted
New Pod Created Automatically
```

Example:

```text
myrc-7js84
myrc-ttscq
myrc-kd9w2
```

This proves RC provides:

```text
Self-Healing
```

---

# Exposing RC Using LoadBalancer Service

Since RC creates multiple Pods, users need a Service to access them.

---

# Step 1: Create LoadBalancer Service YAML

## File: mobile-svc.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mobile-service

spec:
  type: LoadBalancer

  selector:
    app: mobile

  ports:
  - port: 80
    targetPort: 80
```

Notice:

```yaml
selector:
  app: mobile
```

This selector matches the Pods created by RC.

---

# Step 2: Create Service

```bash
kubectl create -f mobile-svc.yml
```

Expected:

```text
service/mobile-service created
```

---

# Step 3: Verify Service

```bash
kubectl get svc
```

Example:

```text
NAME             TYPE           CLUSTER-IP
mobile-service   LoadBalancer   100.69.20.84
```

After a few minutes:

```text
EXTERNAL-IP

a177146dd8fd8432cb4a87af79...
ap-south-1.elb.amazonaws.com
```

---

# Step 4: Verify Load Balancer in AWS

Navigate:

```text
AWS Console
→ EC2
→ Load Balancers
```

You should see a new AWS Load Balancer created automatically.

---

# Step 5: Test Application

Get ELB DNS:

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

# Scaling ReplicationController

One of the biggest advantages of RC is scaling.

Scaling means increasing or decreasing the number of Pods.

---

# Scale Up

Current:

```text
3 Pods
```

Increase to 5 Pods:

```bash
kubectl scale rc myrc --replicas=5
```

Verify:

```bash
kubectl get rc
```

Expected:

```text
NAME   DESIRED   CURRENT   READY
myrc   5         5         5
```

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

Reduce to 2 Pods:

```bash
kubectl scale rc myrc --replicas=2
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
kubectl scale rc <rc-name> --replicas=<number>
```

Example:

```bash
kubectl scale rc myrc --replicas=10
```

---

# Useful Commands

## View RC

```bash
kubectl get rc
```

---

## Detailed RC Information

```bash
kubectl describe rc myrc
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

## View Endpoints

```bash
kubectl get endpoints
```

---

# Cleanup

Delete Service:

```bash
kubectl delete -f mobile-svc.yml
```

---

Delete RC:

```bash
kubectl delete -f my-rc.yml
```

OR

```bash
kubectl delete rc myrc
```

---

Verify:

```bash
kubectl get rc

kubectl get pods

kubectl get svc
```

Everything should be removed.

---

# ReplicationController vs Standalone Pod

| Feature           | Pod | ReplicationController |
| ----------------- | --- | --------------------- |
| Single Instance   | Yes | No                    |
| Self Healing      | No  | Yes                   |
| Auto Recreation   | No  | Yes                   |
| Scaling           | No  | Yes                   |
| High Availability | No  | Yes                   |

---

# Important Notes

### ReplicationController is an older Kubernetes object.

Modern Kubernetes generally uses:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pods
```

However, ReplicationController is still important for understanding Kubernetes fundamentals and is commonly asked in interviews.

---

# Interview Questions

### What is a ReplicationController?

A Kubernetes controller that ensures a specified number of Pod replicas are always running.

---

### What is the purpose of replicas?

To define how many Pod copies should run.

---

### What happens if a Pod managed by RC is deleted?

ReplicationController automatically creates a new Pod.

---

### How do you scale a ReplicationController?

```bash
kubectl scale rc myrc --replicas=5
```

---

### Which field identifies Pods managed by RC?

```yaml
selector:
```

---

### Which field defines Pod configuration?

```yaml
template:
```

---

### What is the modern replacement for ReplicationController?

```text
ReplicaSet (managed through Deployment)
```

---

# Summary

```text
ReplicationController
        |
        +--> Maintains Desired Number of Pods
        |
        +--> Provides Self-Healing
        |
        +--> Supports Scaling
        |
        +--> Improves High Availability
```

### Workflow

```text
Create RC
     |
     v
RC Creates Pods
     |
     v
Create Service
     |
     v
AWS Load Balancer Created
     |
     v
Application Accessible
     |
     v
Scale Pods Up/Down
     |
     v
RC Maintains Desired State
```
