# Kubernetes Horizontal Pod Autoscaler (HPA)

# What is HPA?

The **Horizontal Pod Autoscaler (HPA)** is a Kubernetes resource that automatically **increases or decreases the number of Pods** based on resource usage such as CPU or Memory.

Instead of manually scaling an application, Kubernetes continuously monitors resource utilization and automatically updates the number of replicas.

> **HPA performs Horizontal Scaling (Pods), not Vertical Scaling (CPU/Memory).**

---

# Why Do We Need HPA?

Suppose we deploy an application with:

```yaml
replicas: 2
```

```
Deployment
     │
     ▼
Pod 1
Pod 2
```

During normal traffic, two Pods are enough.

Now imagine thousands of users suddenly access the application.

```
Users
   │
   ▼
Deployment
   │
   ▼
Pod 1 🔥
Pod 2 🔥
```

Both Pods become overloaded.

Without HPA, Kubernetes **will never increase Pods automatically**.

Someone must manually execute:

```bash
kubectl scale deployment medical-deploy --replicas=6
```

Later, when traffic decreases, someone again has to reduce the replicas.

This manual process is slow and can cause downtime or unnecessary cloud costs.

HPA solves this problem by automatically scaling Pods according to the application's workload.

---

# Real-Life Example

Imagine a supermarket.

### Normal Day

```
Customers

↓

Counter 1
Counter 2
```

Two billing counters are enough.

---

### Festival Season

Suddenly hundreds of customers arrive.

```
Customers

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

Counter 1
Counter 2
```

Long queues start forming.

The manager immediately opens more counters.

```
Customers

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

Counter 1
Counter 2
Counter 3
Counter 4
Counter 5
Counter 6
```

When customers leave,

the manager closes the extra counters.

Exactly the same happens in Kubernetes.

| Supermarket | Kubernetes |
|-------------|------------|
| Customers | User Requests |
| Counter | Pod |
| Manager | HPA |
| Open Counter | Create Pod |
| Close Counter | Delete Pod |

---

# How HPA Works

```
Users
   │
   ▼
Pods
   │
   ▼
CPU Usage
   │
   ▼
Kubelet
   │
   ▼
Metrics Server
   │
   ▼
HPA
   │
   ▼
Deployment
   │
   ▼
ReplicaSet
   │
   ▼
New Pods
```

---

# Components Used

| Component | Purpose |
|-----------|---------|
| Pod | Runs Application |
| Kubelet | Collects Pod Metrics |
| Metrics Server | Stores CPU & Memory Metrics |
| HPA | Makes Scaling Decisions |
| Deployment | Updates Replica Count |
| ReplicaSet | Creates/Deletes Pods |

---

# Practical Lab (Class Practical)

## Architecture

```
Deployment

↓

Service

↓

Metrics Server

↓

HPA

↓

CPU Stress Test

↓

Automatic Scaling
```

---

# Step 1 - Create Deployment

Create a file.

**deployment.yml**

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: medical-deploy

spec:
  replicas: 2

  selector:
    matchLabels:
      app: medical

  template:
    metadata:
      labels:
        app: medical

    spec:
      containers:
      - name: medical-container
        image: aarushtechnologies/medical

        ports:
        - containerPort: 80

        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"

          limits:
            cpu: "200m"
            memory: "256Mi"
```

Apply it.

```bash
kubectl apply -f deployment.yml
```

Verify.

```bash
kubectl get deployment
```

Expected Output

```
NAME              READY

medical-deploy    2/2
```

---

## Why did we create a Deployment?

Because HPA **cannot scale Pods directly**.

It only changes the replica count of a Deployment.

```
HPA

↓

Deployment

↓

ReplicaSet

↓

Pods
```

---

## New Component - resources

```yaml
resources:
```

Defines how much CPU and Memory a container needs.

---

### requests

```yaml
requests:
```

Minimum resources Kubernetes reserves for this Pod.

In our case,

```yaml
cpu: 100m
```

means

```
0.1 CPU Core
```

---

### Why is CPU Request Required?

HPA calculates

```
(Current CPU Usage)

/

(CPU Request)

×

100
```

Example

Current CPU

```
50m
```

Requested CPU

```
100m
```

Calculation

```
50 / 100

=

50%
```

This percentage is what HPA actually uses.

Without CPU Requests,

HPA cannot calculate CPU utilization.

---

### limits

```yaml
limits:
```

Maximum CPU or Memory the container is allowed to use.

Example

```yaml
cpu: 200m
```

means

The container can use up to

```
0.2 CPU Core
```

but not more.

---

# Step 2 - Create Service

Create

**service.yml**

```yaml
apiVersion: v1
kind: Service

metadata:
  name: medical-service

spec:
  selector:
    app: medical

  ports:
  - port: 80
    targetPort: 80

  type: LoadBalancer
```

Apply

```bash
kubectl apply -f service.yml
```

Verify

```bash
kubectl get svc
```

---

## Why are we creating a Service?

Pods are temporary.

Whenever Pods are recreated,

their IP addresses change.

```
Pod

↓

Deleted

↓

New Pod

↓

New IP
```

Users should not communicate directly with Pods.

Instead,

they communicate with the Service.

```
Users

↓

Service

↓

Pods
```

The Service automatically forwards traffic to all available Pods.

When HPA creates more Pods,

the Service automatically starts sending traffic to them.

No manual changes are required.

---

# Step 3 - Install Metrics Server

Install

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify

```bash
kubectl get deployment metrics-server -n kube-system
```

Expected

```
metrics-server

1/1
```

---

## Why do we need Metrics Server?

Pods do not directly send CPU usage to HPA.

Instead,

```
Pods

↓

Kubelet

↓

Metrics Server

↓

HPA
```

Without Metrics Server,

```
HPA

↓

No Metrics

↓

No Scaling
```

---

## Verify Metrics

```bash
kubectl top nodes
```

Example

```
NAME

ip-172-31-xx

CPU

4%
```

Verify Pods

```bash
kubectl top pods
```

Example

```
medical-deploy

1m
```

If these commands work,

Metrics Server is installed correctly.

---

# Step 4 - Create HPA

Create

**hpa.yml**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: web-hpa

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: medical-deploy

  minReplicas: 2
  maxReplicas: 10

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply

```bash
kubectl apply -f hpa.yml
```

Verify

```bash
kubectl get hpa
```

Expected

```
NAME

web-hpa

TARGETS

1%/50%
```

---

# Why are we creating HPA?

Deployment always keeps a fixed number of Pods.

```
Deployment

↓

2 Pods
```

HPA continuously checks CPU usage.

If CPU crosses the configured target,

it changes

```
Deployment

replicas

2

↓

4
```

Deployment then creates new Pods automatically.

---

# New Components

### scaleTargetRef

```yaml
scaleTargetRef:
```

Specifies **which Deployment should be scaled**.

---

### minReplicas

```yaml
minReplicas: 2
```

HPA will never go below

```
2 Pods
```

---

### maxReplicas

```yaml
maxReplicas: 10
```

Even if CPU reaches 100%,

HPA will never create more than

```
10 Pods
```

---

### averageUtilization

```yaml
averageUtilization: 50
```

Target CPU Utilization

```
CPU < 50%

↓

No Scaling
```

```
CPU > 50%

↓

Scale Up
```

# Practical 2 - Live CPU Stress Test (Our Practice)

In the previous practical, we created the HPA.

Now let's verify whether it actually scales Pods.

Instead of generating user traffic, we created CPU load inside one Pod.

---

## Step 1 - Open Four Terminal Sessions

Since we were using the AWS EC2 browser terminal (which doesn't support multiple panes), we opened **four browser tabs**.

### Terminal 1

Monitor HPA

```bash
watch kubectl get hpa
```

---

### Terminal 2

Monitor Deployment

```bash
watch kubectl get deployment
```

---

### Terminal 3

Monitor Pod CPU Usage

```bash
watch kubectl top pod
```

---

### Terminal 4

Generate CPU Load

---

## Why did we open four terminals?

Because HPA works continuously.

We wanted to observe every component at the same time.

```
Terminal 1

↓

HPA Decision

Terminal 2

↓

Deployment Scaling

Terminal 3

↓

CPU Usage

Terminal 4

↓

Generate Load
```

This makes it very easy to understand what Kubernetes is doing internally.

---

# Step 2 - Enter the Running Pod

List Pods.

```bash
kubectl get pods
```

Example

```bash
kubectl exec -it medical-deploy-xxxxxxxxxx-abcde -- sh
```

---

## Why are we entering the Pod?

Because we want to generate CPU usage **inside the application container**.

This allows HPA to detect increased CPU utilization.

---

# Step 3 - Verify the "yes" Command

Run

```bash
which yes
```

Output

```
/usr/bin/yes
```

---

## Why are we checking this?

The `yes` command continuously prints the letter `y`.

Normally,

```
yes

↓

yyyyyyyyyyyyyyyyyyyy
```

This consumes CPU continuously.

Instead of printing on the screen, we discard the output.

---

# Step 4 - Generate CPU Load

Run

```bash
yes > /dev/null
```

---

## Why `/dev/null`?

`/dev/null` is a special Linux file.

Anything written to it is discarded.

```
yes

↓

Infinite Output

↓

/dev/null

↓

Discarded
```

The CPU keeps working,

but the terminal remains clean.

---

# Step 5 - Observe CPU Usage

Terminal 3

```bash
watch kubectl top pod
```

Initially

```
CPU

1m
```

After running

```bash
yes > /dev/null
```

CPU became

```
200m
```

---

## Why did CPU become 200m?

Our Deployment had

```yaml
limits:
  cpu: "200m"
```

The container kept consuming CPU until it reached its configured limit.

---

# Step 6 - Observe HPA

Terminal 1

```bash
watch kubectl get hpa
```

Initially

```
1% / 50%
```

After a few seconds

```
51% / 50%
```

---

## Why 51%?

Our Deployment requested

```yaml
requests:
  cpu: "100m"
```

HPA calculates

```
CPU Utilization

=

(Current CPU Usage)

/

(Request CPU)

×

100
```

As CPU usage increased, utilization crossed the configured target.

```
Target

50%

↓

Current

51%

↓

Scale Up
```

---

# Step 7 - Observe Deployment

Terminal 2

```bash
watch kubectl get deployment
```

Initially

```
READY

2/2
```

Later

```
READY

4/4
```

---

## What happened internally?

```
CPU Increased

↓

Kubelet

↓

Metrics Server

↓

HPA

↓

Deployment

↓

ReplicaSet

↓

New Pods
```

Notice something important.

HPA never creates Pods.

It only updates

```
Deployment

replicas

2

↓

4
```

ReplicaSet creates the new Pods.

---

# Step 8 - Stop CPU Load

Inside the Pod

Press

```
Ctrl + C
```

---

## What happened?

CPU immediately dropped.

Terminal 3

```
200m

↓

1m
```

HPA detected

```
51%

↓

1%
```

---

# Step 9 - Observe Scale Down

Deployment remained

```
4 Pods
```

for a few minutes.

Then,

```
4 Pods

↓

2 Pods
```

---

## Why didn't Kubernetes remove Pods immediately?

Imagine traffic changes like this.

```
50%

↓

52%

↓

48%

↓

53%

↓

49%
```

If Kubernetes immediately created and deleted Pods,

the cluster would continuously scale.

```
Create

↓

Delete

↓

Create

↓

Delete
```

This is called

**Thrashing (Flapping).**

To avoid this,

Kubernetes waits before scaling down.

---

# Our Practical Observations

| CPU | HPA | Deployment |
|------|------|------------|
| 1m | 1% / 50% | 2 Pods |
| 200m | 51% / 50% | 4 Pods |
| 1m | 1% / 50% | Wait... |
| 1m | 1% / 50% | 2 Pods |

This practical demonstrated both automatic **Scale-Up** and **Scale-Down**.

---

# Extra Practice - Production Load Testing

Instead of generating CPU load inside a Pod,

production teams usually generate actual HTTP requests.

Common tools

- ApacheBench (ab)
- hey
- wrk
- k6
- JMeter
- Locust

---

## Install ApacheBench

```bash
sudo yum install httpd-tools -y
```

Verify

```bash
ab -V
```

---

## Generate Load

```bash
ab -n 50000 -c 100 http://<LOADBALANCER-DNS>/
```

Meaning

```
-n

↓

Total Requests

50000
```

```
-c

↓

Concurrent Users

100
```

Observe

```bash
watch kubectl get hpa
```

```bash
watch kubectl get deployment
```

```bash
watch kubectl top pod
```

If CPU crosses the configured threshold,

HPA automatically scales the Deployment.

---

# Common Errors

## Metrics API Not Available

```
kubectl top pod
```

Output

```
Metrics API not available
```

### Reason

Metrics Server is not installed.

### Solution

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## HPA Shows

```
<unknown>/50%
```

### Reason

Metrics are unavailable.

### Check

```bash
kubectl top pod
```

---

## HPA Doesn't Scale

Possible reasons

- CPU Requests not defined
- CPU usage below target
- Wrong Deployment name
- Metrics Server not running

---

# Useful Commands

```bash
kubectl get hpa
```

```bash
kubectl describe hpa web-hpa
```

```bash
kubectl delete hpa web-hpa
```

```bash
kubectl top pod
```

```bash
kubectl top nodes
```

```bash
watch kubectl get deployment
```

```bash
watch kubectl get hpa
```

---

# Interview Questions

### 1. What is HPA?

Automatically scales Pods according to resource utilization.

---

### 2. What resources can HPA monitor?

- CPU
- Memory
- Custom Metrics
- External Metrics

---

### 3. Which component provides metrics?

Metrics Server.

---

### 4. Why are CPU Requests mandatory?

Because HPA calculates

```
(Current CPU)

/

(Request CPU)

×

100
```

Without Requests,

CPU utilization cannot be calculated.

---

### 5. Does HPA create Pods?

No.

```
HPA

↓

Deployment

↓

ReplicaSet

↓

Pods
```

---

### 6. Difference between HPA and Deployment?

Deployment maintains a fixed replica count.

HPA automatically changes the Deployment replica count.

---

### 7. Difference between Horizontal and Vertical Scaling?

Horizontal

```
Increase Pods
```

Vertical

```
Increase CPU & Memory
```

---

### 8. Why doesn't HPA immediately scale down?

To avoid Thrashing (Flapping).

---

### 9. Can HPA scale StatefulSets?

Yes.

---

### 10. What happens if Metrics Server stops?

HPA cannot collect metrics and therefore cannot make scaling decisions.

---

# Summary

```
HPA

↓

Automatic Pod Scaling

Needs

↓

Metrics Server

Uses

↓

CPU Requests

Scales

↓

Deployment

Deployment Updates

↓

ReplicaSet

ReplicaSet Creates

↓

Pods
```

---

## Complete Practical Flow

```
Create Deployment
        │
        ▼
Create Service
        │
        ▼
Install Metrics Server
        │
        ▼
Verify Metrics
        │
        ▼
Create HPA
        │
        ▼
Verify HPA
        │
        ▼
Open 4 Terminals
        │
        ▼
Generate CPU Load
        │
        ▼
CPU Increases
        │
        ▼
Metrics Server Collects Metrics
        │
        ▼
HPA Updates Deployment
        │
        ▼
ReplicaSet Creates New Pods
        │
        ▼
Stop CPU Load
        │
        ▼
CPU Drops
        │
        ▼
HPA Waits
        │
        ▼
Deployment Scales Down
```

---

# Key Takeaways

- HPA performs **Horizontal Scaling**.
- HPA depends on **Metrics Server**.
- CPU Requests are mandatory for CPU-based HPA.
- HPA scales the **Deployment**, not the Pods.
- ReplicaSet creates and removes Pods.
- Scale-Up is quick.
- Scale-Down is intentionally delayed.
- Our practical demonstrated the complete HPA lifecycle from **2 Pods → 4 Pods → 2 Pods** automatically.
