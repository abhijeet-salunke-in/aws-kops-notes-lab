# Kubernetes ClusterRole and ClusterRoleBinding

## Introduction

In the previous topic, we learned about **Role** and **RoleBinding**. They are used to provide permissions **inside a single namespace**.

However, Kubernetes also contains many resources that do not belong to any namespace. These resources exist at the **cluster level**.

To control access to these resources, Kubernetes provides:

- ClusterRole
- ClusterRoleBinding

In this chapter, we will understand why they are required, how they work internally, and perform practical implementations.

---

# Prerequisites

Before working with ClusterRole and ClusterRoleBinding, we must already have a **Kubernetes user identity**.

In our previous practical, we created a user named **vishal** using certificate authentication.

The complete process was:

- Generate Private Key
- Generate CSR (Certificate Signing Request)
- Extract Kubernetes CA Certificate and CA Key
- Sign the CSR using Kubernetes CA
- Create Client Certificate
- Register the user inside kubeconfig
- Create a Context for the user

Example:

```bash
kubectl config set-credentials vishal \
  --client-certificate=/root/vishal.crt \
  --client-key=/root/vishal.key

kubectl config set-context vishal-context \
  --cluster=abhi.k8s.local \
  --user=vishal
```

After switching to this context,

```bash
kubectl config use-context vishal-context
```

Kubernetes successfully recognized the identity as:

```
User = vishal
```

But this user still had **no permissions**, because Kubernetes only authenticated the user.

Authorization still needed to be configured using RBAC.

---

# Authentication vs Authorization

One of the most common interview questions is the difference between Authentication and Authorization.

## Authentication

Authentication answers the question:

> **Who are you?**

In our practical,

- Private Key proves our identity.
- CSR requests a certificate.
- Kubernetes CA signs the certificate.
- kubeconfig uses that certificate.

Result:

```
User Identity = vishal
```

Authentication only verifies identity.

It does **not** decide what the user can do.

---

## Authorization

Authorization answers the question:

> **What are you allowed to do?**

Authorization is controlled using RBAC objects such as:

- Role
- RoleBinding
- ClusterRole
- ClusterRoleBinding

These decide:

- Which resources can be accessed
- Which actions are allowed
- Which user receives those permissions

---

# Kubernetes Resource Types

Before understanding ClusterRole, we must understand that Kubernetes resources are divided into **two categories**.

## 1. Namespace Resources

These resources always belong to a namespace.

Examples:

- Pods
- Deployments
- ReplicaSets
- StatefulSets
- DaemonSets
- Services
- ConfigMaps
- Secrets
- Jobs
- CronJobs
- PVCs
- Role
- RoleBinding

These resources exist **inside** a namespace.

Example:

```
dev Namespace
 ├── Pod
 ├── Deployment
 ├── Secret
 ├── ConfigMap

production Namespace
 ├── Pod
 ├── Deployment
 ├── Secret
```

Each namespace contains its own independent resources.

---

## 2. Cluster Resources

Some resources belong to the entire Kubernetes cluster.

They are not stored inside any namespace.

Examples:

- Nodes
- Namespaces
- PersistentVolumes (PV)
- StorageClasses
- ClusterRole
- ClusterRoleBinding
- CSINodes
- VolumeAttachments

These resources are shared by the whole cluster.

Example:

```
Entire Kubernetes Cluster

 ├── Nodes
 ├── Namespaces
 ├── StorageClasses
 ├── PersistentVolumes
 ├── ClusterRoles
 └── ClusterRoleBindings
```

---

# Why Role Cannot Access Cluster Resources

A **Role** always belongs to one namespace.

Example:

```
Role
Namespace = dev
```

Since it is confined to a namespace, it cannot control permissions for resources that exist outside namespaces.

For example,

Suppose user **vishal** executes:

```bash
kubectl get nodes
```

Nodes are cluster resources.

A Role inside the **dev** namespace has no authority over cluster resources.

Therefore Kubernetes returns:

```
Error from server (Forbidden)
```

---

# Why ClusterRole Exists

ClusterRole was introduced to provide permissions for resources that belong to the entire Kubernetes cluster.

Examples:

```
kubectl get nodes

kubectl get namespaces

kubectl get storageclass

kubectl get pv
```

These commands require permissions that are not tied to a namespace.

Only a **ClusterRole** can grant access to these resources.

---

# Key Points

- Authentication verifies the user's identity.
- Authorization decides the user's permissions.
- Certificate Authentication creates the user identity.
- RBAC grants permissions.
- Namespace resources are controlled using Role and RoleBinding.
- Cluster resources are controlled using ClusterRole and ClusterRoleBinding.
- Without RBAC, an authenticated user cannot perform operations.
- ClusterRole is designed for cluster-level resources.

---

## What We Will Learn Next

In the next part, we will understand:

- Role vs ClusterRole
- RoleBinding vs ClusterRoleBinding
- Complete comparison table
- Internal working of RBAC
- Which combination gives access to namespace resources and which gives access to cluster resources

# Role vs ClusterRole

Although both Role and ClusterRole define permissions, they are used in different situations.

The biggest difference is **where those permissions apply**.

---

# Role

A Role is used to grant permissions **inside a single namespace**.

It cannot control cluster-level resources.

Example:

```
Namespace: dev

Role
 ├── get pods
 ├── list pods
 └── watch pods
```

This Role only works inside the **dev** namespace.

Even if another namespace has Pods, this Role cannot access them.

Example:

```
kubectl get pods -n dev
```

Allowed ✔

```
kubectl get pods -n production
```

Not Allowed ❌

---

# ClusterRole

A ClusterRole defines permissions at the cluster level.

It is mainly used for cluster resources like:

- Nodes
- Namespaces
- StorageClasses
- PersistentVolumes

Example:

```
ClusterRole

get nodes
list nodes
watch nodes
```

Now the user can execute

```bash
kubectl get nodes
```

if that ClusterRole is assigned.

---

# Can ClusterRole Control Namespace Resources?

Yes.

This is an important concept.

Many beginners think ClusterRole only works with cluster resources.

That is **not true**.

ClusterRole can also contain permissions for namespace resources.

Example:

```yaml
rules:
- apiGroups: [""]
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
```

This ClusterRole contains Pod permissions.

Whether these Pod permissions apply to one namespace or all namespaces depends on **how the ClusterRole is bound**.

This is where RoleBinding and ClusterRoleBinding become important.

---

# RoleBinding vs ClusterRoleBinding

These objects **do not create permissions**.

They only assign existing permissions to users, groups, or service accounts.

Think of it like this:

```
Role / ClusterRole
        │
        │ contains permissions
        ▼

RoleBinding / ClusterRoleBinding
        │
        │ assigns permissions
        ▼

User
```

---

# RoleBinding

RoleBinding always works inside one namespace.

It can bind:

- a Role
- or a ClusterRole

Example:

```
Namespace = dev

RoleBinding

↓

User = vishal
```

The permissions are available **only inside the dev namespace**.

---

# ClusterRoleBinding

ClusterRoleBinding works at the entire cluster level.

It binds only a ClusterRole.

Example:

```
ClusterRoleBinding

↓

ClusterRole

↓

User
```

The permissions become available across the cluster.

---

# Permission Combinations

Understanding these combinations is very important.

## 1. Role + RoleBinding

```
Role
     ↓
RoleBinding
     ↓
User
```

Result:

✔ Access only inside one namespace.

Example:

```
kubectl get pods -n dev
```

Allowed

```
kubectl get pods -A
```

Forbidden

---

## 2. ClusterRole + RoleBinding

```
ClusterRole
      ↓
RoleBinding
      ↓
User
```

Although the ClusterRole contains permissions, the RoleBinding limits those permissions to **one namespace**.

Example:

ClusterRole contains:

```
pods
get
list
watch
```

RoleBinding exists in namespace **dev**.

Result:

```
kubectl get pods -n dev
```

Allowed ✔

```
kubectl get pods -A
```

Forbidden ❌

---

## 3. ClusterRole + ClusterRoleBinding

```
ClusterRole
       ↓
ClusterRoleBinding
       ↓
User
```

Permissions apply across the entire cluster.

Example:

```
kubectl get pods -A
```

Allowed ✔

```
kubectl get nodes
```

Allowed ✔ (if nodes are included in the ClusterRole)

---

# Can We Use Role with ClusterRoleBinding?

No.

This is one of the most common interview questions.

A ClusterRoleBinding can reference **only a ClusterRole**.

It cannot reference a Role.

Therefore, the following combination is **invalid**:

```
Role
   ↓
ClusterRoleBinding
```

Kubernetes does not allow this.

---

# Summary Table

| Combination | Namespace Access | Cluster-wide Access |
|-------------|------------------|---------------------|
| Role + RoleBinding | ✅ Yes | ❌ No |
| ClusterRole + RoleBinding | ✅ Yes (only the binding namespace) | ❌ No |
| ClusterRole + ClusterRoleBinding | ✅ Yes | ✅ Yes |
| Role + ClusterRoleBinding | ❌ Invalid | ❌ Invalid |

---

# Internal Working

Whenever a request reaches the Kubernetes API Server, RBAC checks permissions in this order:

```
User Request
      │
      ▼
Authentication
      │
      ▼
RBAC Authorization
      │
      ├── RoleBindings
      │
      ├── ClusterRoleBindings
      │
      ▼
Permission Found?
      │
 ┌────┴────┐
 │         │
Yes        No
 │         │
 ▼         ▼
Allow    Forbidden
```

---

# Key Points

- Role is namespace scoped.
- ClusterRole is cluster scoped.
- ClusterRole can also define permissions for namespace resources.
- RoleBinding can bind either a Role or a ClusterRole.
- ClusterRoleBinding can bind only a ClusterRole.
- ClusterRole + RoleBinding limits permissions to one namespace.
- ClusterRole + ClusterRoleBinding grants permissions across the cluster.
- Role + ClusterRoleBinding is not a valid Kubernetes RBAC configuration.

---

## What We Will Learn Next

In the next part, we will start the practical implementation by:

- Switching to the administrator context
- Creating a ClusterRole
- Granting permissions to read Nodes
- Understanding each YAML field before applying it

# Practical: Creating ClusterRole and ClusterRoleBinding

In this practical, we will give the user **vishal** permission to read **Nodes**.

Since Nodes are cluster-level resources, we must use:

- ClusterRole
- ClusterRoleBinding

---

# Lab Architecture

```
               Kubernetes Cluster
                      │
      ┌───────────────┴───────────────┐
      │                               │
 Control Plane                    Worker Node
      │                               │
      └───────────────┬───────────────┘
                      │
                 Cluster Resource
                      │
                    Nodes
                      │
             ClusterRole (node-reader)
                      │
             ClusterRoleBinding
                      │
                  User: vishal
```

Our goal is:

```
vishal

↓

kubectl get nodes

↓

Success
```

But the user should **not** be able to access Pods, Deployments, Secrets, or other resources because we are granting permission only for Nodes.

---

# Step 1: Switch to Administrator Context

Before creating RBAC objects, we must switch to the administrator context.

Check the current context:

```bash
kubectl config current-context
```

Expected output:

```
abhi.k8s.local
```

If you are using the user context, switch back:

```bash
kubectl config use-context abhi.k8s.local
```

Verify again:

```bash
kubectl config current-context
```

Output:

```
abhi.k8s.local
```

### Why?

Only an administrator has permission to create ClusterRoles and ClusterRoleBindings.

If user **vishal** tries to create them, Kubernetes returns:

```
Error from server (Forbidden)
```

---

# Step 2: Create the ClusterRole YAML

Create a file:

```bash
vim clusterrole.yaml
```

Paste:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: node-reader

rules:
- apiGroups: [""]
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
```

Save the file.

---

# Understanding Each Field

## apiVersion

```yaml
apiVersion: rbac.authorization.k8s.io/v1
```

RBAC resources belong to the RBAC API group.

---

## kind

```yaml
kind: ClusterRole
```

This tells Kubernetes that we are creating a **ClusterRole**, not a Role.

---

## metadata

```yaml
metadata:
  name: node-reader
```

The ClusterRole will be stored with the name:

```
node-reader
```

---

## rules

Rules define **what permissions** this ClusterRole contains.

---

### apiGroups

```yaml
apiGroups: [""]
```

Nodes belong to the **core API group**, represented by an empty string.

---

### resources

```yaml
resources:
- nodes
```

We are granting permissions only for the **nodes** resource.

Nothing else is included.

---

### verbs

```yaml
verbs:
- get
- list
- watch
```

This allows the user to:

- Get a specific node
- List all nodes
- Watch node changes

The user still cannot:

- Create nodes
- Delete nodes
- Edit nodes

---

# Step 3: Apply the ClusterRole

```bash
kubectl apply -f clusterrole.yaml
```

Expected output:

```
clusterrole.rbac.authorization.k8s.io/node-reader created
```

---

# Step 4: Verify the ClusterRole

List the ClusterRole:

```bash
kubectl get clusterrole node-reader
```

Expected output:

```
NAME          CREATED AT
node-reader
```

Describe it:

```bash
kubectl describe clusterrole node-reader
```

Expected output:

```
Resources:
  nodes

Verbs:
  get
  list
  watch
```

This confirms that the ClusterRole has been created successfully.

---

# What Has Happened So Far?

We have created a **permission set** called **node-reader**.

It contains permission to read Nodes.

However, no user can use these permissions yet because the ClusterRole is **not attached to anyone**.

Think of it like this:

```
Permission Created

↓

ClusterRole

↓

(No User Assigned Yet)

↓

No Effect
```

The next step is to create a **ClusterRoleBinding**, which will attach this permission set to the user **vishal**.

---

# Key Points

- ClusterRole stores permissions only.
- It is not assigned to any user automatically.
- The ClusterRole can now read Nodes.
- No user has these permissions yet.
- We need a ClusterRoleBinding to assign the permissions.

---

## What We Will Learn Next

In the next part, we will:

- Create a ClusterRoleBinding
- Bind the **node-reader** ClusterRole to the **vishal** user
- Verify that **vishal** can successfully run:

```bash
kubectl get nodes
```

while still being denied access to Pods and other resources.

# Practical: Creating a ClusterRoleBinding

In the previous part, we created a **ClusterRole** named **node-reader**.

That ClusterRole contains permissions to read Nodes.

However, those permissions are still **not assigned to any user**.

Now we will bind those permissions to the user **vishal**.

---

# Before Creating the ClusterRoleBinding

Current situation:

```
                ClusterRole
               (node-reader)

             get
             list
             watch
                 │
                 │
                 ▼

        Not attached to anyone
```

If **vishal** tries:

```bash
kubectl get nodes
```

Result:

```
Error from server (Forbidden)
```

because Kubernetes has not yet associated those permissions with the user.

---

# Step 1: Create the ClusterRoleBinding YAML

Create a file:

```bash
vim clusterrolebinding.yaml
```

Paste the following:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: node-reader-binding

subjects:
- kind: User
  name: vishal
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

Save the file.

---

# Understanding Every Field

## apiVersion

```yaml
apiVersion: rbac.authorization.k8s.io/v1
```

This resource belongs to the Kubernetes RBAC API.

---

## kind

```yaml
kind: ClusterRoleBinding
```

This tells Kubernetes that we are creating a ClusterRoleBinding.

A ClusterRoleBinding attaches a ClusterRole to a user, group, or service account.

---

## metadata

```yaml
metadata:
  name: node-reader-binding
```

This is simply the name of the ClusterRoleBinding object.

---

## subjects

This section answers the question:

**Who will receive these permissions?**

```yaml
subjects:
```

Our subject is:

```yaml
kind: User
```

Meaning we are binding permissions to a Kubernetes User.

---

```yaml
name: vishal
```

The authenticated identity is:

```
vishal
```

This must exactly match the Common Name (CN) inside the client certificate.

Earlier we created:

```
CN=vishal
```

That is why Kubernetes recognizes this user.

---

```yaml
apiGroup: rbac.authorization.k8s.io
```

This tells Kubernetes that the subject belongs to the RBAC API group.

---

# roleRef

This section answers another important question:

**Which permission set should be assigned?**

```yaml
roleRef:
```

---

```yaml
kind: ClusterRole
```

We are referencing a ClusterRole.

---

```yaml
name: node-reader
```

Use the ClusterRole that we created in the previous step.

---

```yaml
apiGroup: rbac.authorization.k8s.io
```

Again, this tells Kubernetes that the referenced object belongs to the RBAC API.

---

# Internal Flow

```
User
vishal
     │
     ▼

ClusterRoleBinding
(node-reader-binding)

     │
     ▼

ClusterRole
(node-reader)

     │
     ▼

Permissions

get
list
watch

Resource

Nodes
```

Now Kubernetes knows:

> User **vishal** can use the permissions stored inside **node-reader**.

---

# Step 2: Apply the ClusterRoleBinding

```bash
kubectl apply -f clusterrolebinding.yaml
```

Expected output:

```
clusterrolebinding.rbac.authorization.k8s.io/node-reader-binding created
```

---

# Step 3: Verify the ClusterRoleBinding

List it:

```bash
kubectl get clusterrolebinding node-reader-binding
```

Describe it:

```bash
kubectl describe clusterrolebinding node-reader-binding
```

Expected output:

```
Role:

Kind: ClusterRole

Name: node-reader
```

and

```
Subjects:

Kind: User

Name: vishal
```

This confirms that:

- the **node-reader** ClusterRole
- has been assigned
- to the user **vishal**

---

# What Has Happened?

Previously:

```
Permissions

↓

No User

↓

Nobody can use them.
```

Now:

```
Permissions

↓

ClusterRole

↓

ClusterRoleBinding

↓

User

↓

Permission Granted
```

The RBAC chain is now complete.

---

# Step 4: Test as the User

Switch to the user context:

```bash
kubectl config use-context vishal-context
```

Verify:

```bash
kubectl config current-context
```

Expected output:

```
vishal-context
```

Now test:

```bash
kubectl get nodes
```

Expected:

```
NAME                  STATUS
worker-node           Ready
control-plane         Ready
```

The command should succeed because the user now has permission to read Nodes.

---

# Verify Restricted Access

Even though **vishal** can read Nodes, try:

```bash
kubectl get pods -A
```

Expected:

```
Error from server (Forbidden)
```

Also try:

```bash
kubectl get namespaces
```

Expected:

```
Error from server (Forbidden)
```

This proves that only the permissions defined in the **ClusterRole** are available.

---

# What We Learned

- A ClusterRole only stores permissions.
- A ClusterRoleBinding assigns those permissions to a user.
- The user **vishal** can now access Nodes.
- The same user still cannot access Pods or Namespaces because those permissions were never granted.
- RBAC follows the principle of **least privilege**: users receive only the permissions they explicitly need.

---

## Next Part

In the final part, we will verify the behavior from both the **administrator** and **vishal** contexts, summarize the complete RBAC flow, and compare **Role/RoleBinding** with **ClusterRole/ClusterRoleBinding** using practical examples.

# Practical Verification, Interview Questions and Summary

In this chapter, we successfully implemented **ClusterRole** and **ClusterRoleBinding** to grant cluster-level permissions to a Kubernetes user.

Let's verify the complete authorization flow and understand what happened internally.

---

# Practical Verification

After creating the **ClusterRole** and **ClusterRoleBinding**, we switched to the **vishal** context.

```bash
kubectl config use-context vishal-context
```

Verify:

```bash
kubectl config current-context
```

Output:

```
vishal-context
```

Now execute:

```bash
kubectl get nodes
```

Output:

```
NAME                  STATUS
control-plane         Ready
worker-node           Ready
```

This confirms that the RBAC configuration is working correctly.

---

# Verify Unauthorized Operations

Although **vishal** can now read Nodes, he still cannot access other resources.

Example:

```bash
kubectl get pods -A
```

Output:

```
Error from server (Forbidden)
```

Reason:

The ClusterRole contains permissions only for:

```
resources:
- nodes
```

Pods were never included.

---

Another example:

```bash
kubectl get namespaces
```

Output:

```
Error from server (Forbidden)
```

Again, permission was never granted.

This demonstrates the **Principle of Least Privilege**, where users receive only the permissions they explicitly require.

---

# Internal Authorization Flow

Whenever Vishal executes:

```bash
kubectl get nodes
```

Kubernetes performs the following sequence:

```
kubectl

      │

      ▼

Read kubeconfig

      │

      ▼

Current Context

vishal-context

      │

      ▼

User

vishal

      │

      ▼

Authentication

(Client Certificate)

      │

      ▼

API Server

      │

      ▼

Authorization (RBAC)

      │

      ▼

ClusterRoleBinding

node-reader-binding

      │

      ▼

ClusterRole

node-reader

      │

      ▼

Resource

Nodes

      │

      ▼

Verb

list

      │

      ▼

Permission Found

      │

      ▼

Request Allowed
```

---

# Complete RBAC Decision Flow

```
User Request

      │

      ▼

Authentication

      │

Certificate Valid?

      │

 ┌────┴────┐
 │         │
Yes        No
 │         │
 ▼         ▼

User      Reject

 │

 ▼

Authorization

 │

 ▼

ClusterRoleBinding

 │

 ▼

ClusterRole

 │

 ▼

Permission Exists?

 │

┌────┴────┐
│         │
Yes       No
│         │
▼         ▼

Allow   Forbidden
```

---

# Role vs ClusterRole

| Feature | Role | ClusterRole |
|---------|------|-------------|
| Scope | Namespace | Cluster |
| Can Access Nodes | ❌ No | ✅ Yes |
| Can Access Namespaces | ❌ No | ✅ Yes |
| Can Access Pods | ✅ Yes | ✅ Yes |
| Namespace Required | Yes | No |

---

# RoleBinding vs ClusterRoleBinding

| Feature | RoleBinding | ClusterRoleBinding |
|---------|-------------|--------------------|
| Scope | One Namespace | Entire Cluster |
| Can Bind Role | ✅ Yes | ❌ No |
| Can Bind ClusterRole | ✅ Yes | ✅ Yes |
| Grants Cluster-wide Access | ❌ No | ✅ Yes |

---

# Complete Combination Table

| RBAC Objects | Result |
|--------------|--------|
| Role + RoleBinding | Access only within one namespace |
| ClusterRole + RoleBinding | Uses ClusterRole permissions, but only inside one namespace |
| ClusterRole + ClusterRoleBinding | Access across the entire cluster |
| Role + ClusterRoleBinding | Invalid configuration |

---

# When Should We Use Each?

## Role + RoleBinding

Use when a user should access resources only inside a specific namespace.

Example:

```
Developer

↓

Namespace

dev
```

---

## ClusterRole + ClusterRoleBinding

Use when a user needs cluster-wide access.

Examples:

- Cluster Administrators
- Monitoring Tools
- Infrastructure Engineers
- Node Administrators

---

## ClusterRole + RoleBinding

This is very common in production.

A single ClusterRole is created once and reused across multiple namespaces using different RoleBindings.

Example:

```
ClusterRole

↓

pod-reader

↓

RoleBinding

↓

dev

RoleBinding

↓

qa

RoleBinding

↓

production
```

This avoids creating duplicate Roles in every namespace.

---

# Common Interview Questions

### 1. What is the difference between Role and ClusterRole?

A Role is namespace-scoped, whereas a ClusterRole is cluster-scoped and can also define permissions for namespace resources.

---

### 2. What is the difference between RoleBinding and ClusterRoleBinding?

A RoleBinding grants permissions within a single namespace, while a ClusterRoleBinding grants permissions across the entire cluster.

---

### 3. Can a RoleBinding reference a ClusterRole?

Yes.

A RoleBinding can bind either a Role or a ClusterRole.

---

### 4. Can a ClusterRoleBinding reference a Role?

No.

A ClusterRoleBinding can reference only a ClusterRole.

---

### 5. Can a ClusterRole contain Pod permissions?

Yes.

A ClusterRole can define permissions for both cluster resources and namespace resources.

The scope depends on the binding used.

---

### 6. Why did `kubectl get nodes` work after creating the ClusterRoleBinding?

Because the user **vishal** received the permissions defined in the **node-reader** ClusterRole through the **node-reader-binding** ClusterRoleBinding.

---

### 7. Why did `kubectl get pods -A` fail?

Because the ClusterRole only contained permissions for the **nodes** resource.

Pods were never included.

---

### 8. Why can't a Role be used with a ClusterRoleBinding?

Because a Role exists only inside a namespace, while a ClusterRoleBinding operates at the cluster level and can reference only ClusterRoles.

---

# Command Cheatsheet

## Check Current Context

```bash
kubectl config current-context
```

---

## Switch Context

```bash
kubectl config use-context vishal-context
```

---

## Create ClusterRole

```bash
kubectl apply -f clusterrole.yaml
```

---

## Verify ClusterRole

```bash
kubectl get clusterrole

kubectl describe clusterrole node-reader
```

---

## Create ClusterRoleBinding

```bash
kubectl apply -f clusterrolebinding.yaml
```

---

## Verify ClusterRoleBinding

```bash
kubectl get clusterrolebinding

kubectl describe clusterrolebinding node-reader-binding
```

---

## Test Node Access

```bash
kubectl get nodes
```

---

## Test Restricted Access

```bash
kubectl get pods -A

kubectl get namespaces
```

---

# Key Takeaways

- **ClusterRole** stores cluster-level permissions.
- **ClusterRoleBinding** assigns a ClusterRole to a User, Group, or ServiceAccount across the entire cluster.
- A **ClusterRole** can define permissions for both cluster resources and namespace resources.
- A **RoleBinding** can bind either a **Role** or a **ClusterRole**.
- A **ClusterRoleBinding** can bind only a **ClusterRole**.
- The **binding determines the scope** of the permissions.
- Kubernetes follows the **Principle of Least Privilege**, granting only the permissions explicitly defined in RBAC.
- Understanding the relationship between **Role**, **ClusterRole**, **RoleBinding**, and **ClusterRoleBinding** is fundamental for Kubernetes administration and is a frequently tested interview topic.

---

# Summary

In this chapter, we extended our understanding of Kubernetes RBAC from namespace-level authorization to cluster-wide authorization. We first distinguished namespace resources from cluster resources, then compared **Role**, **ClusterRole**, **RoleBinding**, and **ClusterRoleBinding** in terms of scope and usage. Through practical implementation, we created a **ClusterRole** named `node-reader`, bound it to the user **vishal** using a **ClusterRoleBinding**, and verified that the user could successfully access cluster-level resources such as Nodes while remaining restricted from unauthorized resources like Pods and Namespaces. Finally, we explored common RBAC combinations, internal authorization flow, interview questions, and best practices for designing secure Kubernetes access control.
