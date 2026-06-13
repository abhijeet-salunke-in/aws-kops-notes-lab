# Kubernetes Namespace

---

# Overview

As Kubernetes clusters grow, multiple teams, applications, and environments start sharing the same cluster.

Example:

```text
Development Team

Testing Team

QA Team

Production Team
```

All teams may deploy:

```text
Pods

Services

Deployments

ReplicaSets

ConfigMaps

Secrets
```

inside the same Kubernetes cluster.

Without proper separation:

```text
Everything becomes mixed together
```

which makes management difficult.

To solve this problem Kubernetes provides:

```text
Namespace
```

---

# Real World Problem

Suppose a company has:

```text
Development Environment

Testing Environment

QA Environment

Production Environment
```

Without Namespace:

```text
Cluster
│
├── Pod1
├── Pod2
├── Pod3
├── Service1
├── Service2
├── Deployment1
└── Deployment2
```

After a few months:

```text
Hundreds of Pods

Hundreds of Services

Hundreds of Deployments
```

Everything becomes difficult to identify and manage.

---

# What is a Namespace?

A Namespace is a logical partition inside a Kubernetes cluster.

It divides one Kubernetes cluster into multiple virtual environments.

Think of Namespace like folders in Linux:

```text
/home/dev

/home/test

/home/prod
```

Each folder contains its own files.

Similarly:

```text
Cluster
│
├── dev
│
├── test
│
├── qa
│
└── prod
```

Each namespace contains its own resources.

---

# Namespace Architecture

```text
Kubernetes Cluster
│
├── Namespace: dev
│      ├── Pods
│      ├── Services
│      └── Deployments
│
├── Namespace: test
│      ├── Pods
│      ├── Services
│      └── Deployments
│
├── Namespace: qa
│
└── Namespace: prod
```

---

# Why Do We Need Namespace?

---

## Resource Isolation

Development resources should not interfere with Production resources.

Example:

```text
dev namespace

pod1
service1
deployment1
```

and

```text
prod namespace

pod1
service1
deployment1
```

can exist simultaneously.

---

## Environment Separation

Different environments can use different namespaces.

```text
dev

test

qa

prod
```

---

## Team Separation

Different teams can have their own namespace.

Example:

```text
frontend-team

backend-team

devops-team
```

---

## Better Resource Management

Instead of managing:

```text
1000 Pods
```

in one place,

you can manage:

```text
100 Pods in dev

200 Pods in test

700 Pods in prod
```

separately.

---

# Default Namespaces in Kubernetes

Check available namespaces:

```bash
kubectl get ns
```

Example:

```text
NAME

default

kube-system

kube-public

kube-node-lease
```

---

# default Namespace

If no namespace is specified:

```bash
kubectl run nginx --image=nginx
```

Kubernetes automatically creates resources inside:

```text
default
```

namespace.

---

# kube-system Namespace

Contains Kubernetes internal components.

Examples:

```text
CoreDNS

etcd

kube-proxy

kube-controller-manager

kube-scheduler
```

---

# kube-public Namespace

Contains publicly readable resources.

---

# kube-node-lease Namespace

Stores heartbeat information for nodes.

---

# Namespace vs Cluster

Cluster:

```text
Physical Kubernetes Environment
```

Namespace:

```text
Logical Separation Inside Cluster
```

Example:

```text
1 Cluster

4 Namespaces

dev
test
qa
prod
```

---

# Practical Lab 1

# Create Namespace Using Imperative Method

This is exactly what was performed in class.

---

## Step 1: View Existing Namespaces

```bash
kubectl get ns
```

---

## Step 2: Create Development Namespace

```bash
kubectl create ns dev
```

Expected:

```text
namespace/dev created
```

---

## Step 3: Create Testing Namespace

```bash
kubectl create ns test
```

Expected:

```text
namespace/test created
```

---

## Step 4: Create QA Namespace

```bash
kubectl create ns qa
```

Expected:

```text
namespace/qa created
```

---

## Step 5: Verify Namespaces

```bash
kubectl get ns
```

Example:

```text
NAME

default

dev

test

qa
```

---

# Creating Pods Inside Namespaces

This was demonstrated in class.

---

## Create Pod in dev Namespace

```bash
kubectl run pod1 --image=nginx -n dev
```

---

## Create Pod in test Namespace

```bash
kubectl run pod2 --image=nginx -n test
```

---

## Create Another Pod in test Namespace

```bash
kubectl run pod3 --image=nginx -n test
```

---

# Verify Pods

Check default namespace:

```bash
kubectl get pods
```

Output:

```text
No resources found
```

Reason:

```text
Pods were created inside dev and test namespaces
```

---

Check dev namespace:

```bash
kubectl get pods -n dev
```

Expected:

```text
pod1
```

---

Check test namespace:

```bash
kubectl get pods -n test
```

Expected:

```text
pod2

pod3
```

---

# What Did We Learn?

Namespace creates isolation.

Example:

```text
dev namespace

pod1
```

and

```text
test namespace

pod2
pod3
```

exist independently.

---

# Practical Lab 2

# Create Namespace Using Declarative Method

Instead of commands, we use YAML files.

---

# Step 1: Create Namespace YAML

## File: ns1.yml

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: devops
```

---

# Understanding Namespace YAML

## apiVersion

```yaml
apiVersion: v1
```

Namespace belongs to Core API Group.

---

## kind

```yaml
kind: Namespace
```

Creates Namespace resource.

---

## metadata

Stores object information.

---

## name

```yaml
name: devops
```

Namespace name.

---

# Step 2: Create Namespace

```bash
kubectl create -f ns1.yml
```

Expected:

```text
namespace/devops created
```

---

# Step 3: Verify Namespace

```bash
kubectl get ns
```

Expected:

```text
devops
```

should appear.

---

# Create Pod Inside Namespace Using YAML

This is exactly what was performed in class.

---

## File: ns-pod.yml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: pod4
  namespace: devops

spec:
  containers:
  - name: nginx
    image: nginx

    ports:
    - containerPort: 80
```

---

# Understanding Pod YAML

## namespace

```yaml
namespace: devops
```

This instructs Kubernetes:

```text
Create Pod Inside devops Namespace
```

---

# Step 4: Create Pod

```bash
kubectl create -f ns-pod.yml
```

Expected:

```text
pod/pod4 created
```

---

# Step 5: Verify Pod

Check default namespace:

```bash
kubectl get pods
```

Output:

```text
No resources found
```

---

Check devops namespace:

```bash
kubectl get pods -n devops
```

Expected:

```text
pod4 Running
```

---

# Namespace Isolation Example

Suppose:

```text
Namespace: dev

pod1
```

and

```text
Namespace: test

pod1
```

Both are valid because namespaces isolate resources.

Without Namespace:

```text
pod1

pod1
```

Conflict occurs.

---

# Important Commands

## View Namespaces

```bash
kubectl get ns
```

---

## Create Namespace

```bash
kubectl create ns dev
```

---

## Delete Namespace

```bash
kubectl delete ns dev
```

---

## View Pods in Namespace

```bash
kubectl get pods -n dev
```

---

## View All Resources in Namespace

```bash
kubectl get all -n dev
```

---

## Describe Namespace

```bash
kubectl describe ns devops
```

---

## View Pods Across All Namespaces

```bash
kubectl get pods -A
```

OR

```bash
kubectl get pods --all-namespaces
```

---

# Cleanup

Delete Pod

```bash
kubectl delete -f ns-pod.yml
```

---

Delete Namespace

```bash
kubectl delete -f ns1.yml
```

OR

```bash
kubectl delete ns devops
```

---

# Interview Questions

### What is a Namespace?

A logical partition inside a Kubernetes cluster used to organize and isolate resources.

---

### Why are Namespaces used?

```text
Resource Isolation

Environment Separation

Team Separation

Better Resource Management
```

---

### What is the default namespace?

```text
default
```

---

### How do you create a namespace?

```bash
kubectl create ns dev
```

---

### How do you create a resource inside a namespace?

```yaml
metadata:
  namespace: devops
```

OR

```bash
kubectl run pod1 --image=nginx -n devops
```

---

### Can the same Pod name exist in multiple namespaces?

```text
Yes
```

Example:

```text
dev/pod1

test/pod1
```

---

### How do you view resources from a specific namespace?

```bash
kubectl get pods -n devops
```

---

### Which namespaces are created by default in Kubernetes?

```text
default

kube-system

kube-public

kube-node-lease
```

---

# Summary

```text
Namespace
      |
      +--> Resource Isolation
      |
      +--> Environment Separation
      |
      +--> Team Separation
      |
      +--> Better Resource Management
      |
      +--> Same Resource Names Allowed
      |
      +--> Logical Partitioning
```

## Evolution of Kubernetes Objects

```text
Pod
│
├─ Problem:
│   No Self Healing
│   No Scaling
│
▼

ReplicationController
│
├─ Self Healing
├─ Scaling
│
▼

ReplicaSet
│
├─ Better Selectors
├─ Self Healing
├─ Scaling
│
▼

Deployment
│
├─ Rolling Updates
├─ Rollbacks
├─ Version History
│
▼

DaemonSet
│
├─ One Pod Per Node
│
▼

Namespace
│
├─ Resource Isolation
├─ Environment Separation
├─ Team Separation
│
▼

Production Ready Kubernetes Clusters
```
