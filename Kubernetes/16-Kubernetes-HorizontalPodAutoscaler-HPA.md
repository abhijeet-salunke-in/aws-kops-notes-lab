# Kubernetes Horizontal Pod Autoscaler (HPA)

## Introduction

As applications become more popular, the number of users accessing them can change throughout the day. During peak hours, thousands or even millions of users may send requests simultaneously, while during off-peak hours only a few users may be using the application.

If an application always runs with a fixed number of Pods, two major problems can occur:

- **Too few Pods** → The application becomes slow or unavailable because existing Pods cannot handle the traffic.
- **Too many Pods** → Unnecessary Pods continue consuming CPU, memory, and cloud resources, increasing infrastructure costs.

To solve this problem, Kubernetes provides an automatic scaling mechanism called the **Horizontal Pod Autoscaler (HPA)**.

Instead of manually increasing or decreasing the number of Pods, HPA continuously monitors resource usage (such as CPU or Memory) and automatically adjusts the number of Pod replicas based on the workload.

This allows applications to remain highly available while also optimizing infrastructure costs.

---

# What is Horizontal Pod Autoscaler (HPA)?

The **Horizontal Pod Autoscaler (HPA)** is a Kubernetes resource that automatically increases or decreases the number of Pod replicas in a Deployment, ReplicaSet, or StatefulSet based on resource utilization or custom metrics.

Unlike manual scaling, HPA continuously monitors application performance and automatically adjusts the replica count whenever predefined thresholds are crossed.

For example:

- During high traffic, HPA creates additional Pods.
- During low traffic, HPA removes unnecessary Pods.
- The application remains highly available without manual intervention.

---

# Why Do We Need HPA?

Before understanding HPA, let's first understand the limitation of a normal Deployment.

Suppose we create a Deployment with:

```yaml
spec:
  replicas: 2
```

This tells Kubernetes to always maintain **2 running Pods**.

Now imagine the application suddenly receives thousands of user requests.

```
Users
   │
   ▼
Deployment
   │
   ▼
2 Pods
```

Even though the Pods are overloaded, Kubernetes **will not automatically create additional Pods** because the Deployment only ensures that the desired replica count is maintained.

The only way to increase Pods without HPA is by manually executing:

```bash
kubectl scale deployment medical-deploy --replicas=6
```

After traffic decreases, someone must again manually reduce the replica count.

```
Traffic ↑
      │
      ▼
Administrator scales manually
      │
      ▼
Traffic ↓
      │
      ▼
Administrator scales down manually
```

This manual approach has several disadvantages:

- Requires continuous monitoring.
- Human intervention is required.
- Slow response during sudden traffic spikes.
- Higher chances of downtime.
- Unnecessary infrastructure costs.

This is where HPA becomes extremely useful.

Instead of relying on an administrator, Kubernetes automatically performs scaling whenever resource utilization crosses the configured threshold.

---

# Real-Life Example

Imagine a supermarket with only **2 billing counters**.

### Normal Day

```
Customers

      ↓

Counter 1
Counter 2
```

Since only a few customers are shopping, two billing counters are sufficient.

---

### Festival Season

Suddenly hundreds of customers enter the supermarket.

```
Customers

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

Counter 1  (Busy)
Counter 2  (Busy)
```

Long queues begin to form.

The supermarket manager immediately opens more billing counters.

```
Customers

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

Counter 1
Counter 2
Counter 3
Counter 4
Counter 5
Counter 6
```

More billing counters reduce waiting time and improve customer experience.

---

### After the Rush Ends

Late at night, customer traffic decreases.

Keeping all six billing counters open would waste employee time and increase operational costs.

The manager closes the extra counters and returns to only two active counters.

```
Customers

↓

Counter 1
Counter 2
```

This is exactly how HPA works.

| Supermarket | Kubernetes |
|-------------|------------|
| Customers | User Requests |
| Billing Counter | Pod |
| Store Manager | Horizontal Pod Autoscaler |
| Open More Counters | Create More Pods |
| Close Extra Counters | Remove Extra Pods |

---

# Why is it Called "Horizontal" Pod Autoscaler?

There are two ways to improve application performance.

## 1. Vertical Scaling

Increase the resources of the existing machine or Pod.

Example:

```
Old Pod

CPU : 100m
Memory : 128Mi

        │

        ▼

New Pod

CPU : 500m
Memory : 512Mi
```

The Pod count remains the same.

Only CPU and Memory are increased.

---

### Advantages

- Easy to implement.
- No additional Pods are created.
- Good for small workloads.

### Disadvantages

- Limited by machine capacity.
- Eventually the node cannot provide additional CPU or Memory.
- Usually requires restarting the application.

---

## 2. Horizontal Scaling

Instead of increasing CPU or Memory, Kubernetes creates additional Pods.

```
Before

Deployment

↓

Pod 1
Pod 2
```

```
After

Deployment

↓

Pod 1
Pod 2
Pod 3
Pod 4
Pod 5
Pod 6
```

Each Pod shares the incoming traffic.

---

### Advantages

- High availability.
- Better fault tolerance.
- Easier to handle sudden traffic spikes.
- Suitable for cloud-native applications.
- No need to upgrade machine hardware.

---

### Disadvantages

- Applications should preferably be stateless.
- Requires proper load balancing.
- More networking components are involved.

---

# Vertical Scaling vs Horizontal Scaling

| Feature | Vertical Scaling | Horizontal Scaling |
|----------|------------------|--------------------|
| What changes? | CPU & Memory | Number of Pods |
| Hardware upgrade required? | Yes | No |
| Downtime possible? | Usually Yes | Usually No |
| Maximum limit | Limited by hardware | Limited by cluster capacity |
| High Availability | No | Yes |
| Used by HPA? | ❌ No | ✅ Yes |

---

# Key Takeaways

- A Deployment maintains a fixed number of replicas.
- A Deployment **cannot automatically scale** based on CPU or Memory usage.
- Manual scaling is slow and requires human intervention.
- Horizontal Pod Autoscaler (HPA) automatically adjusts the number of Pods based on resource utilization.
- HPA improves application availability while reducing infrastructure costs.
- HPA performs **Horizontal Scaling**, not Vertical Scaling.
- Horizontal Scaling is the preferred approach for modern cloud-native applications.

# Kubernetes Components Involved in HPA

Before learning how HPA works, let's understand the Kubernetes components involved in the autoscaling process.

Several Kubernetes components work together whenever HPA scales an application.

```
                    User Requests
                          │
                          ▼
                     Application Pods
                          │
                          ▼
                       Kubelet
                          │
                          ▼
                    Metrics Server
                          │
                          ▼
              Horizontal Pod Autoscaler
                          │
                          ▼
                      Deployment
                          │
                          ▼
                      ReplicaSet
                          │
                          ▼
                    Create/Delete Pods
```

Each component has its own responsibility.

---

## 1. Pods

Pods run the actual application containers.

Example:

```
Medical Application

↓

Pod 1
Pod 2
```

Users send requests to these Pods.

As more users access the application:

- CPU increases
- Memory increases

The Pods themselves do not know whether they should scale.

They simply execute the application.

---

## 2. Kubelet

Every Kubernetes node runs a component called **Kubelet**.

Kubelet continuously monitors the Pods running on its node.

It collects information such as:

- CPU usage
- Memory usage
- Pod status
- Container status

Think of Kubelet as a **health inspector** for all Pods running on a node.

---

## 3. Metrics Server

The Metrics Server is one of the most important components for HPA.

Without Metrics Server, HPA cannot determine CPU or Memory usage.

Its job is simple.

```
Pods

↓

Kubelet

↓

Metrics Server
```

Metrics Server collects resource usage from every Kubelet and stores it temporarily.

Whenever HPA needs CPU metrics, it asks Metrics Server.

---

## Why is Metrics Server Required?

Suppose HPA wants to know:

```
Current CPU Usage = ?
```

Pods don't directly send this information.

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

Without Metrics Server:

```
HPA

↓

No CPU Metrics

↓

Cannot Decide Scaling
```

This is why the following command is required before creating HPA.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## How to Verify Metrics Server?

Check whether it is running.

```bash
kubectl get deployment metrics-server -n kube-system
```

Example:

```
NAME             READY

metrics-server   1/1
```

Now verify metrics.

```bash
kubectl top nodes
```

Example:

```
NAME      CPU    MEMORY

Node-1    4%     40%
```

Similarly,

```bash
kubectl top pods
```

Example:

```
NAME

medical-pod

CPU

1m

MEMORY

6Mi
```

If these commands work successfully, Metrics Server is functioning correctly.

---

# Why CPU Requests are Mandatory for HPA

This is one of the most commonly misunderstood concepts.

Suppose your Deployment contains:

```yaml
resources:
  requests:
    cpu: "100m"
```

HPA does **not** directly compare CPU usage.

Instead, it calculates **CPU Utilization**.

Formula:

```
CPU Utilization

=

Current CPU Usage

/

Requested CPU

× 100
```

Example:

Current CPU:

```
50m
```

Requested CPU:

```
100m
```

Calculation:

```
50m

/

100m

=

0.5

=

50%
```

Therefore HPA sees:

```
Current Utilization = 50%
```

Now suppose your HPA target is:

```yaml
averageUtilization: 50
```

Since:

```
Current = 50%

Target = 50%
```

HPA keeps the current number of Pods.

---

## Our Practical Example

Our Deployment requested:

```yaml
requests:
  cpu: "100m"
```

Initially:

```
kubectl top pod

↓

1m
```

Calculation:

```
1m

/

100m

=

1%
```

That is exactly why HPA displayed:

```
cpu: 1% / 50%
```

Later we generated CPU load.

```
yes > /dev/null
```

CPU became:

```
200m
```

The container was allowed to use CPU up to its configured limit.

HPA immediately observed that the utilization exceeded the configured target and decided to increase the number of replicas.

---

# Important Difference

Many beginners think HPA uses:

```
Current CPU

↓

200m
```

This is incorrect.

HPA uses:

```
CPU Utilization

=

(Current CPU)

/

(Requested CPU)
```

Therefore CPU **requests** are mandatory for CPU-based HPA.

Without CPU requests, HPA cannot correctly calculate CPU utilization.

---

# Internal Working of HPA

Let's understand what actually happens internally.

### Step 1

Users start accessing the application.

```
Users

↓

Pods
```

---

### Step 2

CPU usage increases.

```
Pod CPU

↓

70%
```

---

### Step 3

Kubelet measures resource usage.

```
Pods

↓

Kubelet
```

---

### Step 4

Metrics Server collects the metrics.

```
Pods

↓

Kubelet

↓

Metrics Server
```

---

### Step 5

HPA periodically requests metrics.

```
Metrics Server

↓

HPA
```

Example:

```
Current CPU = 70%

Target CPU = 50%
```

---

### Step 6

HPA decides.

```
Current > Target

↓

Need More Pods
```

---

### Step 7

HPA updates the Deployment.

```
Deployment

replicas:

2

↓

4
```

Notice something important.

HPA **does not create Pods directly**.

It only updates the Deployment.

---

### Step 8

Deployment updates ReplicaSet.

```
Deployment

↓

ReplicaSet
```

---

### Step 9

ReplicaSet creates additional Pods.

```
ReplicaSet

↓

Pod 3

Pod 4
```

Now traffic can be distributed across more Pods.

---

# Complete Internal Workflow

```
Users
   │
   ▼
Application Pods
   │
   ▼
CPU Usage Increases
   │
   ▼
Kubelet Measures CPU
   │
   ▼
Metrics Server Collects Metrics
   │
   ▼
Horizontal Pod Autoscaler
   │
   ▼
Updates Deployment Replica Count
   │
   ▼
Deployment Updates ReplicaSet
   │
   ▼
ReplicaSet Creates New Pods
```

---

# Scale-Up Process

```
Current CPU = 70%

Target CPU = 50%

↓

HPA detects overload

↓

Deployment replicas

2

↓

4

↓

ReplicaSet

↓

Creates 2 New Pods

↓

Traffic gets distributed
```

---

# Scale-Down Process

Suppose traffic suddenly decreases.

```
CPU

↓

10%
```

HPA notices that utilization is much lower than the target.

However, Kubernetes **does not immediately remove Pods**.

Instead, it waits for a stabilization period.

Why?

To avoid continuously creating and deleting Pods if traffic fluctuates for a short time.

After the stabilization period:

```
Deployment

4 Pods

↓

3 Pods

↓

2 Pods
```

HPA never scales below:

```yaml
minReplicas: 2
```

because we configured:

```yaml
minReplicas: 2
```

---

# Key Takeaways

- HPA depends on Metrics Server.
- Metrics Server collects CPU and Memory metrics from Kubelets.
- CPU utilization is calculated using **CPU Requests**, not CPU Limits.
- HPA compares **Current Utilization** with **Target Utilization**.
- HPA **never creates Pods directly**.
- HPA updates the Deployment.
- Deployment updates the ReplicaSet.
- ReplicaSet creates or removes Pods.
- Scale-up happens relatively quickly.
- Scale-down is intentionally delayed to avoid unnecessary Pod creation and deletion.

# HPA YAML Explained

Let's understand the HPA configuration that we used during the practical.

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

---

## apiVersion

```yaml
apiVersion: autoscaling/v2
```

Specifies the API version of the HPA resource.

The `autoscaling/v2` API supports:

- CPU Metrics
- Memory Metrics
- Custom Metrics
- External Metrics

It is the recommended version for modern Kubernetes clusters.

---

## kind

```yaml
kind: HorizontalPodAutoscaler
```

Creates an HPA resource.

---

## metadata

```yaml
metadata:
  name: web-hpa
```

The name of the HPA object.

---

## scaleTargetRef

```yaml
scaleTargetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: medical-deploy
```

This tells HPA **which workload should be scaled**.

In our case:

```
HPA

↓

Deployment

↓

medical-deploy
```

Notice something important.

HPA does **NOT** point directly to Pods.

Instead, it points to the Deployment.

---

## minReplicas

```yaml
minReplicas: 2
```

The minimum number of Pods HPA is allowed to maintain.

Even if CPU usage becomes 0%,

HPA will never reduce Pods below:

```
2 Pods
```

---

## maxReplicas

```yaml
maxReplicas: 10
```

The maximum number of Pods HPA can create.

Even if CPU reaches 100%,

HPA will never create more than:

```
10 Pods
```

---

## metrics

```yaml
metrics:
```

Specifies which metrics HPA should monitor.

Examples include:

- CPU
- Memory
- Custom Metrics
- External Metrics

---

## type

```yaml
type: Resource
```

The metric comes from Kubernetes resources.

Examples:

- CPU
- Memory

---

## resource

```yaml
resource:
```

Specifies which resource should be monitored.

---

## CPU

```yaml
name: cpu
```

HPA continuously checks CPU utilization.

---

## target

```yaml
target:
```

Defines the desired target value.

---

## Utilization

```yaml
type: Utilization
```

HPA compares CPU utilization as a percentage of the requested CPU.

Formula:

```
(Current CPU Usage / CPU Request) × 100
```

---

## averageUtilization

```yaml
averageUtilization: 50
```

Target CPU utilization is 50%.

Meaning:

```
Current CPU < 50%

↓

No Scaling
```

```
Current CPU > 50%

↓

Scale Up
```

---

# Practical 1 – Trainer's Lab

## Step 1 – Create Deployment

```bash
kubectl apply -f medical-deploy.yml
```

Verify:

```bash
kubectl get deployment
```

---

## Step 2 – Create Service

```bash
kubectl apply -f medical-service.yml
```

Verify:

```bash
kubectl get svc
```

---

## Step 3 – Verify Metrics Server

```bash
kubectl top nodes
```

If you receive:

```
Metrics API not available
```

Install Metrics Server.

---

## Step 4 – Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify:

```bash
kubectl get deployment metrics-server -n kube-system
```

---

## Step 5 – Verify Metrics

```bash
kubectl top nodes

kubectl top pods
```

Example:

```
CPU

1m
```

---

## Step 6 – Create HPA

```bash
kubectl apply -f web-hpa.yml
```

Verify:

```bash
kubectl get hpa
```

Example:

```
NAME

web-hpa

TARGETS

1% / 50%
```

---

# Practical 2 – CPU Stress Test (Hands-on)

This practical demonstrates how HPA automatically scales Pods.

Unlike the previous practical, here we intentionally increase CPU usage to observe HPA in action.

---

## Step 1 – Open Multiple Terminal Sessions

Open four SSH sessions.

```
Terminal 1

↓

watch kubectl get hpa
```

```
Terminal 2

↓

watch kubectl get deployment
```

```
Terminal 3

↓

watch kubectl top pod
```

```
Terminal 4

↓

Generate CPU Load
```

---

## Step 2 – Enter a Running Pod

```bash
kubectl get pods
```

Example:

```bash
kubectl exec -it medical-deploy-858c545755-7kz46 -- sh
```

---

## Step 3 – Verify the yes Command

```bash
which yes
```

Example:

```
/usr/bin/yes
```

---

## Step 4 – Generate CPU Load

Run:

```bash
yes > /dev/null
```

This command continuously generates output while discarding it.

The CPU remains busy.

---

## Step 5 – Observe CPU Usage

Terminal 3:

```bash
watch kubectl top pod
```

Example:

Before:

```
1m
```

After:

```
200m
```

---

## Step 6 – Observe HPA

Terminal 1:

```bash
watch kubectl get hpa
```

Example:

Before:

```
cpu: 1% / 50%
```

After:

```
cpu: 51% / 50%
```

HPA detects that CPU utilization has exceeded the configured threshold.

---

## Step 7 – Observe Deployment

Terminal 2:

```bash
watch kubectl get deployment
```

Example:

Before:

```
READY

2/2
```

After:

```
READY

4/4
```

HPA updated the Deployment.

Deployment updated the ReplicaSet.

ReplicaSet created two additional Pods.

---

## Step 8 – Stop CPU Stress

Press:

```
Ctrl + C
```

inside the Pod.

CPU usage immediately decreases.

---

## Step 9 – Observe Scale Down

CPU:

```
200m

↓

1m
```

HPA:

```
51%

↓

1%
```

Deployment:

```
4 Pods

↓

(wait)

↓

2 Pods
```

Notice that Kubernetes does **not** immediately remove Pods.

Scale down happens after a stabilization period.

This prevents unnecessary scaling caused by temporary traffic fluctuations.

---

# Practical 3 – Production Style Load Testing

Instead of generating artificial CPU load inside the container, production environments usually simulate real user traffic.

Common tools:

- ApacheBench (ab)
- hey
- wrk
- k6
- JMeter
- Locust

---

## Install ApacheBench

```bash
yum install httpd-tools -y
```

Verify:

```bash
ab -V
```

---

## Generate HTTP Traffic

```bash
ab -n 50000 -c 100 http://<LOADBALANCER-DNS>/
```

Where:

- `-n` → Total requests
- `-c` → Concurrent requests

---

## Observe HPA

```bash
watch kubectl get hpa
```

Observe Deployment:

```bash
watch kubectl get deployment
```

Observe CPU:

```bash
watch kubectl top pod
```

If the application consumes enough CPU while serving requests, HPA automatically creates additional Pods.

---

# Observations from Our Practical

During our lab we observed the complete lifecycle of HPA.

### Initial State

```
Pods

2
```

```
CPU

1m
```

```
HPA

1% / 50%
```

---

### During CPU Stress

```
yes > /dev/null
```

CPU:

```
200m
```

HPA:

```
51% / 50%
```

Deployment:

```
2 Pods

↓

4 Pods
```

---

### After Stopping CPU Stress

```
Ctrl + C
```

CPU:

```
1m
```

HPA:

```
1% / 50%
```

Deployment:

```
4 Pods

↓

(wait)

↓

2 Pods
```

This demonstrated both **automatic scale-up** and **automatic scale-down** using HPA.

# Common Errors & Troubleshooting

While working with HPA, you may encounter some common issues. Understanding these problems will help you troubleshoot quickly in production environments.

---

## 1. Metrics API Not Available

### Error

```bash
kubectl top pods
```

Output:

```text
error: Metrics API not available
```

### Cause

The Metrics Server is not installed or not running.

### Solution

Install the Metrics Server.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify:

```bash
kubectl get deployment metrics-server -n kube-system
```

---

## 2. HPA Always Shows

```
<unknown>/50%
```

### Cause

HPA is unable to collect CPU metrics.

Possible reasons:

- Metrics Server is not working.
- Pod is still starting.
- Metrics have not yet been collected.

### Solution

Verify:

```bash
kubectl top pods
```

---

## 3. HPA Never Scales

Possible reasons:

### CPU is Too Low

Example:

```
Current CPU = 5%

Target CPU = 50%
```

HPA will not scale because the target has not been reached.

---

### CPU Requests Not Defined

Example:

```yaml
resources:
  limits:
    cpu: "200m"
```

Missing:

```yaml
requests:
  cpu: "100m"
```

Without CPU Requests, HPA cannot correctly calculate CPU utilization.

---

### Wrong Deployment Name

Example:

```yaml
scaleTargetRef:
  name: wrong-name
```

HPA cannot find the Deployment.

Verify:

```bash
kubectl get deployment
```

---

## 4. Pods Don't Scale Down Immediately

Many beginners think HPA is not working.

Actually, Kubernetes intentionally delays scale down.

Why?

To avoid:

```
Traffic ↑

↓

Create Pods

↓

Traffic ↓

↓

Delete Pods

↓

Traffic ↑

↓

Create Pods Again
```

This constant creation and deletion is called:

- Thrashing
- Flapping

To avoid this, Kubernetes waits before reducing replicas.

---

## 5. HPA Creates Pods But Application is Still Slow

Possible reasons:

- Database bottleneck
- Slow application code
- External API latency
- Storage bottleneck
- Network bottleneck

HPA only adds Pods.

It cannot solve every performance problem.

---

# Useful Commands

## View HPA

```bash
kubectl get hpa
```

---

## Detailed HPA Information

```bash
kubectl describe hpa web-hpa
```

---

## View YAML

```bash
kubectl get hpa web-hpa -o yaml
```

---

## Delete HPA

```bash
kubectl delete hpa web-hpa
```

---

## View Deployment

```bash
kubectl get deployment
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

## View CPU Usage

```bash
kubectl top pods
```

---

## View Node Metrics

```bash
kubectl top nodes
```

---

## Watch HPA

```bash
watch kubectl get hpa
```

---

## Watch Deployment

```bash
watch kubectl get deployment
```

---

## Watch Pod Metrics

```bash
watch kubectl top pods
```

---

# Best Practices

- Always define CPU Requests when using CPU-based HPA.
- Keep realistic `minReplicas` and `maxReplicas` values.
- Install and verify Metrics Server before creating HPA.
- Monitor application performance after scaling.
- Use HPA for stateless applications whenever possible.
- Test HPA in a staging environment before production.
- Use proper readiness and liveness probes.
- Combine HPA with Cluster Autoscaler for complete automatic scaling.
- Avoid setting extremely low CPU targets, which can cause unnecessary scaling.
- Continuously monitor application metrics using Prometheus and Grafana in production.

---

# Interview Questions and Answers

## 1. What is HPA?

**Answer:**

Horizontal Pod Autoscaler (HPA) is a Kubernetes resource that automatically increases or decreases the number of Pod replicas based on resource utilization or custom metrics.

---

## 2. What is the purpose of HPA?

**Answer:**

To automatically scale applications according to workload, improving availability while reducing infrastructure costs.

---

## 3. What metrics does HPA support?

**Answer:**

- CPU
- Memory
- Custom Metrics
- External Metrics

---

## 4. What component provides metrics to HPA?

**Answer:**

Metrics Server.

---

## 5. Does HPA create Pods directly?

**Answer:**

No.

HPA updates the Deployment.

Deployment updates the ReplicaSet.

ReplicaSet creates or removes Pods.

---

## 6. Why are CPU Requests required?

**Answer:**

HPA calculates CPU utilization using CPU Requests.

Formula:

```
(Current CPU Usage / CPU Request) × 100
```

Without CPU Requests, HPA cannot correctly calculate CPU utilization.

---

## 7. Difference between HPA and Deployment?

**Deployment**

Maintains a fixed number of Pods.

**HPA**

Automatically changes the Deployment replica count based on workload.

---

## 8. Difference between Horizontal and Vertical Scaling?

**Horizontal Scaling**

Adds or removes Pods.

**Vertical Scaling**

Increases CPU or Memory of existing Pods.

---

## 9. Why doesn't HPA immediately remove Pods?

**Answer:**

Kubernetes waits before scaling down to avoid flapping caused by temporary traffic fluctuations.

---

## 10. Can HPA scale StatefulSets?

**Answer:**

Yes.

HPA can scale Deployments, ReplicaSets, and StatefulSets if the workload supports scaling.

---

## 11. What happens if Metrics Server is unavailable?

**Answer:**

HPA cannot obtain CPU or Memory metrics and therefore cannot make scaling decisions.

---

## 12. What happens when CPU exceeds the target?

**Answer:**

HPA increases the replica count by updating the Deployment.

---

## 13. Can HPA scale based on Memory?

**Answer:**

Yes.

HPA supports Memory metrics using the Resource metric type.

---

## 14. What is the difference between minReplicas and maxReplicas?

**minReplicas**

Minimum number of Pods HPA maintains.

**maxReplicas**

Maximum number of Pods HPA is allowed to create.

---

## 15. Why is HPA important in cloud environments?

**Answer:**

Because workloads constantly change.

HPA automatically adjusts the number of Pods according to demand, improving availability while reducing infrastructure costs.

---

# Summary

In this chapter, we learned how Kubernetes automatically scales applications using the Horizontal Pod Autoscaler (HPA).

We started by understanding why a fixed number of Pods is not sufficient for applications with changing traffic patterns. We then explored the difference between horizontal and vertical scaling and learned why HPA performs horizontal scaling by increasing or decreasing the number of Pod replicas.

Next, we studied the Kubernetes components involved in autoscaling, including Pods, Kubelet, Metrics Server, HPA, Deployment, and ReplicaSet. We also learned why CPU Requests are essential for CPU-based autoscaling and how HPA calculates CPU utilization.

Finally, we performed two practical demonstrations:

- A trainer-style practical where we created the Deployment, Service, Metrics Server, and HPA.
- A live CPU stress test using:

```bash
yes > /dev/null
```

During the practical, we observed:

- CPU utilization increasing.
- HPA detecting the increased utilization.
- Deployment automatically scaling from **2 Pods to 4 Pods**.
- ReplicaSet creating new Pods.
- CPU returning to normal after stopping the stress.
- HPA waiting before scaling down to avoid unnecessary Pod creation and deletion.
- Deployment automatically returning from **4 Pods to 2 Pods** after the stabilization period.

This demonstrated the complete lifecycle of HPA, from monitoring resource utilization to automatic scale-up and scale-down.

---

# Quick Revision

- HPA performs **Horizontal Scaling**.
- HPA depends on **Metrics Server**.
- HPA calculates CPU utilization using **CPU Requests**.
- HPA updates the **Deployment**, not the Pods.
- ReplicaSet creates or removes Pods.
- Scale-up is relatively fast.
- Scale-down is intentionally delayed.
- `minReplicas` defines the minimum number of Pods.
- `maxReplicas` defines the maximum number of Pods.
- HPA improves both **application availability** and **cost optimization**.
