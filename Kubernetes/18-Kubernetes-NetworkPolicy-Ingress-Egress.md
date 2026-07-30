# 18-Kubernetes-NetworkPolicy-Ingress-Egress.md

# Kubernetes NetworkPolicy (Ingress & Egress)

## Table of Contents

1. Introduction
2. What is a NetworkPolicy?
3. Why Do We Need NetworkPolicies?
4. Kubernetes Default Networking
5. NetworkPolicy vs RBAC
6. How NetworkPolicy Works
7. Understanding Pod Selectors
8. Understanding Ingress
9. Understanding Egress
10. Stateful Nature of NetworkPolicies
11. NetworkPolicy Terminology
12. Key Takeaways

---

# Introduction

By default, Kubernetes follows an **open networking model**.

Every Pod can communicate with every other Pod in the cluster unless restricted.

This behavior is convenient during development but becomes a serious security concern in production environments.

Imagine an application consisting of:

```text
Frontend

↓

Backend

↓

Database
```

Ideally,

- Frontend should communicate only with Backend.
- Backend should communicate only with Database.
- Frontend should never directly access Database.

Without NetworkPolicies, Kubernetes allows this:

```text
Frontend ─────────────► Backend

Frontend ─────────────► Database

Backend ──────────────► Database

Database ─────────────► Backend

Every Pod can communicate with every Pod.
```

This violates the **Principle of Least Privilege**.

NetworkPolicy allows us to control which Pods are allowed to communicate with each other.

---

# What is a NetworkPolicy?

A **NetworkPolicy** is a Kubernetes resource used to control **network traffic between Pods**.

It acts like a firewall for Pods.

Instead of controlling servers, it controls communication between workloads inside the Kubernetes cluster.

Think of it as:

```text
Firewall

↓

Server Traffic
```

NetworkPolicy:

```text
Firewall

↓

Pod Traffic
```

NetworkPolicies define:

- Who can send traffic to a Pod (Ingress)
- Where a Pod can send traffic (Egress)

---

# Why Do We Need NetworkPolicies?

Consider a production application.

```text
                Internet
                    │
                    ▼
             Frontend Pod
                    │
                    ▼
             Backend Pod
                    │
                    ▼
             Database Pod
```

The expected communication is:

```text
Frontend ─────────► Backend

Backend ──────────► Database
```

The following communication should **NOT** happen.

```text
Frontend ────────X──► Database

Database ────────X──► Frontend

Database ────────X──► Backend
```

Without NetworkPolicies, Kubernetes allows everything.

This creates several security problems.

Example:

Suppose an attacker compromises the Frontend Pod.

Without NetworkPolicies:

```text
Attacker

↓

Frontend

↓

Database

↓

Sensitive Data
```

With proper NetworkPolicies:

```text
Attacker

↓

Frontend

↓

Database

❌ Access Denied
```

NetworkPolicies help implement a **Zero Trust** architecture.

Every communication must be explicitly allowed.

---

# Kubernetes Default Networking

Kubernetes networking follows two important rules.

## Rule 1

Every Pod gets its own IP address.

Example

```text
Frontend

100.96.1.171
```

```text
Backend

100.96.1.227
```

```text
Database

100.96.1.222
```

---

## Rule 2

Every Pod can communicate with every other Pod by default.

```text
Frontend ─────────► Backend

Frontend ─────────► Database

Backend ──────────► Database

Database ─────────► Frontend

Database ─────────► Backend
```

There is **no firewall** by default.

---

# NetworkPolicy vs RBAC

Many beginners confuse RBAC with NetworkPolicy.

They solve completely different problems.

| RBAC | NetworkPolicy |
|------|---------------|
| Controls Kubernetes API access | Controls Pod-to-Pod communication |
| Works at API Server level | Works at Network level |
| Used for Users and Service Accounts | Used for Pods |
| Controls permissions | Controls network traffic |

Example:

RBAC controls:

```text
kubectl get pods
```

Can the user execute this command?

YES or NO.

NetworkPolicy controls:

```text
Frontend

↓

Backend
```

Can this network connection happen?

YES or NO.

Think of it like this.

RBAC asks:

```text
Who can talk to Kubernetes?
```

NetworkPolicy asks:

```text
Which Pods can talk to each other?
```

---

# How NetworkPolicy Works

A NetworkPolicy never applies to the entire cluster automatically.

Instead,

it selects Pods using labels.

Example:

```yaml
podSelector:
  matchLabels:
    app: backend
```

This means

"Apply this policy only to Pods having"

```text
app=backend
```

Everything else remains unaffected.

This is one of the most important concepts in NetworkPolicy.

---

# NetworkPolicy Components

A NetworkPolicy mainly consists of four parts.

## 1. podSelector

Determines which Pods are protected.

Example

```yaml
podSelector:
  matchLabels:
    app: backend
```

Meaning:

Protect only Backend Pods.

---

## 2. policyTypes

Defines what traffic should be controlled.

Possible values

```yaml
Ingress
```

or

```yaml
Egress
```

or both

```yaml
Ingress
Egress
```

---

## 3. Ingress Rules

Controls

Incoming traffic.

Example

```text
Frontend

↓

Backend
```

Traffic is entering Backend.

This is Ingress.

---

## 4. Egress Rules

Controls

Outgoing traffic.

Example

```text
Backend

↓

Database
```

Traffic leaves Backend.

This is Egress.

---

# Understanding Pod Selectors

Suppose we have three Pods.

```text
Frontend

Label

app=frontend
```

```text
Backend

Label

app=backend
```

```text
Database

Label

app=database
```

Now consider

```yaml
podSelector:
  matchLabels:
    app: backend
```

Question

Which Pod is affected?

Answer

Only

```text
Backend
```

Frontend and Database remain completely unaffected.

Always remember

A NetworkPolicy protects the Pods selected by the **podSelector**.

---

# Understanding Ingress

Ingress means

Traffic coming **into** a Pod.

Suppose Frontend sends an HTTP request.

```text
Frontend

────────────►

Backend
```

Ask yourself

Who is receiving the request?

Answer

Backend.

Since traffic is entering Backend,

this is

```text
Ingress
```

Think of it as

Someone is knocking on Backend's door.

Backend decides

Should I allow this connection?

or

Should I reject it?

Ingress controls exactly this decision.

---

# Understanding Egress

Egress means

Traffic leaving a Pod.

Suppose Backend connects to Database.

```text
Backend

────────────►

Database
```

Who is sending the request?

Backend.

Traffic is leaving Backend.

This is

```text
Egress
```

A simple rule.

```text
Someone comes to me

↓

Ingress
```

```text
I go to someone

↓

Egress
```

Remember this rule.

It works in almost every scenario.

---

# Understanding Requests and Responses

One concept confuses almost everyone.

Consider

```text
Frontend

────────────►

Backend
```

This request is

For Frontend

```text
Egress
```

For Backend

```text
Ingress
```

Now Backend replies.

```text
Frontend

◄────────────

Backend
```

This response is

For Backend

```text
Egress
```

For Frontend

```text
Ingress
```

The same packet is

Outgoing for one Pod

Incoming for another Pod.

Always think from the perspective of the Pod you are protecting.

---

# Stateful Nature of NetworkPolicies

NetworkPolicies are **stateful**.

This is an extremely important interview question.

Suppose Backend connects to Database.

```text
Backend

────────────►

Database
```

The request is allowed.

Now Database sends back the response.

```text
Backend

◄────────────

Database
```

Will the response be blocked?

No.

The response is automatically allowed.

Why?

Because it belongs to an already established connection.

However,

suppose Database starts a completely new request.

```text
Database

────────────►

Backend
```

Now Kubernetes checks the NetworkPolicy again.

If Backend doesn't allow Database,

the connection is rejected.

Remember

Replies to allowed connections are automatically allowed.

Only **new connections** are evaluated against NetworkPolicies.

---

# NetworkPolicy Terminology

| Term | Meaning |
|------|---------|
| Pod Selector | Selects the Pods to protect |
| Ingress | Incoming traffic |
| Egress | Outgoing traffic |
| Policy Types | Defines whether Ingress or Egress rules are evaluated |
| Labels | Used to identify Pods |
| Allow List | Only explicitly allowed traffic is permitted |

---

# Common Interview Questions

### Question 1

Does Kubernetes block Pod communication by default?

Answer

No.

All Pods can communicate by default.

---

### Question 2

Does NetworkPolicy deny traffic automatically?

Answer

No.

Traffic is denied only for Pods selected by a NetworkPolicy.

---

### Question 3

Does NetworkPolicy replace RBAC?

Answer

No.

RBAC controls Kubernetes API permissions.

NetworkPolicy controls Pod network communication.

---

### Question 4

What is Ingress?

Answer

Traffic entering a Pod.

---

### Question 5

What is Egress?

Answer

Traffic leaving a Pod.

---

### Question 6

Are NetworkPolicies stateful?

Answer

Yes.

Replies to allowed connections are automatically permitted.

---

# Key Takeaways

- Every Pod receives its own IP.
- Every Pod can communicate with every other Pod by default.
- NetworkPolicy works like a firewall for Pods.
- Ingress controls incoming traffic.
- Egress controls outgoing traffic.
- NetworkPolicies are label-based.
- NetworkPolicies are stateful.
- RBAC and NetworkPolicy solve different problems.
- Always think from the perspective of the Pod being protected.
- NetworkPolicies implement the Principle of Least Privilege and Zero Trust Networking.

---

## Next Part

In the next section we will perform a **complete production-style hands-on lab** using:

- Frontend Pod
- Backend Pod
- Database Pod
- Backend Ingress Policy
- Database Ingress Policy
- Connectivity Testing
- Verification
- Architecture Diagrams
- Expected Results
- Troubleshooting

# Production Hands-on Lab

In this lab, we will secure a simple **3-tier application** using Kubernetes NetworkPolicies.

Instead of using generic Pod1, Pod2 and Pod3 examples, we will use a real production architecture.

---

# Lab Architecture

Initially, our application consists of three Pods.

```text
                Frontend

                    │

                    ▼

                Backend

                    │

                    ▼

                Database
```

All Pods are running an NGINX container.

Each Pod has a label.

| Pod | Label |
|-----|-------|
| Frontend | app=frontend |
| Backend | app=backend |
| Database | app=database |

---

# Lab Goal

Our security requirements are:

```text
Frontend

↓

Backend

Allowed
```

```text
Backend

↓

Database

Allowed
```

The following communication should NOT be allowed.

```text
Frontend

↓

Database

Blocked
```

```text
Database

↓

Backend

Blocked
```

Final architecture

```text
                  Allowed
Frontend --------------------► Backend

                                  │
                                  │
                                  │
                                  ▼

                             Database

Frontend --------X--------► Database

Database --------X--------► Backend
```

---

# Step 1

## Create Pods

Create a file

```text
pods.yaml
```

Contents

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  containers:
  - name: nginx
    image: nginx

---
apiVersion: v1
kind: Pod
metadata:
  name: backend
  labels:
    app: backend
spec:
  containers:
  - name: nginx
    image: nginx

---
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    app: database
spec:
  containers:
  - name: nginx
    image: nginx
```

Deploy

```bash
kubectl apply -f pods.yaml
```

Expected Output

```text
pod/frontend created

pod/backend created

pod/database created
```

---

# Step 2

Verify Pods

```bash
kubectl get pods
```

Example

```text
NAME        READY   STATUS

frontend    1/1     Running

backend     1/1     Running

database    1/1     Running
```

Get Pod IPs

```bash
kubectl get pods -o wide
```

Example

```text
NAME        IP

frontend    100.96.1.171

backend     100.96.1.227

database    100.96.1.222
```

Your IP addresses may be different.

---

# Step 3

## Verify Default Kubernetes Networking

Before creating any NetworkPolicy,

let us see the default behavior.

Remember

Kubernetes allows all Pod communication by default.

---

## Test 1

Frontend → Backend

```bash
kubectl exec frontend -- curl -s <BACKEND-IP>
```

Example

```bash
kubectl exec frontend -- curl -s 100.96.1.227
```

Expected Output

```text
Welcome to nginx!
```

Connection Successful

---

## Test 2

Frontend → Database

```bash
kubectl exec frontend -- curl -s <DATABASE-IP>
```

Expected Output

```text
Welcome to nginx!
```

Connection Successful

---

## Test 3

Backend → Database

```bash
kubectl exec backend -- curl -s <DATABASE-IP>
```

Expected Output

```text
Welcome to nginx!
```

Connection Successful

---

Observation

Everything works.

Reason

No NetworkPolicy exists.

Therefore,

all Pods can communicate.

Current architecture

```text
Frontend -------------► Backend

Frontend -------------► Database

Backend --------------► Database

Database -------------► Backend

Everything is Allowed
```

---

# Step 4

## Create Backend Ingress Policy

Question

Which Pod should be protected?

Answer

Backend.

Therefore

```yaml
podSelector:
  matchLabels:
    app: backend
```

Question

Which traffic should be controlled?

Traffic coming into Backend.

Therefore

```yaml
Ingress
```

Question

Who should be allowed?

Only Frontend.

---

Create file

```text
backend-ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: backend-ingress

spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
  - Ingress

  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

Deploy

```bash
kubectl apply -f backend-ingress.yaml
```

Verify

```bash
kubectl get netpol
```

Expected

```text
NAME

backend-ingress
```

---

# Step 5

## Test Backend Policy

Test

Frontend → Backend

```bash
kubectl exec frontend -- curl -m 5 -s <BACKEND-IP>
```

Expected

```text
Welcome to nginx!
```

Allowed

---

Database → Backend

```bash
kubectl exec database -- curl -m 5 -s <BACKEND-IP>
```

Expected

```text
command terminated with exit code 28
```

or

```text
curl: (28) Operation timed out
```

Blocked

---

Frontend → Database

```bash
kubectl exec frontend -- curl -m 5 -s <DATABASE-IP>
```

Expected

```text
Welcome to nginx!
```

Still Allowed

Why?

Because Database has no NetworkPolicy yet.

Current architecture

```text
Frontend ----------► Backend

Database -----X----► Backend

Frontend ----------► Database
```

---

# Step 6

## Create Database Ingress Policy

Question

Which Pod should be protected?

Database.

Question

Who should access Database?

Only Backend.

Create file

```text
database-ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: database-ingress

spec:
  podSelector:
    matchLabels:
      app: database

  policyTypes:
  - Ingress

  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```

Deploy

```bash
kubectl apply -f database-ingress.yaml
```

Verify

```bash
kubectl get netpol
```

Expected

```text
NAME

backend-ingress

database-ingress
```

---

# Step 7

## Final Testing

Test 1

Frontend → Backend

```bash
kubectl exec frontend -- curl -m 5 -s <BACKEND-IP>
```

Expected

```text
Welcome to nginx!
```

Allowed

---

Test 2

Backend → Database

```bash
kubectl exec backend -- curl -m 5 -s <DATABASE-IP>
```

Expected

```text
Welcome to nginx!
```

Allowed

---

Test 3

Frontend → Database

```bash
kubectl exec frontend -- curl -m 5 -s <DATABASE-IP>
```

Expected

```text
command terminated with exit code 28
```

Blocked

---

Test 4

Database → Backend

```bash
kubectl exec database -- curl -m 5 -s <BACKEND-IP>
```

Expected

```text
command terminated with exit code 28
```

Blocked

---

# Final Secure Architecture

```text
                   Allowed
Frontend ----------------------► Backend
                                    │
                                    │
                                    ▼
                               Database

Frontend -----------X----------► Database

Database -----------X----------► Backend
```

Communication Matrix

| Source | Destination | Result |
|---------|-------------|--------|
| Frontend | Backend | ✅ Allowed |
| Backend | Database | ✅ Allowed |
| Frontend | Database | ❌ Blocked |
| Database | Backend | ❌ Blocked |

---

# Understanding Why Database → Backend is Blocked

Many beginners think it is blocked because Database has no Egress policy.

This is incorrect.

The actual reason is:

Backend has an Ingress policy.

Backend only accepts traffic from Frontend.

When Database starts a new connection,

Backend checks its Ingress policy.

Database is not in the allowed list.

Therefore,

the connection is rejected.

Remember

Ingress controls

"Who can come to me?"

Egress controls

"Where can I go?"

---

# Verify CNI Supports NetworkPolicy

NetworkPolicies require a CNI plugin that implements them.

Example

```bash
kubectl get pods -n kube-system
```

Example Output

```text
cilium-xxxxx
```

In this lab,

the cluster uses

**Cilium**

Therefore,

all NetworkPolicies are enforced correctly.

---

# Common Mistakes

### Mistake 1

Forgetting labels.

Example

```yaml
app: backend
```

If labels don't match,

the policy never applies.

---

### Mistake 2

Protecting the wrong Pod.

Always ask

Which Pod am I protecting?

---

### Mistake 3

Confusing Ingress and Egress.

Remember

```text
Someone comes to me

↓

Ingress
```

```text
I go to someone

↓

Egress
```

---

### Mistake 4

Thinking replies are blocked.

NetworkPolicies are Stateful.

Replies to allowed connections are always permitted.

---

# Practical Summary

Pods created

✅ Frontend

✅ Backend

✅ Database

Policies created

✅ backend-ingress

✅ database-ingress

Traffic Allowed

✅ Frontend → Backend

✅ Backend → Database

Traffic Blocked

❌ Frontend → Database

❌ Database → Backend

Congratulations!

You have successfully implemented a production-style 3-tier application security model using Kubernetes NetworkPolicies.

# Advanced NetworkPolicy Concepts

So far, we have created two Ingress policies.

```text
Frontend ─────────► Backend

Backend ─────────► Database
```

Everything else is blocked.

Now let's understand how NetworkPolicies work in real production environments.

---

# Default Deny NetworkPolicy

One of the biggest security best practices is:

> Deny everything first.

Then,

allow only the required communication.

This is known as the **Default Deny** approach.

Instead of writing hundreds of deny rules,

we create one policy that blocks everything,

then gradually allow required traffic.

---

## Why Default Deny?

Imagine a namespace containing

```text
Frontend

Backend

Database

Redis

Prometheus

Grafana

Jenkins

SonarQube
```

If Kubernetes allows all communication,

every Pod can communicate with every other Pod.

This creates a huge attack surface.

Instead,

we deny everything first.

---

## Default Deny Ingress

Create

```text
default-deny-ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny-ingress

spec:
  podSelector: {}

  policyTypes:
  - Ingress
```

Apply

```bash
kubectl apply -f default-deny-ingress.yaml
```

What does

```yaml
podSelector: {}
```

mean?

It selects

ALL Pods

inside the namespace.

No Ingress rules exist.

Therefore,

every incoming connection is denied.

---

Current Situation

```text
Frontend ------X------► Backend

Frontend ------X------► Database

Backend -------X------► Database
```

Everything is blocked.

---

## Default Deny Egress

Ingress blocks incoming traffic.

Egress blocks outgoing traffic.

Create

```text
default-deny-egress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny-egress

spec:
  podSelector: {}

  policyTypes:
  - Egress
```

Apply

```bash
kubectl apply -f default-deny-egress.yaml
```

Now

every Pod

is prevented from creating new outgoing connections.

---

Default Deny Both

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny-all

spec:
  podSelector: {}

  policyTypes:
  - Ingress
  - Egress
```

This is a common production baseline.

---

# Understanding Egress Policies

Until now,

we controlled

who can come to a Pod.

Now,

we will control

where a Pod can go.

Suppose

```text
Backend
```

should communicate only with

```text
Database
```

and nowhere else.

---

## Backend Egress Policy

Create

```text
backend-egress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: backend-egress

spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
  - Egress

  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
```

Meaning

Protect

Backend.

Allow outgoing traffic

only

to Database.

---

Current Architecture

```text
Backend ------------► Database

Allowed

Backend ------X------► Frontend

Blocked
```

---

# How Ingress and Egress Work Together

Suppose

Backend

has

```text
Egress

↓

Database
```

Database

has

```text
Ingress

↓

Backend
```

Result

```text
Backend ------------► Database

Allowed
```

Both sides agree.

Communication succeeds.

Now

suppose

Backend allows

Database,

but

Database blocks Backend.

```text
Backend ------------► Database

Blocked
```

Why?

Because both policies must permit the communication.

Think of it as

```text
Sender

↓

Permission

+

Receiver

↓

Permission

↓

Connection Allowed
```

---

# Multiple NetworkPolicies

A Pod may have

more than one

NetworkPolicy.

Example

```text
Policy A

Policy B

Policy C
```

Does Kubernetes process them

top to bottom?

No.

There is

NO priority.

NO ordering.

NO first match.

Instead,

Kubernetes combines all policies.

This is called

the **Union of Allowed Traffic**.

---

Example

Policy A

allows

```text
Frontend
```

Policy B

allows

```text
Prometheus
```

Final Result

```text
Frontend

Allowed

+

Prometheus

Allowed
```

Both can communicate.

---

Example Diagram

```text
             Policy A

Frontend ------------► Backend



             Policy B

Prometheus ----------► Backend
```

Backend finally allows

```text
Frontend

+

Prometheus
```

---

# namespaceSelector

Sometimes,

Pods in another namespace

should communicate.

Example

Namespace

```text
monitoring
```

contains

```text
Prometheus
```

Application namespace

contains

```text
Backend
```

Prometheus should scrape metrics.

Instead of selecting Pods,

we can select namespaces.

Example

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring
```

Meaning

Allow traffic

from every Pod

inside

the monitoring namespace.

---

# Pod Selector + Namespace Selector

This is very common in production.

Suppose

Only

Prometheus

inside

Monitoring namespace

should communicate.

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring

    podSelector:
      matchLabels:
        app: prometheus
```

Meaning

Namespace alone

is not enough.

Pod label alone

is not enough.

Both must match.

---

Diagram

```text
Monitoring Namespace

Prometheus

Allowed

Grafana

Blocked
```

---

# IPBlock

Sometimes,

traffic originates

outside Kubernetes.

Example

Corporate VPN

10.0.0.0/24

Allow only

that subnet.

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/24
```

You can also exclude addresses.

Example

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/24

      except:
      - 10.0.0.50/32
```

Meaning

Allow

10.0.0.0/24

except

10.0.0.50

---

# Combining Rules

Example

```yaml
ingress:
- from:

  - namespaceSelector:
      matchLabels:
        name: monitoring

  - podSelector:
      matchLabels:
        app: frontend

  - ipBlock:
      cidr: 10.10.10.0/24
```

Backend accepts traffic from

- Monitoring Namespace
- Frontend Pods
- Corporate Network

---

# Production Example 1

E-Commerce

```text
Internet

↓

Frontend

↓

Backend

↓

Database
```

Policies

Frontend

↓

Backend

Allowed

Backend

↓

Database

Allowed

Everything else

Blocked

---

# Production Example 2

Monitoring

```text
Prometheus

↓

Backend

Allowed
```

Grafana

↓

Backend

Blocked

---

# Production Example 3

CI/CD

```text
Jenkins

↓

SonarQube

Allowed
```

Developer Pods

↓

SonarQube

Blocked

---

# Production Example 4

Banking Application

```text
Customer Portal

↓

Payment API

↓

Core Banking Database
```

Customer Portal

↓

Database

Blocked

Payment API

↓

Database

Allowed

---

# Best Practices

✅ Use Default Deny

✅ Allow only required traffic

✅ Use meaningful labels

✅ Test after every policy

✅ Keep policies small

✅ Document communication flow

✅ Use namespaces

✅ Follow Least Privilege

---

# Common Interview Questions

### What does

```yaml
podSelector: {}
```

mean?

Answer

Select all Pods.

---

### Are NetworkPolicies Deny Rules?

No.

NetworkPolicies are

Allow Lists.

Anything not allowed

is denied.

---

### Can multiple NetworkPolicies apply?

Yes.

Kubernetes combines all allowed traffic.

---

### Which policy has higher priority?

None.

There is no ordering.

---

### Can NetworkPolicies control Services?

No.

They control Pod traffic.

---

### Can NetworkPolicies filter HTTP URLs?

No.

They operate at Layer 3/Layer 4.

They understand

IP addresses,

Ports,

Protocols.

Not

HTTP Paths,

Headers,

Cookies.

---

# Key Takeaways

- Default Kubernetes networking is open.
- Default Deny is a production best practice.
- Ingress controls incoming traffic.
- Egress controls outgoing traffic.
- Both sender and receiver policies may affect communication.
- Multiple policies are additive.
- `podSelector: {}` selects every Pod.
- `namespaceSelector` controls namespaces.
- `ipBlock` controls CIDR ranges.
- NetworkPolicies are Layer 3 / Layer 4 firewall rules.

# Troubleshooting NetworkPolicies

When a NetworkPolicy is not working as expected, follow this checklist.

---

## Step 1 - Verify NetworkPolicy Exists

```bash
kubectl get networkpolicy
```

Example

```text
NAME                 POD-SELECTOR

backend-ingress      app=backend

database-ingress     app=database
```

---

## Step 2 - Describe the NetworkPolicy

```bash
kubectl describe networkpolicy backend-ingress
```

Verify:

- Pod Selector
- Policy Type
- Allowed Pods
- Allowed Ports
- Namespace

---

## Step 3 - Verify Pod Labels

NetworkPolicies depend entirely on labels.

Check labels.

```bash
kubectl get pods --show-labels
```

Example

```text
NAME        LABELS

frontend    app=frontend

backend     app=backend

database    app=database
```

If labels don't match,

the policy won't apply.

---

## Step 4 - Verify Pod IP

```bash
kubectl get pods -o wide
```

---

## Step 5 - Test Connectivity

Example

```bash
kubectl exec frontend -- curl -m 5 -s <BACKEND-IP>
```

or

```bash
kubectl exec backend -- curl -m 5 -s <DATABASE-IP>
```

---

## Step 6 - Check CNI Plugin

NetworkPolicies require a CNI that supports them.

Example

```bash
kubectl get pods -n kube-system
```

Example Output

```text
cilium

calico

antrea
```

If your CNI doesn't support NetworkPolicies,

they will not be enforced.

---

## Step 7 - Verify Cilium

If using Cilium

```bash
kubectl get pods -n kube-system | grep cilium
```

Pods should be Running.

---

# Common Mistakes

## Mistake 1

Wrong Pod Selector

Wrong

```yaml
podSelector:
  matchLabels:
    app: backends
```

Correct

```yaml
podSelector:
  matchLabels:
    app: backend
```

---

## Mistake 2

Wrong Label

Policy

```yaml
app: frontend
```

Pod

```yaml
application: frontend
```

Labels must match exactly.

---

## Mistake 3

Confusing Ingress and Egress

Remember

```text
Traffic comes to me

↓

Ingress
```

```text
I go somewhere

↓

Egress
```

---

## Mistake 4

Thinking Replies Are Blocked

NetworkPolicies are Stateful.

Replies to allowed connections are always permitted.

---

## Mistake 5

Thinking NetworkPolicies are Deny Rules

They are Allow Lists.

Anything not explicitly allowed

is denied.

---

## Mistake 6

Forgetting Default Behavior

Without NetworkPolicies

Everything is allowed.

---

## Mistake 7

Assuming Rule Order Matters

There is no priority.

There is no first-match.

Kubernetes combines all allowed traffic.

---

# Best Practices

Use Default Deny.

Allow only required traffic.

Use meaningful labels.

Document application communication.

Keep policies small.

Use namespaces to isolate applications.

Review policies regularly.

Test every policy before production.

Use least privilege.

Never rely only on NetworkPolicies.

Combine them with

- RBAC
- Pod Security Admission
- Secrets
- Service Accounts
- Image Scanning

---

# Production Scenarios

## Banking Application

```text
Customer

↓

Web

↓

Payment API

↓

Database
```

Allowed

```text
Web

↓

Payment API
```

Allowed

```text
Payment API

↓

Database
```

Blocked

```text
Web

↓

Database
```

---

## E-Commerce

```text
Frontend

↓

Backend

↓

Redis

↓

Database
```

Allowed

Frontend → Backend

Backend → Redis

Backend → Database

Blocked

Frontend → Database

Frontend → Redis

Redis → Frontend

---

## Monitoring

```text
Prometheus

↓

Application Pods
```

Allowed

Prometheus → Application

Blocked

Grafana → Application

unless explicitly required.

---

## CI/CD

```text
Developer

↓

Git

↓

Jenkins

↓

SonarQube

↓

Kubernetes
```

Only Jenkins should communicate with SonarQube.

Developer Pods should never access SonarQube directly.

---

# Interview Questions

## Beginner

### What is a NetworkPolicy?

Controls Pod-to-Pod network communication.

---

### What is Ingress?

Incoming traffic to a Pod.

---

### What is Egress?

Outgoing traffic from a Pod.

---

### Does Kubernetes block Pod communication by default?

No.

All Pods communicate by default.

---

### Does RBAC replace NetworkPolicy?

No.

RBAC controls API permissions.

NetworkPolicy controls Pod traffic.

---

### What is podSelector?

Selects the Pods that the policy protects.

---

### What is namespaceSelector?

Selects Pods from another namespace.

---

### What is ipBlock?

Allows traffic from a CIDR block.

---

## Intermediate

### Are NetworkPolicies Stateful?

Yes.

Replies to allowed connections are automatically allowed.

---

### Can multiple policies apply to one Pod?

Yes.

All allowed traffic is combined.

---

### What happens if no policy exists?

Everything is allowed.

---

### What does

```yaml
podSelector: {}
```

mean?

Select all Pods in the namespace.

---

### What happens if labels don't match?

The policy does not apply.

---

### Does policy order matter?

No.

---

### Which Kubernetes object enforces NetworkPolicies?

The CNI plugin.

Examples

- Cilium
- Calico
- Antrea

---

## Advanced

### Difference between RBAC and NetworkPolicy?

RBAC

↓

API Server

NetworkPolicy

↓

Pod Network

---

### Can NetworkPolicies filter HTTP URLs?

No.

They operate at Layer 3 and Layer 4.

---

### Can NetworkPolicies secure traffic between namespaces?

Yes.

Using namespaceSelector.

---

### Can NetworkPolicies secure external IP ranges?

Yes.

Using ipBlock.

---

### Are Services affected?

Indirectly.

Policies apply to Pods,

not Services.

---

### Can one Pod have multiple NetworkPolicies?

Yes.

Kubernetes combines all allowed rules.

---

### Can NetworkPolicies block DNS?

Yes.

If Egress policies don't allow DNS traffic,

name resolution may fail.

---

### Why is my policy not working?

Possible reasons

- Wrong labels
- Unsupported CNI
- Wrong namespace
- Incorrect selector
- Testing wrong Pod
- Typographical errors

---

# Practical Exercises

## Exercise 1

Create four Pods

```text
Frontend

Backend

Redis

Database
```

Requirements

Frontend

↓

Backend

Allowed

Backend

↓

Redis

Allowed

Backend

↓

Database

Allowed

Everything else

Blocked.

---

## Exercise 2

Create two namespaces

```text
Production

Monitoring
```

Allow

Prometheus

↓

Backend

Only.

---

## Exercise 3

Create

Default Deny

for an entire namespace.

Then gradually allow

Frontend

↓

Backend

↓

Database.

---

## Exercise 4

Allow only

CIDR

```text
10.0.0.0/24
```

using

ipBlock.

---

# Commands Cheat Sheet

Create Policy

```bash
kubectl apply -f policy.yaml
```

List Policies

```bash
kubectl get networkpolicy
```

Describe Policy

```bash
kubectl describe networkpolicy
```

Delete Policy

```bash
kubectl delete networkpolicy backend-ingress
```

View Labels

```bash
kubectl get pods --show-labels
```

View Pod IP

```bash
kubectl get pods -o wide
```

Test Connectivity

```bash
kubectl exec <pod-name> -- curl -m 5 -s <pod-ip>
```

---

# One-Page Revision

NetworkPolicy

↓

Firewall for Pods

Ingress

↓

Incoming Traffic

Egress

↓

Outgoing Traffic

podSelector

↓

Select Protected Pods

namespaceSelector

↓

Select Namespace

ipBlock

↓

Select CIDR Range

Default Kubernetes

↓

Allow All

Default Deny

↓

Block Everything

RBAC

↓

API Permissions

NetworkPolicy

↓

Pod Communication

Replies

↓

Allowed

New Connections

↓

Checked

Policies

↓

Additive

CNI

↓

Enforces NetworkPolicies

Examples

↓

Cilium

Calico

Antrea

---

# Summary

In this chapter, you learned:

- Kubernetes default networking
- Why NetworkPolicies are required
- Ingress
- Egress
- Stateful behavior
- Backend Ingress Policy
- Database Ingress Policy
- Default Deny Policies
- Egress Policies
- namespaceSelector
- ipBlock
- Multiple NetworkPolicies
- Production security design
- Troubleshooting
- Interview questions
- Best practices
- Practical labs

Congratulations!

You now understand how Kubernetes secures Pod-to-Pod communication using NetworkPolicies and are ready to design secure networking for production workloads.
