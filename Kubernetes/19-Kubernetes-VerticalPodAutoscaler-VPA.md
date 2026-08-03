# 19-Kubernetes-VerticalPodAutoscaler-VPA

# Vertical Pod Autoscaler (VPA)

## What is Autoscaling?

In production, the amount of traffic on an application changes continuously.

For example:

- Morning → Low traffic
- Afternoon → Medium traffic
- Sale/Festival → Very High traffic

If we always allocate the maximum CPU and Memory, resources are wasted.

If we allocate very little resources, the application becomes slow or crashes.

To solve this problem, Kubernetes provides **Autoscaling**.

Autoscaling automatically adjusts application resources according to workload.

---

# Types of Autoscaling in Kubernetes

There are mainly three types of autoscaling.

| Autoscaler | Scales | Changes |
|------------|---------|---------|
| Horizontal Pod Autoscaler (HPA) | Horizontally | Number of Pods |
| Vertical Pod Autoscaler (VPA) | Vertically | CPU & Memory of Pod |
| Cluster Autoscaler (CA) | Infrastructure | Number of Worker Nodes |

---

# Understanding Horizontal vs Vertical Scaling

Before learning VPA, let's quickly revise Horizontal Scaling.

## Horizontal Scaling

Horizontal scaling means:

> **Increase or decrease the number of Pods.**

Example:

Initially

```
Traffic
   │
   ▼
+---------+
| Pod-1   |
+---------+
```

Traffic increases.

```
Traffic
   │
   ▼
+---------+
| Pod-1   |
+---------+

+---------+
| Pod-2   |
+---------+

+---------+
| Pod-3   |
+---------+
```

Instead of making one Pod stronger, Kubernetes creates more Pods.

This is called **Horizontal Scaling**.

---

## Vertical Scaling

Vertical Scaling means:

> **Increase or decrease CPU and Memory of the existing Pod.**

Example:

Initially

```
Pod

CPU     : 100m
Memory  : 128Mi
```

After VPA

```
Pod

CPU     : 400m
Memory  : 512Mi
```

The number of Pods remains the same.

Only the Pod becomes more powerful.

---

# Real World Example

Imagine a restaurant.

### Horizontal Scaling

Customer count increases.

Owner hires more waiters.

```
2 Waiters

↓

6 Waiters
```

More workers.

Same strength.

---

### Vertical Scaling

Instead of hiring more waiters,

Owner gives one waiter:

- Faster billing system
- Better tools
- Bigger kitchen

The waiter becomes more capable.

This is Vertical Scaling.

---

# Why do we need VPA?

Sometimes traffic is not the problem.

Instead,

each Pod continuously consumes more CPU or Memory.

Example:

```
Database

CPU Usage

100m

↓

250m

↓

400m

↓

500m
```

Even though only one Pod is enough,

its CPU requirement keeps increasing.

Creating more Pods won't solve the issue.

Instead,

the Pod itself needs more CPU and Memory.

This is where VPA is useful.

---

# Example Scenario

Suppose we have:

```
Deployment

Replicas = 2

Each Pod

CPU = 100m
Memory = 128Mi
```

Initially

```
CPU Usage

40m
```

Everything works fine.

After a few weeks,

more users start using the application.

Now each Pod consumes

```
CPU

95m
```

Soon,

```
CPU

100m
```

Pods become CPU throttled.

Instead of creating more Pods,

VPA increases CPU.

```
100m

↓

300m
```

Application becomes healthy again.

---

# Horizontal Pod Autoscaler (HPA) Revision

HPA increases or decreases Pods according to CPU/Memory utilization.

Example

Initially

```
2 Pods
```

Traffic increases.

```
2 Pods

↓

4 Pods

↓

6 Pods
```

Traffic decreases.

```
6 Pods

↓

3 Pods

↓

2 Pods
```

HPA changes

- Number of Pods

It does **NOT** change CPU or Memory.

---

# Vertical Pod Autoscaler (VPA)

VPA changes

- CPU Request
- Memory Request

and optionally

- CPU Limit
- Memory Limit

Example

Before

```
Pod

CPU Request     : 100m
Memory Request  : 128Mi
```

After Recommendation

```
Pod

CPU Request     : 300m
Memory Request  : 512Mi
```

The Pod receives more resources.

---

# HPA vs VPA

| Feature | HPA | VPA |
|----------|-----|-----|
| Changes Number of Pods | ✅ Yes | ❌ No |
| Changes CPU | ❌ No | ✅ Yes |
| Changes Memory | ❌ No | ✅ Yes |
| Good for Traffic Spikes | ✅ Yes | ❌ No |
| Good for Long-Term Resource Growth | ❌ No | ✅ Yes |
| Pod Restart Required | ❌ Usually No | ✅ Yes (Auto mode) |

---

# Which one should we use?

## Use HPA when

- Traffic suddenly increases
- Website receives many users
- API receives more requests
- More Pods are required

Example

```
Black Friday Sale

100 Users

↓

10,000 Users
```

HPA creates more Pods.

---

## Use VPA when

Traffic is almost constant,

but every Pod continuously needs more CPU or Memory.

Example

- Java Application
- Machine Learning Model
- Database
- Elasticsearch
- Kafka

These applications usually become heavier over time.

Instead of creating more Pods,

we increase Pod resources.

---

# Can HPA and VPA work together?

Yes.

Modern production environments often use both.

Example

```
Daily Workload

↓

VPA increases CPU and Memory

↓

Festival Sale

↓

HPA creates more Pods
```

VPA handles **Pod Size**.

HPA handles **Pod Count**.

Both solve different problems.

---

# Real Production Example

Suppose we have an E-Commerce Application.

```
Users

↓

Frontend

↓

Backend

↓

Database
```

Normal Day

```
Backend

2 Pods

CPU = 200m
```

After several months,

Backend starts processing larger orders.

Each Pod now needs

```
CPU

500m
```

VPA increases CPU.

Later,

Diwali Sale starts.

Traffic becomes 10x.

Now,

HPA creates more Backend Pods.

So,

```
VPA

↓

Makes each Pod stronger

AND

HPA

↓

Creates more Pods
```

This combination is very common in production Kubernetes clusters.

---

# Key Points

- Autoscaling automatically adjusts resources.
- HPA changes the number of Pods.
- VPA changes CPU and Memory.
- Horizontal Scaling = More Pods.
- Vertical Scaling = Bigger Pod.
- VPA is useful when one Pod continuously needs more resources.
- HPA is useful when more users arrive.
- Both HPA and VPA solve different problems.
- In production, HPA and VPA are often used together.

---

## What's Next?

In **Part 2**, we will learn:

- VPA Architecture
- VPA Components
- Recommender
- Updater
- Admission Controller
- Complete Internal Workflow
- How VPA actually decides new CPU & Memory values
- YAML Deep Dive

# VPA Architecture

Before writing any YAML, we should first understand **how VPA works internally**.

Most beginners think:

> "VPA directly increases CPU and Memory."

This is **not true**.

VPA follows a complete workflow before changing any resources.

---

# VPA Internal Architecture

```
                  +----------------------+
                  |     Application      |
                  +----------+-----------+
                             |
                             |
                      Resource Usage
                             |
                             ▼
                  +----------------------+
                  |    Metrics Server     |
                  +----------+-----------+
                             |
                             |
                   CPU & Memory Metrics
                             |
                             ▼
                  +----------------------+
                  |   VPA Recommender    |
                  +----------+-----------+
                             |
                  Recommendation
                             |
            +----------------+----------------+
            |                                 |
            ▼                                 ▼
+----------------------+          +---------------------------+
|    VPA Updater       |          | VPA Admission Controller |
+----------+-----------+          +-------------+-------------+
           |                                      |
      Evicts Pod                          Modifies Pod Spec
           |                                      |
           +----------------+---------------------+
                            |
                            ▼
                    New Pod Created
                            |
                            ▼
             Pod starts with new CPU & Memory
```

---

# Components of VPA

VPA consists of **three main components**.

1. Recommender
2. Updater
3. Admission Controller

Each component has a different responsibility.

---

# 1. VPA Recommender

The Recommender is the **brain of VPA**.

Its job is to continuously monitor resource usage and calculate recommendations.

It does **NOT** change the Pod.

It only recommends.

---

## What does Recommender monitor?

It continuously collects:

- CPU Usage
- Memory Usage

from the Metrics Server.

Example

```
Pod

CPU Usage

80m
85m
92m
96m
101m
```

Recommender notices that the Pod is consistently using around **100m CPU**.

It may recommend:

```
Target CPU

200m
```

---

## Think Like This

Imagine a doctor.

The doctor checks:

- Blood Pressure
- Temperature
- Heart Rate

The doctor only gives advice.

The doctor doesn't perform surgery.

Similarly,

Recommender only gives recommendations.

---

# Recommender Workflow

```
Pod

↓

Metrics Server

↓

CPU & Memory Metrics

↓

VPA Recommender

↓

Recommendation Generated
```

---

# Recommendation Example

Current Pod

```
CPU Request

100m
```

Observed Usage

```
180m
```

Recommendation

```
Target CPU

250m
```

No changes happen yet.

Only a recommendation is created.

---

# 2. VPA Updater

Updater is responsible for applying the recommendation.

It continuously checks:

```
Has Recommender suggested
different resources?
```

If **No**

Do nothing.

If **Yes**

Evict the Pod.

---

## What is Eviction?

Eviction means

**Gracefully deleting a Pod.**

It is **NOT** a force delete.

```
Old Pod

↓

Evicted

↓

Deployment creates New Pod
```

---

# Why Evict the Pod?

A running Pod's CPU and Memory cannot usually be changed.

Therefore,

VPA removes the old Pod.

Deployment automatically creates a new Pod.

The new Pod receives updated resources.

---

# Updater Workflow

```
Recommendation

↓

Updater

↓

Evict Pod

↓

Deployment

↓

New Pod
```

---

# Important Note

Updater **never** creates Pods.

Deployment creates Pods.

Updater only removes them.

---

# 3. Admission Controller

This component works **only when a Pod is being created**.

It intercepts the Pod creation request.

Before Kubernetes starts the Pod,

Admission Controller injects the new resource values.

---

Example

Deployment YAML

```yaml
requests:
  cpu: 100m
```

VPA Recommendation

```
250m
```

Admission Controller modifies the Pod.

Final Pod

```yaml
requests:
  cpu: 250m
```

---

# Why is Admission Controller Needed?

Without it,

the new Pod would again start with

```
100m CPU
```

Admission Controller ensures that

the newly created Pod starts with the recommended resources.

---

# Complete VPA Workflow

Let's understand the entire process.

---

## Step 1

Application is running.

```
Deployment

↓

Pods
```

---

## Step 2

Metrics Server collects usage.

```
CPU

Memory
```

---

## Step 3

Recommender analyzes metrics.

```
Average CPU

Average Memory

Peak Usage
```

---

## Step 4

Recommendation is generated.

Example

```
CPU

100m

↓

250m
```

---

## Step 5

Updater notices that recommendation.

```
Recommendation Available

↓

Evict Pod
```

---

## Step 6

Deployment creates a new Pod.

```
ReplicaSet

↓

New Pod
```

---

## Step 7

Admission Controller injects resources.

```
New Pod

CPU

250m

Memory

512Mi
```

---

## Step 8

Application continues with updated resources.

---

# Complete Flow Diagram

```
Application

↓

Metrics Server

↓

Recommender

↓

Recommendation

↓

Updater

↓

Old Pod Deleted

↓

Deployment

↓

New Pod

↓

Admission Controller

↓

Updated CPU & Memory
```

---

# Why Metrics Server is Required

VPA cannot guess resource usage.

It needs real usage data.

Without Metrics Server,

```
kubectl top pods
```

will fail.

Recommender will have no metrics.

No recommendation will be generated.

---

# How to Verify VPA Components

After installation,

run:

```bash
kubectl get pods -n kube-system | grep vpa
```

Expected

```
vpa-admission-controller

vpa-recommender

vpa-updater
```

Each Pod has a specific job.

| Component | Responsibility |
|-----------|----------------|
| Recommender | Calculates CPU & Memory recommendations |
| Updater | Evicts Pods when needed |
| Admission Controller | Applies new resources to newly created Pods |

---

# Understanding VPA Update Modes

VPA supports different update modes.

---

## 1. Off

```
updateMode: "Off"
```

VPA only provides recommendations.

It never changes Pods.

Good for testing.

---

## 2. Initial

```
updateMode: "Initial"
```

Recommendations are applied **only when a Pod is created for the first time**.

Running Pods are never updated.

---

## 3. Auto

```
updateMode: "Auto"
```

This is the mode we used in our practical.

Workflow

```
Recommendation

↓

Updater

↓

Pod Evicted

↓

New Pod

↓

Admission Controller

↓

Updated Resources
```

Everything happens automatically.

---

## 4. Recreate (Newer Recommendation)

In newer Kubernetes/VPA versions,

`Recreate` is recommended instead of `Auto`.

Behavior is similar.

Pods are recreated with updated resources.

---

# Key Points

- VPA has three components.
- Recommender is the brain.
- Updater evicts Pods.
- Admission Controller injects new resources.
- Metrics Server provides CPU & Memory metrics.
- VPA does not directly edit a running Pod.
- A new Pod is created with updated resources.
- `Auto` mode fully automates the process.
- `Off` mode only gives recommendations.
- `Initial` updates resources only during initial Pod creation.

---

## What's Next?

In **Part 3**, we'll cover the complete hands-on practical:

- Install Metrics Server
- Install VPA
- Verify Components
- Create Deployment
- Create VPA
- Generate Load
- Observe Recommendations
- Understand VPA YAML line by line
- Explain every command and expected output

# Practical Lab - Vertical Pod Autoscaler (VPA)

In this practical, we will build a complete VPA environment from scratch.

We will learn:

- Install Metrics Server
- Install VPA Components
- Create Deployment
- Create Service
- Create VPA
- Generate Load
- Observe Recommendations
- Observe Pod Recreation
- Verify Updated Resources

This is very similar to how VPA is used in production environments.

---

# Lab Architecture

```

```
                    +----------------------+
                    |   Load Generator     |
                    +----------+-----------+
                               |
                               ▼
                    +----------------------+
                    |   ClusterIP Service  |
                    +----------+-----------+
                               |
                               ▼
                    +----------------------+
                    | NGINX Deployment     |
                    |     (2 Pods)         |
                    +----------+-----------+
                               |
                    CPU & Memory Usage
                               |
                               ▼
                    +----------------------+
                    |   Metrics Server     |
                    +----------+-----------+
                               |
                               ▼
                    +----------------------+
                    |   VPA Recommender    |
                    +----------+-----------+
                               |
                        Recommendation
                               |
                               ▼
                    +----------------------+
                    |    VPA Updater       |
                    +----------+-----------+
                               |
                        Evicts Old Pod
                               |
                               ▼
                    +----------------------+
                    | Deployment Creates   |
                    |      New Pod         |
                    +----------+-----------+
                               |
                               ▼
                    +----------------------+
                    | Admission Controller |
                    +----------+-----------+
                               |
                               ▼
                     Updated CPU & Memory
```

---

# Step 1 - Verify Cluster

Before starting, ensure the cluster is healthy.

```bash
kubectl get nodes

kubectl get pods -n kube-system
```

Expected:

- All Nodes should be **Ready**
- All kube-system Pods should be **Running**

---

# Step 2 - Install Metrics Server

VPA requires Metrics Server to collect CPU and Memory usage.

Install Metrics Server.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Edit Metrics Server

```bash
kubectl edit deployment metrics-server -n kube-system
```

Inside `args:` add:

```yaml
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```

These flags help Metrics Server communicate with kubelets in our lab environment.

---

## Verify Metrics Server

```bash
kubectl rollout status deployment metrics-server -n kube-system
```

Now verify:

```bash
kubectl top nodes

kubectl top pods -A
```

If both commands work, Metrics Server is installed correctly.

---

# Step 3 - Install VPA

Clone the Kubernetes Autoscaler repository.

```bash
git clone https://github.com/kubernetes/autoscaler.git
```

Go to VPA directory.

```bash
cd autoscaler/vertical-pod-autoscaler
```

Install VPA.

```bash
./hack/vpa-up.sh
```

---

## Verify Installation

```bash
kubectl get pods -n kube-system | grep vpa
```

Expected Output

```
vpa-admission-controller

vpa-recommender

vpa-updater
```

All Pods should be in **Running** state.

---

# Step 4 - Create Working Directory

```bash
mkdir ~/vpa

cd ~/vpa
```

---

# Step 5 - Create Deployment

Create `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
      - name: nginx
        image: nginx:latest

        ports:
        - containerPort: 80

        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"

          limits:
            cpu: "500m"
            memory: "512Mi"
```

Apply it.

```bash
kubectl apply -f deployment.yaml
```

Verify.

```bash
kubectl get pods
```

Expected

```
2 Pods Running
```

---

# Understanding Resource Requests

Initially every Pod requests:

```
CPU

100m
```

Memory

```
128Mi
```

These values are only the starting point.

Later,

VPA may recommend larger or smaller values.

---

# Step 6 - Create Service

Instead of sending traffic directly to Pods,

we expose the Deployment using a ClusterIP Service.

```bash
kubectl expose deployment nginx-deployment \
--name=nginx-service \
--port=80 \
--target-port=80
```

Verify.

```bash
kubectl get svc
```

Expected

```
nginx-service
```

---

# Why do we create a Service?

In production,

applications never communicate directly with Pod IPs.

Instead,

they communicate through Services.

```
Application

↓

ClusterIP Service

↓

Pods
```

This allows Pods to be recreated without changing application configuration.

---

# Step 7 - Create VPA

Create `vpa.yaml`

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler

metadata:
  name: nginx-vpa

spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment

  updatePolicy:
    updateMode: Auto
```

Apply it.

```bash
kubectl apply -f vpa.yaml
```

Verify.

```bash
kubectl get vpa
```

Expected

```
NAME

nginx-vpa
```

---

# Step 8 - Understand VPA YAML

## targetRef

```yaml
targetRef:
```

Defines which workload VPA will monitor.

Here,

```
Deployment

↓

nginx-deployment
```

---

## updateMode

```yaml
updateMode: Auto
```

Means:

- Generate recommendation
- Evict old Pod
- Create new Pod
- Apply new resources

Everything happens automatically.

---

# Step 9 - Generate Continuous Load

Create a BusyBox Pod.

```bash
kubectl run load-generator \
--image=busybox \
--restart=Never \
-- sh -c "while true; do wget -q -O- http://nginx-service; done"
```

This Pod continuously sends requests.

```
BusyBox

↓

HTTP Request

↓

Service

↓

Deployment

↓

Pods
```

---

# Step 10 - Watch Resource Usage

Open another terminal.

Run

```bash
watch kubectl top pods
```

You should observe CPU usage changing over time.

---

# Step 11 - Observe VPA Recommendation

Run

```bash
watch kubectl describe vpa nginx-vpa
```

Inside

```
Recommendation
```

you may see

```
Lower Bound

Target

Upper Bound
```

Example

```
CPU

Target

250m
```

Memory

```
Target

256Mi
```

These values are calculated by the Recommender.

---

# Step 12 - Watch Pods

Open another terminal.

```bash
watch kubectl get pods
```

When VPA decides new resources are needed,

you may observe

```
Running

↓

Terminating

↓

ContainerCreating

↓

Running
```

This means

Updater evicted the Pod,

Deployment recreated it,

Admission Controller injected new resources.

---

# Step 13 - Verify Updated Resources

Describe the newly created Pod.

```bash
kubectl describe pod <pod-name>
```

Look for

```
Requests

Limits
```

Compare them with the original Deployment.

If values changed,

VPA successfully updated Pod resources.

---

# Step 14 - Stop Load Generator

Delete BusyBox.

```bash
kubectl delete pod load-generator
```

Traffic stops.

---

# Important Observation from Our Lab

During our practice,

Pods remained in

```
Pending
```

Reason

```
Insufficient Memory
```

This happened because

our worker node (`t3.small`)

did not have enough allocatable memory after system components, Metrics Server, and VPA components reserved resources.

The fix was to use a larger worker node (for example, `c7i-flex.large`).

This is a real production-style troubleshooting scenario and highlights that Kubernetes schedules Pods based on **resource requests**, not actual memory usage.

---

# Practical Summary

✔ Installed Metrics Server

✔ Installed VPA

✔ Verified VPA Components

✔ Created Deployment

✔ Created Service

✔ Created VPA

✔ Generated Continuous Load

✔ Observed Recommendations

✔ Understood Pod Eviction

✔ Verified Updated Resources

✔ Learned why Pods can remain Pending due to insufficient requested memory

---

## What's Next?

In **Part 4 (Final Part)**, we'll cover:

- Production Best Practices
- Advantages & Limitations of VPA
- Common Errors & Troubleshooting
- Interview Questions
- Cheat Sheet
- Complete Topic Summary

# Production Best Practices

VPA is a powerful autoscaling solution, but it should be configured carefully in production.

Blindly enabling VPA on every workload can lead to unnecessary Pod restarts or unstable applications.

---

# When Should We Use VPA?

VPA is best suited for workloads whose resource requirements change gradually over time.

Examples:

- Java Applications
- Spring Boot Applications
- Elasticsearch
- Kafka
- Jenkins
- Databases
- Machine Learning Models

These applications usually become heavier over time rather than experiencing sudden traffic spikes.

---

# When Should We Avoid VPA?

VPA is generally **not recommended** for applications with sudden traffic spikes.

Examples:

- E-commerce Websites
- Banking Portals
- Ticket Booking Systems
- Gaming Servers
- Live Streaming Platforms

These applications usually require **more Pods**, not bigger Pods.

For such workloads, HPA is the better choice.

---

# Production Architecture

```
                    Internet
                        │
                        ▼
                 Load Balancer
                        │
                        ▼
                    Ingress
                        │
                        ▼
                 Kubernetes Service
                        │
                        ▼
                 Deployment (HPA)
                ┌───────┴────────┐
                │                │
              Pod-1            Pod-2
                │                │
                └───────┬────────┘
                        │
                 Metrics Server
                        │
                        ▼
                VPA Recommender
                        │
                Recommendation
                        │
                        ▼
                   VPA Updater
                        │
                  Pod Recreation
                        │
                        ▼
          Admission Controller updates
             CPU & Memory Requests
```

---

# Advantages of VPA

### 1. Automatic Resource Optimization

No need to manually update CPU or Memory.

---

### 2. Better Resource Utilization

Applications receive only the resources they actually need.

---

### 3. Reduced Resource Waste

Over-provisioning is minimized.

Example:

Instead of giving every Pod

```
CPU = 1000m
```

VPA may recommend

```
CPU = 250m
```

saving cluster resources.

---

### 4. Prevents CPU Throttling

If CPU usage continuously increases,

VPA recommends higher CPU requests.

---

### 5. Prevents Memory Starvation

If Pods frequently consume more memory,

VPA increases memory requests.

---

### 6. Improves Cluster Efficiency

Unused resources become available for other workloads.

---

# Limitations of VPA

### 1. Pod Restart Required

In Auto/Recreate mode,

Pods must be recreated.

```
Old Pod

↓

Deleted

↓

New Pod
```

---

### 2. Not Suitable for Sudden Traffic

VPA cannot instantly create more Pods.

For sudden traffic spikes,

HPA is a better solution.

---

### 3. Requires Metrics Server

Without Metrics Server,

VPA cannot collect CPU or Memory usage.

---

### 4. Requires Historical Metrics

Immediately after installation,

VPA may not have enough data.

Recommendations improve over time.

---

### 5. May Cause Short Downtime

If only one replica exists,

recreating the Pod can temporarily interrupt the application.

To avoid this,

run multiple replicas.

---

# Best Practices

✅ Always deploy **Metrics Server**.

---

✅ Run at least **2 replicas** for high availability.

---

✅ Test VPA using

```yaml
updateMode: Off
```

before enabling automatic updates.

---

✅ Define sensible

```yaml
minAllowed
```

and

```yaml
maxAllowed
```

limits to prevent extremely small or large recommendations.

Example:

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: nginx
    minAllowed:
      cpu: 100m
      memory: 128Mi
    maxAllowed:
      cpu: 1000m
      memory: 1Gi
```

---

✅ Monitor recommendations before trusting automatic changes.

```bash
kubectl describe vpa nginx-vpa
```

---

# Common Commands

Check VPA

```bash
kubectl get vpa
```

---

Describe VPA

```bash
kubectl describe vpa nginx-vpa
```

---

View Recommendations

```bash
kubectl describe vpa nginx-vpa
```

Look under:

```
Recommendation
```

---

View Updated Resources

```bash
kubectl describe pod <pod-name>
```

---

Check Metrics

```bash
kubectl top nodes

kubectl top pods
```

---

Verify Metrics Server

```bash
kubectl get pods -n kube-system | grep metrics
```

---

Verify VPA Components

```bash
kubectl get pods -n kube-system | grep vpa
```

---

# Common Errors & Troubleshooting

## Error 1

```
kubectl top pods

Error:
Metrics API not available
```

### Cause

Metrics Server is not installed or not working.

### Solution

Install or fix Metrics Server.

---

## Error 2

```
No recommendation available
```

### Cause

VPA has not collected enough metrics yet.

### Solution

Generate load and wait a few minutes.

---

## Error 3

```
Pods remain Pending
```

### Cause

Worker node does not have enough allocatable resources.

### Solution

- Increase worker node size.
- Add more worker nodes.
- Reduce Pod requests.

---

## Error 4

```
Recommendation exists

But Pod resources never change
```

### Cause

VPA is running in

```
updateMode: Off
```

### Solution

Use

```yaml
updateMode: Auto
```

or

```yaml
updateMode: Recreate
```

---

## Error 5

```
Pods keep restarting
```

### Cause

VPA is frequently changing recommendations.

### Solution

Review workload patterns and configure appropriate resource policies.

---

# Interview Questions

## Basic

**1. What is VPA?**

**2. Why do we need VPA?**

**3. Difference between HPA and VPA?**

**4. What does VPA scale?**

**5. Does VPA increase replicas?**

---

## Intermediate

**6. What are the three VPA components?**

**7. What is the role of Recommender?**

**8. What is the role of Updater?**

**9. What is the role of Admission Controller?**

**10. Why is Metrics Server required?**

**11. Why does VPA recreate Pods?**

**12. What is Pod Eviction?**

**13. What is updateMode?**

**14. Difference between Off, Initial and Auto modes?**

**15. How does VPA get CPU usage?**

---

## Advanced

**16. Can HPA and VPA work together?**

**17. Why shouldn't VPA be used for sudden traffic spikes?**

**18. Why did our Pods remain Pending during the lab?**

**19. Does Kubernetes Scheduler check actual memory usage or resource requests?**

**20. How can you limit VPA recommendations?**

**21. What is resourcePolicy?**

**22. What are minAllowed and maxAllowed?**

**23. Why should production deployments have multiple replicas when using VPA?**

**24. What happens if Metrics Server fails?**

**25. When would you choose VPA over HPA?**

---

# VPA Cheat Sheet

| Task | Command |
|------|---------|
| Install VPA | `./hack/vpa-up.sh` |
| Check VPA | `kubectl get vpa` |
| Describe VPA | `kubectl describe vpa <name>` |
| View Metrics | `kubectl top pods` |
| Check VPA Pods | `kubectl get pods -n kube-system \| grep vpa` |
| Check Metrics Server | `kubectl get pods -n kube-system \| grep metrics` |
| View Pod Resources | `kubectl describe pod <pod-name>` |

---

# Complete VPA Workflow (Revision)

```
Application Running
        │
        ▼
Metrics Server Collects Usage
        │
        ▼
VPA Recommender Calculates Recommendation
        │
        ▼
VPA Updater Detects Recommendation
        │
        ▼
Old Pod Evicted
        │
        ▼
Deployment Creates New Pod
        │
        ▼
Admission Controller Injects New CPU & Memory
        │
        ▼
Application Continues with Updated Resources
```

---

# Topic Summary

In this chapter, we learned:

- What Autoscaling is
- Difference between HPA and VPA
- Horizontal vs Vertical Scaling
- VPA Architecture
- Recommender, Updater and Admission Controller
- Metrics Server integration
- VPA YAML configuration
- Update Modes (Off, Initial, Auto)
- Complete hands-on lab
- Resource recommendations
- Pod eviction process
- Production best practices
- Troubleshooting common issues
- Interview questions
- Real-world production use cases

---

# Key Takeaways

- **HPA** scales the **number of Pods**.
- **VPA** scales the **CPU and Memory of Pods**.
- VPA relies on **Metrics Server** for usage data.
- VPA consists of **Recommender**, **Updater**, and **Admission Controller**.
- In Auto/Recreate mode, Pods are recreated with updated resources.
- Kubernetes schedules Pods based on **resource requests**, not actual usage.
- VPA is ideal for workloads with gradually changing resource needs.
- HPA is ideal for sudden traffic spikes.
- In many production environments, **HPA and VPA are used together** because they solve different scaling problems.
