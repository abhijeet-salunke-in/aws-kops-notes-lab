# Kubernetes LimitRange

# What is LimitRange?

**LimitRange** is a Kubernetes resource that defines the **minimum, maximum, and default CPU/Memory values** that Pods or Containers are allowed to use within a **Namespace**.

It helps Kubernetes administrators enforce resource usage policies so that developers cannot deploy Pods with invalid or missing resource configurations.

> **LimitRange is a Namespace-level resource. It only affects Pods created inside the namespace where it exists.**

---

# Why Do We Need LimitRange?

In a Kubernetes cluster, multiple developers deploy applications.

Some developers may write:

```yaml
resources:
  requests:
    cpu: "100m"
```

Some may write:

```yaml
resources:
  requests:
    cpu: "2"
```

Some may completely forget to define resources.

```yaml
containers:
- image: nginx
```

Without any rules,

- One Pod may consume too many resources.
- Another Pod may have no limits.
- Cluster resources become difficult to manage.

Instead of checking every YAML manually,

the Kubernetes Administrator creates a **LimitRange**.

Now every Pod inside that namespace follows the same resource policy.

---

# Real-Life Example

Imagine a company issuing laptops to employees.

Without any company policy,

One employee requests

```
64 GB RAM
```

Another requests

```
4 GB RAM
```

Another submits no specification at all.

Managing resources becomes difficult.

So the company creates rules.

```
Minimum RAM

↓

8 GB
```

```
Default RAM

↓

16 GB
```

```
Maximum RAM

↓

32 GB
```

Every employee now follows the company policy.

LimitRange works exactly the same way inside a Kubernetes Namespace.

---

# How LimitRange Works

```
Developer Creates Pod
         │
         ▼
 Kubernetes API Server
         │
         ▼
Check Namespace
         │
         ▼
Is LimitRange Present?
         │
   ┌─────┴─────┐
   │           │
  Yes          No
   │           │
   ▼           ▼
Validate /     Create Pod
Apply Defaults
   │
   ▼
Create Pod
```

---

# What Can LimitRange Do?

A LimitRange can:

- Define **Default Requests**
- Define **Default Limits**
- Define **Minimum CPU/Memory**
- Define **Maximum CPU/Memory**
- Reject Pods that violate resource policies

---

# Practical Lab (Our Class)

## Architecture

```
Create Namespace
        │
        ▼
Create LimitRange
        │
        ▼
Verify LimitRange
        │
        ▼
Create Pod Without Resources
        │
        ▼
Observe Default Values
        │
        ▼
Create Pod With Resources
        │
        ▼
Validate Min & Max Rules
```

---

# Step 1 - Create Namespace

Command

```bash
kubectl create namespace development
```

Verify

```bash
kubectl get ns
```

Expected Output

```
NAME

default
development
kube-system
```

---

## Why are we creating a Namespace?

A **LimitRange** is a **Namespace-scoped** resource.

It cannot exist without a Namespace.

Every Namespace can have its own resource policy.

Example

```
default

↓

No LimitRange
```

```
development

↓

CPU Request = 250m
```

```
production

↓

CPU Request = 500m
```

Different teams can therefore have different resource policies while using the same Kubernetes cluster.

---

# Step 2 - Create LimitRange

Create

**limit-range.yml**

```yaml
apiVersion: v1
kind: LimitRange

metadata:
  name: my-range
  namespace: development

spec:
  limits:
  - type: Container

    default:
      cpu: "500m"
      memory: "512Mi"

    defaultRequest:
      cpu: "250m"
      memory: "256Mi"

    min:
      cpu: "100m"
      memory: "128Mi"

    max:
      cpu: "1"
      memory: "1Gi"
```

Apply

```bash
kubectl apply -f limit-range.yml
```

---

## Why are we creating a LimitRange?

Instead of depending on every developer to correctly configure CPU and Memory,

the Kubernetes Administrator creates a policy.

Every Pod inside this Namespace must follow this policy.

```
Namespace

↓

LimitRange

↓

All Pods
```

---

# New Components

We've already learned:

- apiVersion
- metadata
- namespace

So let's only understand the new fields.

---

## kind

```yaml
kind: LimitRange
```

Creates a **policy object**.

Unlike a Pod or Deployment,

LimitRange does not run containers.

It only stores resource rules.

---

## limits

```yaml
limits:
```

Defines the resource policy.

This block contains:

- Default Limits
- Default Requests
- Minimum Values
- Maximum Values

---

## type

```yaml
type: Container
```

Apply these rules to **every container**.

Example

```
Pod
│
├── Container A
└── Container B
```

Each container individually follows this policy.

---

## default

```yaml
default:
```

Defines the **default resource limits**.

If a developer does not specify limits,

Kubernetes automatically adds them.

Example

```yaml
default:
  cpu: "500m"
  memory: "512Mi"
```

Developer creates

```yaml
containers:
- image: nginx
```

Internally Kubernetes changes it to

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

No manual configuration is required.

---

## Why do we need default?

Without default limits,

developers may accidentally create Pods with unlimited resource usage.

LimitRange prevents this by automatically applying standard limits.

---

## defaultRequest

```yaml
defaultRequest:
```

Defines the **default resource requests**.

If requests are not specified,

Kubernetes automatically adds them.

Example

```yaml
defaultRequest:
  cpu: "250m"
  memory: "256Mi"
```

Pod created

```yaml
containers:
- image: nginx
```

Internally becomes

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
```

---

## Why do we need defaultRequest?

Remember from HPA:

```
Request

↓

Reserved CPU
```

Requests are used by the Kubernetes Scheduler.

Instead of leaving resource reservation undefined,

LimitRange automatically reserves sensible default values.

---

## Difference Between default and defaultRequest

| default | defaultRequest |
|----------|----------------|
| Sets default Limits | Sets default Requests |
| Controls maximum allowed usage | Reserves minimum guaranteed resources |
| Applied automatically | Applied automatically |

---

## Step 3 - Verify LimitRange

Verify

```bash
kubectl get limitrange -n development
```

or

```bash
kubectl get limits -n development
```

Expected Output

```
NAME

my-range
```

---

## Why are we verifying?

To ensure Kubernetes successfully created the LimitRange object.

Just like we verify Deployments and Services,

we should always verify policy objects after creation.

---

# Step 4 - Describe LimitRange

Command

```bash
kubectl describe limitrange my-range -n development
```

Expected Output

```
Type         Resource   Min    Max    Default Request    Default Limit

Container    cpu        100m   1      250m               500m

Container    memory     128Mi  1Gi    256Mi              512Mi
```

---

## Why are we using describe?

`kubectl get`

Only confirms the object exists.

```
my-range
```

`kubectl describe`

Displays the complete resource policy stored inside Kubernetes.

At this point,

the LimitRange is active.

However,

no Pod has been affected yet because no Pod exists inside the namespace.

Current State

```
development Namespace

        │

        ▼

   LimitRange

        │

        ├── Default Request

        ├── Default Limit

        ├── Minimum Values

        └── Maximum Values

        │

        ▼

     Waiting for Pods
```

The actual validation begins only when Pods are created.

# Step 5 - Create Pod Without Resources

Create

**pod1.yml**

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: pod1
  namespace: development

spec:
  containers:
  - name: pod1-cont1
    image: nginx
```

Apply

```bash
kubectl apply -f pod1.yml
```

Verify

```bash
kubectl get pods -n development
```

Expected Output

```
NAME

pod1

Running
```

---

## Why didn't we define resources?

This was done intentionally.

We wanted to check whether **LimitRange automatically adds the default CPU and Memory values**.

Notice there is no

```yaml
resources:
```

block.

Normally this Pod would have no Requests or Limits.

However,

because a LimitRange exists inside the namespace,

Kubernetes automatically injects the default values.

---

## Internal Working

```
Developer Creates Pod

        │

        ▼

API Server Receives Pod

        │

        ▼

LimitRange Found

        │

        ▼

Default Request Added

        │

        ▼

Default Limit Added

        │

        ▼

Pod Stored

        │

        ▼

Pod Running
```

---

# Step 6 - Verify Default Values

Describe the Pod

```bash
kubectl describe pod pod1 -n development
```

Look for

```
Limits

↓

cpu: 500m

memory: 512Mi
```

and

```
Requests

↓

cpu: 250m

memory: 256Mi
```

---

## Observation

Even though our YAML never contained

```yaml
resources:
```

the Pod still has CPU and Memory configured.

This proves that LimitRange automatically added

- Default Requests
- Default Limits

during Pod creation.

---

# Step 7 - Create Pod With Resources

Now create

**pod2.yml**

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: pod2
  namespace: development

spec:
  containers:
  - name: pod2-cont1
    image: nginx

    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"

      limits:
        cpu: "500m"
        memory: "512Mi"
```

Apply

```bash
kubectl apply -f pod2.yml
```

Verify

```bash
kubectl get pods -n development
```

---

## Why are we creating another Pod?

The first Pod demonstrated

```
default

↓

defaultRequest
```

Now we want to verify

```
min

↓

max
```

This time,

the developer is manually providing CPU and Memory values.

Kubernetes must now validate them.

---

# New Components

The only new section is

```yaml
resources:
```

which we already learned during HPA.

No new YAML fields are introduced here.

---

# Understanding min

Our LimitRange contains

```yaml
min:
  cpu: "100m"
  memory: "128Mi"
```

Meaning

Every container must request at least

```
CPU

100m
```

and

```
Memory

128Mi
```

Suppose a developer writes

```yaml
requests:
  cpu: "50m"
```

What happens?

```
50m

↓

Less than

↓

100m

↓

Rejected
```

Kubernetes refuses to create the Pod.

---

# Understanding max

Our LimitRange contains

```yaml
max:
  cpu: "1"
  memory: "1Gi"
```

Meaning

No container can exceed

```
CPU

1 Core
```

or

```
Memory

1Gi
```

Suppose a developer writes

```yaml
limits:
  cpu: "2"
```

Result

```
2 CPU

↓

Greater than

↓

1 CPU

↓

Rejected
```

The Pod will not be created.

---

# Why was pod2 accepted?

Our Pod contains

```yaml
requests:
  cpu: "200m"
```

LimitRange requires

```
Minimum

100m
```

Since

```
200m

>

100m
```

it satisfies the minimum requirement.

---

Our Pod also contains

```yaml
limits:
  cpu: "500m"
```

Maximum allowed

```
1 CPU

=

1000m
```

Since

```
500m

<

1000m
```

the Pod satisfies the maximum requirement.

Therefore Kubernetes allows the Pod.

---

# Step 8 - Verify Pod Resources

Describe

```bash
kubectl describe pod pod2 -n development
```

Observe

```
Requests

↓

200m
```

```
Limits

↓

500m
```

Notice

Kubernetes did **not** overwrite our values.

Because they already satisfy the namespace policy.

---

# Internal Working

```
Developer Creates Pod

        │

        ▼

API Server

        │

        ▼

LimitRange Validation

        │

        ├── Request >= Min ?

        │

        ├── Limit <= Max ?

        │

        └── Valid ?

                │

         Yes ───┘

                │

                ▼

           Create Pod
```

---

# Complete Practical Flow

```
Create Namespace

        │

        ▼

Create LimitRange

        │

        ▼

Verify LimitRange

        │

        ▼

Create pod1

(No Resources)

        │

        ▼

Default Request Applied

Default Limit Applied

        │

        ▼

Describe pod1

        │

        ▼

Create pod2

(Resources Specified)

        │

        ▼

Validate Min & Max

        │

        ▼

Describe pod2
```

---

# Common Errors

## Namespace Not Found

```
Error

↓

namespaces "development" not found
```

Reason

Namespace does not exist.

Solution

```bash
kubectl create namespace development
```

---

## Resource Below Minimum

Example

```yaml
requests:
  cpu: "50m"
```

Minimum

```
100m
```

Result

Pod rejected.

---

## Resource Above Maximum

Example

```yaml
limits:
  cpu: "2"
```

Maximum

```
1 CPU
```

Result

Pod rejected.

---

## Pod Doesn't Receive Default Values

Possible reason

Pod was created in another Namespace.

Remember

LimitRange only works inside its own Namespace.

---

# Useful Commands

```bash
kubectl get limitrange -n development
```

```bash
kubectl describe limitrange my-range -n development
```

```bash
kubectl get pods -n development
```

```bash
kubectl describe pod pod1 -n development
```

```bash
kubectl describe pod pod2 -n development
```

```bash
kubectl delete limitrange my-range -n development
```

---

# Interview Questions

### 1. What is LimitRange?

A Kubernetes resource that defines default, minimum and maximum CPU/Memory values for Pods or Containers inside a Namespace.

---

### 2. Is LimitRange Namespace-scoped?

Yes.

It only affects resources created inside the same Namespace.

---

### 3. What happens if a Pod has no resources section?

If a LimitRange defines

```
default

defaultRequest
```

Kubernetes automatically adds those values during Pod creation.

---

### 4. What is the difference between default and defaultRequest?

**default**

Automatically sets resource Limits.

**defaultRequest**

Automatically sets resource Requests.

---

### 5. What does min do?

Defines the minimum CPU or Memory a container is allowed to request.

Pods below this value are rejected.

---

### 6. What does max do?

Defines the maximum CPU or Memory a container is allowed to request or limit.

Pods exceeding this value are rejected.

---

### 7. Does LimitRange modify existing Pods?

No.

It only affects Pods during creation.

---

### 8. Can different Namespaces have different LimitRanges?

Yes.

Each Namespace can have its own resource policy.

---

### 9. Why is LimitRange useful?

It prevents developers from creating Pods with missing or invalid resource configurations.

---

### 10. Does LimitRange create Pods?

No.

It only validates and applies resource policies.

---

# Summary

```
LimitRange

↓

Namespace Resource Policy

────────────────────────────

default

↓

Automatic Limits

────────────────────────────

defaultRequest

↓

Automatic Requests

────────────────────────────

min

↓

Minimum Allowed Resources

────────────────────────────

max

↓

Maximum Allowed Resources

────────────────────────────

Applied During

↓

Pod Creation

────────────────────────────

Scope

↓

Namespace Only
```

---

# Key Takeaways

- LimitRange is a Namespace-scoped resource.
- It applies policies only within its own Namespace.
- `default` automatically applies resource Limits.
- `defaultRequest` automatically applies resource Requests.
- `min` defines the minimum allowed CPU/Memory.
- `max` defines the maximum allowed CPU/Memory.
- Pods violating `min` or `max` are rejected.
- Existing Pods are not modified.
- LimitRange helps standardize resource usage across teams.
