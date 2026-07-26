# Kubernetes Roles, RoleBindings & Client Certificate Authentication

---

# Role & RoleBinding

## The full form of RBAC is Role-Based Access Control.

## What Problem Does RBAC Solve?

Imagine a company with 500 employees.

Not everyone should have access to every department.

For example:

- HR employees should access employee records.
- Finance employees should access salary information.
- Developers should deploy applications.
- Visitors should not access confidential systems.

The same concept exists inside Kubernetes.

A Kubernetes cluster contains many resources such as:

- Pods
- Deployments
- Services
- Secrets
- ConfigMaps
- Nodes

If every user could access everything, anyone could accidentally delete applications, secrets, or even the entire cluster.

To solve this problem, Kubernetes uses **RBAC (Role-Based Access Control).**

---

# Real-Life Analogy

Imagine a hospital.

There are different people working inside the hospital.

| Person | Permissions |
|----------|-------------|
| Doctor | View patient records, prescribe medicines |
| Nurse | View patient records only |
| Receptionist | Register patients |
| Visitor | No medical access |

Notice something.

The hospital never gives permissions directly to every employee.

Instead, it creates **roles**.

Example:

Doctor Role

↓

Read Patient Records

↓

Write Prescriptions

Then employees are assigned that role.

Kubernetes works exactly the same way.

---

# What is RBAC?

RBAC stands for:

**Role-Based Access Control**

It is the authorization mechanism used by Kubernetes.

RBAC answers one question:

> **What is this authenticated user allowed to do?**

---

# Authentication vs Authorization

One of the most common interview questions.

Many beginners confuse these two concepts.

Authentication answers:

> **Who are you?**

Authorization answers:

> **What are you allowed to do?**

Example:

```text
Employee enters office

↓

Shows ID Card

↓

Security verifies identity

(Authentication)

↓

Manager checks department permissions

(Authorization)

↓

Access Granted
```

Kubernetes follows the same flow.

```text
kubectl

↓

API Server

↓

Authentication

↓

RBAC Authorization

↓

Allow / Deny
```

Authentication always happens **before** authorization.

---

# Kubernetes User

One surprising fact about Kubernetes is:

> **Kubernetes does NOT have a User resource.**

There is:

- Pod
- Deployment
- Service
- ConfigMap
- Secret

But there is NO:

```yaml
kind: User
```

Then where does the user come from?

The answer is:

Authentication systems.

Examples:

- Client Certificates
- AWS IAM
- LDAP
- Active Directory
- OIDC (Google, Azure AD, Okta, Keycloak)

After successful authentication, Kubernetes receives a username.

Example:

```text
Username

↓

abhi
```

RBAC then checks whether that username has permissions.

---

# Complete Authentication & Authorization Flow

```text
kubectl

↓

Authentication

↓

Username = abhi

↓

RoleBinding

↓

Role

↓

Permission Found?

↓

YES → Allowed

NO → Forbidden
```

---

# What is a Role?

A Role is simply a collection of permissions.

It defines:

- Which resource
- Which actions

inside a particular namespace.

Think of it as a permission template.

Example:

```
Pod Reader Role

Resources:
Pods

Permissions:

Get

List

Watch
```

Notice that the Role does NOT belong to any user.

It only defines permissions.

---

# Role Characteristics

- Namespace scoped
- Contains permissions
- Can be assigned to multiple users
- Does NOT contain usernames

---

# Role YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: my-role
  namespace: dev

rules:
- apiGroups: [""]
  resources:
  - pods

  verbs:
  - get
  - list
  - watch
  - create
  - delete
```

---

# Understanding Every Field

## apiVersion

Specifies which Kubernetes RBAC API version is being used.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
```

---

## kind

Defines the object type.

```yaml
kind: Role
```

---

## metadata

Stores information about the object.

```yaml
metadata:
  name: my-role
  namespace: dev
```

Role exists only inside the **dev** namespace.

---

## rules

Contains permission rules.

```yaml
rules:
```

One Role can contain multiple rules.

---

## apiGroups

Defines which API group the resource belongs to.

```yaml
apiGroups:
- ""
```

Empty string represents the **Core API Group**.

Examples:

- Pods
- Services
- ConfigMaps
- Secrets
- PersistentVolumeClaims

Example for Deployments:

```yaml
apiGroups:
- apps
```

---

## resources

Defines which Kubernetes resources the Role controls.

Example:

```yaml
resources:
- pods
```

You can specify multiple resources.

```yaml
resources:
- pods
- services
- configmaps
```

---

## verbs

Defines allowed operations.

Example:

```yaml
verbs:
- get
- list
- watch
- create
- delete
```

Common verbs:

| Verb | Meaning |
|-------|----------|
| get | Read one object |
| list | Read multiple objects |
| watch | Watch live changes |
| create | Create resource |
| update | Update resource |
| patch | Partial update |
| delete | Delete resource |

---

# What is RoleBinding?

A Role only defines permissions.

It does NOT give permissions to anyone.

RoleBinding connects:

**User**

↓

**Role**

Think of it as assigning a job to an employee.

---

# Real-Life Analogy

Company

↓

Creates "Developer Role"

↓

Employee joins company

↓

Manager assigns Developer Role

↓

Employee receives permissions

RoleBinding performs this assignment.

---

# RoleBinding YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: my-role-binding
  namespace: dev

subjects:
- kind: User
  name: abhi
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: my-role
  apiGroup: rbac.authorization.k8s.io
```

---

# Understanding Every Field

## subjects

Defines **who** receives permissions.

Example:

```yaml
subjects:
- kind: User
  name: abhi
```

This tells Kubernetes:

Assign permissions to user:

```
abhi
```

---

## roleRef

Defines **which Role** should be assigned.

```yaml
roleRef:
  kind: Role
  name: my-role
```

Notice something.

RoleBinding never contains permissions.

It only points to a Role.

---

# Why Doesn't RoleBinding Contain Permissions?

Imagine 100 developers.

If permissions were written inside every RoleBinding:

Developer 1

↓

Permissions

Developer 2

↓

Permissions

Developer 3

↓

Permissions

It would create unnecessary duplication.

Instead:

```
One Role

↓

100 RoleBindings

↓

100 Users
```

This makes RBAC easier to manage.

---

# Practical (Class Practical)

## Step 1 — Create Namespace

```bash
kubectl create namespace dev
```

Verify:

```bash
kubectl get ns
```

### Why?

Roles are namespace-scoped.

The namespace must exist before creating a Role.

---

## Step 2 — Create Role

Create `role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: my-role
  namespace: dev

rules:
- apiGroups: [""]
  resources:
  - pods

  verbs:
  - get
  - list
  - watch
  - create
  - delete
```

Apply:

```bash
kubectl apply -f role.yaml
```

Verify:

```bash
kubectl get role -n dev
```

Describe:

```bash
kubectl describe role my-role -n dev
```

### Why?

This creates a reusable permission template.

No user has permissions yet.

---

## Step 3 — Create RoleBinding

Create `rolebinding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: my-role-binding
  namespace: dev

subjects:
- kind: User
  name: abhi
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: my-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f rolebinding.yaml
```

Verify:

```bash
kubectl get rolebinding -n dev
```

Describe:

```bash
kubectl describe rolebinding my-role-binding -n dev
```

### Why?

RoleBinding connects:

```
User

↓

Role

↓

Permissions
```

---

## Step 4 — Test Permissions (Impersonation)

Check:

```bash
kubectl auth can-i get pods --as=abhi -n dev
```

Expected:

```text
yes
```

Check:

```bash
kubectl auth can-i delete pods --as=abhi -n dev
```

Expected:

```text
yes
```

Check:

```bash
kubectl auth can-i get deployments --as=abhi -n dev
```

Expected:

```text
no
```

### Why?

The Role only grants permissions for **Pods**.

It does not include **Deployments**.

---

# Important Note

The `--as` option **does not authenticate** as the user.

It only **impersonates** the specified username for an authorization check.

This is useful for testing RBAC but does not represent a real user login.

In the next section, we will create a real Kubernetes user using client certificates and authenticate as that user.

---

# Interview Questions

### Q1. What is RBAC?

RBAC (Role-Based Access Control) is Kubernetes' authorization mechanism that determines what an authenticated user is allowed to do.

---

### Q2. Does Kubernetes have a User object?

No.

Kubernetes authenticates users through external identity providers such as client certificates, IAM, LDAP, or OIDC. There is no native `User` resource.

---

### Q3. What is the difference between a Role and a RoleBinding?

Role defines permissions.

RoleBinding assigns those permissions to a user, group, or service account.

---

### Q4. Can a Role exist without a RoleBinding?

Yes.

It simply won't be used until it is bound to a subject.

---

### Q5. Does RoleBinding contain permissions?

No.

Permissions are defined only in the Role.

---

### Q6. Is a Role cluster-scoped?

No.

A Role is namespace-scoped.

---

# Summary

- RBAC is Kubernetes' authorization system.
- Authentication identifies the user.
- Authorization determines what the user can do.
- Kubernetes has no built-in User object.
- A Role defines permissions.
- A RoleBinding assigns a Role to a user, group, or service account.
- `kubectl auth can-i --as=<user>` is impersonation, not real authentication.
- The next section covers real client certificate authentication.

---

# Client Certificate Authentication

## Why Do We Need Another Practical?

In the previous practical, we used the following command to test RBAC:

```bash
kubectl auth can-i get pods --as=abhi -n dev
```

Many beginners think this command logs in as **abhi**.

It does **NOT**.

It only tells the API Server:

> "Pretend this request is coming from user `abhi` and tell me whether RBAC would allow it."

This process is called **Impersonation**.

No authentication happens.

No certificate is used.

No real user logs in.

This is useful for testing RBAC but **not** for understanding how Kubernetes authenticates users.

---

# What Happens in Production?

In a real Kubernetes cluster, a user first authenticates.

Only after successful authentication does Kubernetes check RBAC permissions.

The actual flow is:

```text
User

↓

Authentication

↓

API Server

↓

Authenticated Username

↓

RBAC

↓

Allow / Deny
```

Notice that RBAC never authenticates users.

RBAC only authorizes authenticated users.

---

# What is a Certificate?

A certificate is a **digital identity card**.

It proves the identity of a client.

Think of it like:

- Aadhaar Card
- Passport
- Employee ID Card

Suppose you enter an office.

The security guard asks:

> "Who are you?"

Instead of saying your name, you show your employee ID.

The guard verifies that the ID is genuine.

Kubernetes works exactly the same way.

Instead of showing an ID card, kubectl presents a **Client Certificate**.

---

# Certificate Authority (CA)

Anyone can create a certificate.

But Kubernetes only trusts certificates signed by a trusted authority.

This trusted authority is called the:

**Certificate Authority (CA)**

Real-life example:

Government

↓

Issues Passport

↓

Immigration trusts Passport

Similarly,

```text
Kubernetes CA

↓

Signs Certificate

↓

API Server trusts Certificate
```

If someone creates a fake certificate at home,

the API Server rejects it because it was not signed by the trusted CA.

---

# Client Certificate vs Server Certificate

There are two certificates involved in every Kubernetes connection.

## Server Certificate

Belongs to the API Server.

Purpose:

Proves that the API Server is genuine.

Without it,

someone could pretend to be your Kubernetes cluster.

---

## Client Certificate

Belongs to the Kubernetes user.

Purpose:

Proves the identity of the user.

Example:

```
CN = abhi
```

After successful verification,

the API Server authenticates the request as:

```
User = abhi
```

---

# Mutual TLS (mTLS)

During communication,

both sides verify each other.

```text
Client

↓

Client Certificate

↓

API Server

↓

Server Certificate

↓

Client
```

The client verifies the API Server.

The API Server verifies the client.

This is called:

**Mutual TLS (mTLS)**

---

# Authentication Flow

The complete authentication flow is:

```text
Private Key

↓

Certificate Signing Request (CSR)

↓

Certificate Authority

↓

Client Certificate

↓

kubectl

↓

API Server

↓

Authentication Successful
```

---

# Files Used During Authentication

| File | Purpose |
|--------|----------|
| abhi.key | Private Key |
| abhi.csr | Certificate Signing Request |
| abhi.crt | Signed Client Certificate |
| abhi.kubeconfig | kubectl configuration |

---

# Understanding Every File

## 1. abhi.key

This is the user's private key.

Generated using:

```bash
openssl genrsa -out abhi.key 2048
```

Verify:

```bash
ls -l abhi.key
```

### Why?

Every certificate has a key pair.

Private Key

↓

Public Key

The private key must never be shared.

It proves that you are the real owner of the certificate.

---

## 2. Create Certificate Signing Request

```bash
openssl req -new \
-key abhi.key \
-out abhi.csr \
-subj "/CN=abhi/O=developers"
```

Verify:

```bash
openssl req -text -noout -verify -in abhi.csr
```

Output:

```
CN = abhi

O = developers
```

### Understanding Every Field

### CN

Common Name

Represents the Kubernetes username.

Example:

```
CN = abhi
```

The API Server later authenticates this user as:

```
abhi
```

---

### O

Organization

Represents the Kubernetes group.

Example:

```
developers
```

Groups can also receive RBAC permissions.

---

### Why Create a CSR?

A CSR is simply a request asking the CA:

> "Please create a certificate for me."

The CSR itself is **not** a certificate.

---

# Kubernetes CSR API (Kops Practical)

Unlike manually signing certificates,

our Kops cluster used Kubernetes' built-in CSR API.

This is the recommended approach because:

- No direct access to the CA private key is required.
- Kubernetes manages certificate signing.
- It is safer than manually using the CA key.

---

## Step 1 — Encode CSR

```bash
cat abhi.csr | base64 | tr -d '\n'
```

### Why?

The CSR is binary data.

The Kubernetes YAML expects it in Base64 format.

---

## Step 2 — Create CSR YAML

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest

metadata:
  name: abhi

spec:
  request: <BASE64_CSR>

  signerName: kubernetes.io/kube-apiserver-client

  usages:
  - client auth
```

---

### Understanding Every Field

### signerName

```yaml
kubernetes.io/kube-apiserver-client
```

This tells Kubernetes:

> Create a certificate that can be used by Kubernetes clients.

---

### usages

```yaml
client auth
```

The certificate will be used for:

Client Authentication

---

## Step 3 — Create CSR

```bash
kubectl apply -f abhi-csr.yaml
```

Verify:

```bash
kubectl get csr
```

Expected:

```
Pending
```

### Why?

The certificate has not yet been approved.

---

## Step 4 — Approve CSR

```bash
kubectl certificate approve abhi
```

Verify:

```bash
kubectl get csr
```

Expected:

```
Approved,Issued
```

### Why?

The Kubernetes CA now signs the certificate.

---

## Step 5 — Download Certificate

```bash
kubectl get csr abhi \
-o jsonpath='{.status.certificate}' \
| base64 -d > abhi.crt
```

Verify:

```bash
openssl x509 -text -noout -in abhi.crt
```

Expected:

```
Issuer:

CN = kubernetes-ca

Subject:

CN = abhi

O = developers
```

### Why?

This confirms:

- Kubernetes CA signed the certificate.
- Username = abhi
- Group = developers

---

# Creating kubeconfig

First,

extract the cluster CA.

```bash
kubectl config view --raw \
-o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
| base64 -d > ca.crt
```

---

## Create Cluster

```bash
kubectl config set-cluster abhi-cluster \
--server=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}') \
--certificate-authority=ca.crt \
--embed-certs=true \
--kubeconfig=abhi.kubeconfig
```

---

## Create User

```bash
kubectl config set-credentials abhi \
--client-certificate=abhi.crt \
--client-key=abhi.key \
--embed-certs=true \
--kubeconfig=abhi.kubeconfig
```

---

## Create Context

```bash
kubectl config set-context abhi-context \
--cluster=abhi-cluster \
--user=abhi \
--namespace=default \
--kubeconfig=abhi.kubeconfig
```

---

## Use Context

```bash
kubectl config use-context abhi-context \
--kubeconfig=abhi.kubeconfig
```

Verify:

```bash
kubectl config get-contexts \
--kubeconfig=abhi.kubeconfig
```

---

# First Login

Now authenticate using:

```bash
kubectl --kubeconfig=abhi.kubeconfig get pods
```

Expected:

```
Error from server (Forbidden)

User "abhi"

cannot list resource "pods"
```

Many beginners think this is an authentication failure.

It is **NOT**.

Authentication succeeded.

Authorization failed.

---

# Why "Forbidden"?

This message actually proves that authentication worked.

The API Server successfully extracted:

```
CN = abhi
```

from the client certificate.

If authentication had failed,

the API Server would have returned:

```
Unauthorized
```

or

```
certificate signed by unknown authority
```

Instead,

it authenticated the user and then checked RBAC.

Since no RoleBinding existed,

the request was denied.

```text
Authentication

✔ Success

↓

RBAC

↓

No Permissions

↓

Forbidden
```

---

# Production Way (Without --kubeconfig)

In production,

users usually have their own Linux account.

Example:

```text
/root/.kube/config

/home/abhi/.kube/config

/home/rahul/.kube/config
```

Each user owns a different kubeconfig.

Create Linux user:

```bash
useradd -m abhi
```

Create Kubernetes configuration directory:

```bash
mkdir -p /home/abhi/.kube
```

Copy kubeconfig:

```bash
cp abhi.kubeconfig /home/abhi/.kube/config
```

Set ownership:

```bash
chown -R abhi:abhi /home/abhi/.kube
```

Login:

```bash
su - abhi
```

Verify:

```bash
whoami
```

Run:

```bash
kubectl get pods
```

No `--kubeconfig` option is required because kubectl automatically looks for:

```text
$HOME/.kube/config
```

---

# Complete Authentication Flow

```text
Generate Private Key

↓

Create CSR

↓

CSR Approval

↓

Kubernetes CA

↓

Signed Certificate

↓

Create kubeconfig

↓

kubectl

↓

API Server

↓

Authentication Successful

↓

RBAC

↓

Allow / Deny
```

---

# Interview Questions

### Q1. Does RBAC authenticate users?

No.

RBAC only authorizes authenticated users.

---

### Q2. What is a Client Certificate?

A client certificate is a digital identity used by Kubernetes to authenticate a user.

---

### Q3. What is the purpose of a CSR?

A CSR (Certificate Signing Request) asks a Certificate Authority to issue a certificate.

---

### Q4. What information identifies the Kubernetes user in a client certificate?

The **Common Name (CN)** identifies the username.

Example:

```
CN = abhi
```

---

### Q5. What happens if the certificate is not signed by a trusted CA?

The API Server rejects the connection and authentication fails.

---

### Q6. Why did we receive "Forbidden" instead of "Unauthorized"?

Because authentication succeeded.

RBAC authorization failed.

---

### Q7. Why doesn't production use `--kubeconfig` every time?

Because kubectl automatically uses:

```text
$HOME/.kube/config
```

for the currently logged-in Linux user.

---

# Summary

- Client certificates are used for Kubernetes authentication.
- Kubernetes trusts certificates signed by its Certificate Authority.
- A CSR requests a certificate from the CA.
- Kops can use the Kubernetes CSR API to issue client certificates.
- The CN in the certificate becomes the Kubernetes username.
- Authentication occurs before RBAC.
- "Forbidden" means authentication succeeded but RBAC denied access.
- In production, users typically store their kubeconfig in `$HOME/.kube/config` and use `kubectl` without additional flags.

---

# Complete Authentication & Authorization Architecture

After completing both practicals, we now understand the complete Kubernetes security workflow.

Every request made to Kubernetes follows the same sequence.

```text
kubectl get pods
        │
        ▼
kubectl reads kubeconfig
        │
        ▼
Loads Client Certificate & Private Key
        │
        ▼
Connects to API Server (TLS)
        │
        ▼
API Server sends Server Certificate
        │
        ▼
kubectl verifies API Server
        │
        ▼
kubectl proves ownership of Client Certificate
        │
        ▼
API Server authenticates user
        │
        ▼
Username extracted from Certificate
        │
        ▼
RBAC Authorization
        │
        ▼
Admission Controllers
        │
        ▼
API Processing
        │
        ▼
etcd
```

Notice that RBAC is only **one small part** of the entire request lifecycle.

---

# Complete Request Lifecycle

Suppose a user executes:

```bash
kubectl get pods
```

The complete flow is:

```text
kubectl

↓

Read kubeconfig

↓

Load Client Certificate

↓

TLS Handshake

↓

Verify API Server Certificate

↓

API Server verifies Client Certificate

↓

Authentication

↓

Username = abhi

↓

RBAC

↓

Allowed?

↓

Admission Controllers

↓

API Processing

↓

etcd

↓

Response returned to kubectl
```

Every Kubernetes request follows this sequence.

---

# Authentication vs Authorization

This is one of the most frequently asked interview questions.

| Authentication | Authorization |
|----------------|---------------|
| Verifies identity | Verifies permissions |
| "Who are you?" | "What can you do?" |
| Happens first | Happens second |
| Uses Certificates, IAM, LDAP, OIDC | Uses RBAC |

Remember:

Authentication and Authorization are completely separate processes.

---

# Authentication Methods in Kubernetes

Kubernetes supports multiple authentication methods.

## 1. Client Certificates

Used in:

- kubeadm
- Kops
- Self-managed clusters

Example:

```
Certificate

↓

API Server

↓

Authenticated
```

---

## 2. AWS IAM

Used in:

Amazon EKS

Flow:

```text
kubectl

↓

AWS CLI

↓

Temporary IAM Token

↓

API Server

↓

Authenticated
```

Notice:

EKS normally uses IAM tokens instead of client certificates for users.

---

## 3. LDAP / Active Directory

Large organizations usually authenticate users using corporate directories.

Example:

```
Employee Login

↓

LDAP

↓

API Server

↓

Authenticated
```

---

## 4. OIDC

Examples:

- Google
- Azure AD
- Okta
- Keycloak

Flow:

```
Google Login

↓

OIDC

↓

API Server

↓

Authenticated
```

---

# Kops vs kubeadm vs EKS

| Feature | kubeadm | Kops | EKS |
|----------|----------|------|-----|
| Cluster Type | Self-managed | Self-managed | Managed |
| Authentication | Client Certificates | Client Certificates / CSR API | AWS IAM |
| CA Managed By | Administrator | Kops | AWS |
| Worker Management | Manual | Kops | AWS |
| Control Plane | Self-managed | Self-managed | AWS |

---

# Client Certificate vs IAM Authentication

| Client Certificate | IAM |
|-------------------|-----|
| Uses Certificates | Uses IAM Tokens |
| Common in kubeadm & Kops | Common in EKS |
| Certificate identifies user | IAM identity identifies user |
| No AWS dependency | Requires AWS IAM |

---

# Client Certificate vs ServiceAccount

Many beginners confuse these.

They are completely different.

| Client Certificate | ServiceAccount |
|-------------------|----------------|
| Represents Human User | Represents Application |
| Used by kubectl | Used by Pods |
| Authentication Method | Pod Identity |
| Example: abhi | Example: default ServiceAccount |

Remember:

Users normally authenticate using certificates or IAM.

Pods authenticate using ServiceAccounts.

---

# Why Doesn't Kubernetes Have a User Object?

Another common interview question.

Kubernetes intentionally does not create User resources.

Instead,

it trusts external authentication providers.

Examples:

- Certificates
- IAM
- LDAP
- OIDC

After authentication,

Kubernetes simply receives:

```
Username

↓

abhi
```

RBAC then checks permissions for that username.

---

# Why Does RoleBinding Use User?

RoleBinding simply matches usernames.

Example:

```yaml
subjects:

- kind: User

  name: abhi
```

RBAC does not create the user.

It only says:

> If an authenticated request belongs to user **abhi**, apply this Role.

---

# Authentication Failure vs Authorization Failure

Authentication Failure

Example:

```
Unauthorized
```

Reasons:

- Invalid certificate
- Expired certificate
- Certificate signed by unknown CA
- Invalid IAM token

Authentication never completed.

---

Authorization Failure

Example:

```
Forbidden

User "abhi"

cannot list resource "pods"
```

Authentication succeeded.

RBAC denied permission.

This distinction is extremely important.

---

# Common Beginner Mistakes

## Mistake 1

Thinking:

```
Role

↓

User
```

Correct:

```
Role

↓

RoleBinding

↓

User
```

---

## Mistake 2

Thinking:

```
RoleBinding contains permissions
```

Wrong.

Permissions belong only inside Roles.

---

## Mistake 3

Thinking:

```
kubectl auth can-i --as=abhi
```

logs in as abhi.

Wrong.

It only impersonates the username for authorization testing.

---

## Mistake 4

Thinking:

```
Forbidden

↓

Authentication failed
```

Wrong.

Forbidden means authentication succeeded.

---

## Mistake 5

Thinking Linux User = Kubernetes User

Example:

```
Linux User

↓

abhi
```

and

```
Certificate CN

↓

abhi
```

They happen to have the same name in our lab.

But Kubernetes only trusts the identity presented by the client certificate (or another authentication method). The Linux account itself is not used for authentication.

---

# Real Production Workflow

```text
Developer

↓

SSH

↓

Linux User

↓

~/.kube/config

↓

kubectl

↓

Client Certificate / IAM Token

↓

API Server

↓

Authentication

↓

RBAC

↓

Allowed

↓

Cluster Resources
```

---

# What We Learned in This Practical

We created a complete authentication system.

```text
Generate Private Key

↓

Generate CSR

↓

Approve CSR

↓

Download Client Certificate

↓

Create kubeconfig

↓

Authenticate

↓

Create Role

↓

Create RoleBinding

↓

Authorization

↓

Access Granted
```

This practical demonstrated both authentication and authorization working together.

---

# Frequently Asked Interview Questions

### 1. What is RBAC?

Role-Based Access Control is Kubernetes' authorization mechanism that determines what an authenticated user is allowed to do.

---

### 2. Does Kubernetes have a User resource?

No.

Users are authenticated through external identity providers.

---

### 3. What is the difference between Role and ClusterRole?

Role works only inside one namespace.

ClusterRole works across the entire cluster.

---

### 4. What is the difference between RoleBinding and ClusterRoleBinding?

RoleBinding grants permissions within a namespace.

ClusterRoleBinding grants permissions across the cluster.

---

### 5. What is the purpose of a Client Certificate?

To authenticate a user to the Kubernetes API Server.

---

### 6. What is a Certificate Authority?

A trusted authority that signs certificates.

---

### 7. What is a CSR?

A Certificate Signing Request requesting the CA to issue a certificate.

---

### 8. Why is the Private Key important?

It proves ownership of the certificate.

It must never be shared.

---

### 9. Why did we receive "Forbidden"?

Authentication succeeded.

RBAC denied access.

---

### 10. Why didn't we receive "Unauthorized"?

Because the client certificate was successfully authenticated.

---

### 11. Which field becomes the Kubernetes username?

The Common Name (CN).

Example:

```
CN = abhi
```

---

### 12. Which field becomes the Kubernetes group?

The Organization (O).

Example:

```
O = developers
```

---

### 13. Can a Role exist without a RoleBinding?

Yes.

It simply isn't assigned to anyone.

---

### 14. Can one Role be assigned to multiple users?

Yes.

Multiple RoleBindings can reference the same Role.

---

### 15. Can one user have multiple Roles?

Yes.

A user can be bound to multiple Roles using multiple RoleBindings.

---

# Final Summary

In this chapter, we learned:

- Why Kubernetes uses RBAC.
- The difference between Authentication and Authorization.
- Why Kubernetes has no native User object.
- How Roles define permissions.
- How RoleBindings assign permissions.
- How client certificate authentication works.
- How Kubernetes issues certificates using the CSR API.
- How kubeconfig stores authentication information.
- Why "Forbidden" means authentication succeeded.
- The complete Kubernetes request lifecycle.
- The difference between Kops, kubeadm, and EKS authentication.
- Common interview questions and production concepts.

By completing this practical, you now understand the full security flow of a Kubernetes request—from a user running `kubectl`, through authentication, authorization, admission control, and finally to the API Server and etcd.
