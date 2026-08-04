# 20-Kubernetes-RBAC-Role-RoleBinding-CertificateAuthentication.md

# Kubernetes RBAC (Role & RoleBinding) with Certificate Authentication

## What is RBAC?

**RBAC (Role-Based Access Control)** is the authorization mechanism used by Kubernetes to control **who can perform what action on which resource**.

In a Kubernetes cluster, not every user should have full administrative access. Different users, teams, and applications require different permissions.

RBAC helps implement the **Principle of Least Privilege**, meaning every user receives only the permissions required to perform their job.

---

## Why Do We Need RBAC?

Imagine an organization with multiple teams:

- Developers
- QA Engineers
- DevOps Engineers
- Security Team
- Database Administrators

If everyone has full cluster access:

- Developers may accidentally delete production Pods.
- QA engineers may modify Deployments.
- Applications could access sensitive resources.
- Security risks increase significantly.

Instead, Kubernetes allows us to assign only the required permissions to each user.

For example:

| User | Permission |
|------|------------|
| Developer | View Pods, Deploy Applications |
| QA Engineer | View Pods and Logs |
| Database Admin | Manage Database Pods |
| DevOps Engineer | Full Cluster Access |

This permission management is handled using RBAC.

---

# Authentication vs Authorization

One of the most important concepts in Kubernetes is understanding the difference between **Authentication** and **Authorization**.

Many beginners confuse these two concepts, but they solve completely different problems.

---

## Authentication (Who are you?)

Authentication verifies the identity of the user trying to access the Kubernetes API Server.

Think of it as showing your ID card before entering an office building.

Examples of Authentication methods:

- Client Certificates
- Service Accounts
- OpenID Connect (OIDC)
- Bearer Tokens
- Cloud IAM Authentication (AWS IAM, Azure AD, GCP IAM)

If authentication succeeds, Kubernetes now knows **who you are**.

Example:

```
User = vishal
```

Authentication **does not** decide what the user is allowed to do.

It only verifies identity.

---

## Authorization (What are you allowed to do?)

After authentication succeeds, Kubernetes checks whether the authenticated user has permission to perform the requested action.

This is called **Authorization**.

RBAC is one of Kubernetes' authorization mechanisms.

Example:

Authenticated User:

```
vishal
```

Requested Action:

```
kubectl get pods
```

RBAC checks:

- Does Vishal have permission?
- Which Role is assigned?
- Does that Role allow listing Pods?

If yes:

```
Request Allowed
```

Otherwise:

```
Forbidden
```

---

## Real World Example

Imagine entering a company office.

### Step 1 – Authentication

Security Guard asks:

> Who are you?

You show your Employee ID.

The guard verifies your identity.

```
Identity Verified
```

---

### Step 2 – Authorization

Now the company checks your job role.

Suppose your designation is:

```
Software Developer
```

Can you enter the Server Room?

```
No
```

Can you enter the Development Floor?

```
Yes
```

The company isn't checking your identity anymore.

It is checking your **permissions**.

Exactly the same happens inside Kubernetes.

---

# Kubernetes Authentication Flow

Every request sent to Kubernetes follows this flow:

```
                kubectl

                   │

                   ▼

           Kubernetes API Server

                   │

                   ▼

          Authentication Phase

        "Who is making this request?"

                   │

         Identity Verified?

          Yes                No
           │                 │
           ▼                 ▼
     Authorization       Request Rejected

           │

           ▼

      RBAC Checks Permissions

           │

      Allowed?      Denied?

         │              │
         ▼              ▼

   Execute Request   Forbidden
```

---

# Certificate-Based Authentication

In this chapter, we will authenticate a new Kubernetes user using **Client Certificates**.

Instead of using a Linux user or an AWS IAM user, Kubernetes will identify the user based on a certificate signed by the Kubernetes Certificate Authority (CA).

We will create a new user named:

```
vishal
```

using the following process:

```
Private Key

      │

      ▼

Certificate Signing Request (CSR)

      │

      ▼

Signed by Kubernetes CA

      │

      ▼

Client Certificate

      │

      ▼

Add User to kubeconfig

      │

      ▼

Authenticate to Kubernetes
```

---

# Components Used in Certificate Authentication

Before performing the practical, let's understand the files involved.

---

## 1. Private Key (.key)

A Private Key is generated first.

Example:

```
vishal.key
```

It proves ownership of the certificate.

This file must always remain secret.

Never share it with anyone.

---

## 2. Certificate Signing Request (CSR)

A CSR contains the information that we want inside the certificate.

Example:

```
Common Name (CN)

Organization (O)
```

Example values:

```
CN = vishal

O = developers
```

The CSR is then sent to the Kubernetes Certificate Authority for signing.

---

## 3. Kubernetes Certificate Authority (CA)

Every Kubernetes cluster has its own Certificate Authority.

The CA is responsible for signing certificates.

Only certificates signed by this CA are trusted by the Kubernetes API Server.

If the certificate is signed by another CA, authentication will fail.

---

## 4. Client Certificate (.crt)

After the CSR is signed by the Kubernetes CA, a Client Certificate is generated.

Example:

```
vishal.crt
```

The API Server uses this certificate to identify the user.

---

# Important Files Used in This Practical

| File | Purpose |
|------|---------|
| vishal.key | User's Private Key |
| vishal.csr | Certificate Signing Request |
| ca.key | Kubernetes CA Private Key |
| ca.crt | Kubernetes CA Certificate |
| vishal.crt | Signed Client Certificate |

---

# Authentication Flow for This Practical

```
Generate Private Key

        │

        ▼

Generate CSR

        │

        ▼

Download Kubernetes CA

        │

        ▼

Extract CA Key

        │

        ▼

Extract CA Certificate

        │

        ▼

Sign CSR

        │

        ▼

Generate Client Certificate

        │

        ▼

Configure kubeconfig User

        │

        ▼

Authenticate as Vishal

        │

        ▼

RBAC Authorization

        │

        ▼

Allow / Deny Request
```

---

# What We Will Build in This Practical

By the end of this lab, we will have:

- Generated a Private Key
- Created a Certificate Signing Request (CSR)
- Signed the CSR using the Kubernetes CA
- Generated a Client Certificate
- Added a new Kubernetes user to kubeconfig
- Created a dedicated context for the new user
- Switched to the new user
- Observed permission denied (Forbidden)
- Created a Role
- Created a RoleBinding
- Granted permissions to the user
- Verified RBAC by testing allowed and denied operations

# Practical Lab - Certificate-Based Authentication

In this practical, we will create a new Kubernetes user named **vishal** using **Certificate-Based Authentication**.

Unlike Linux authentication or AWS IAM authentication, Kubernetes will authenticate this user using a **Client Certificate** signed by the Kubernetes Certificate Authority (CA).

After creating the user, we will add it to **kubeconfig**, create a separate context, and later assign permissions using RBAC.

---

# Lab Architecture

```
                Kubernetes Cluster

          +---------------------------+
          |      Kubernetes CA        |
          +---------------------------+
                     │
             Signs User Certificate
                     │
                     ▼
             +----------------+
             |  vishal.crt    |
             +----------------+
                     │
                     ▼
          kubeconfig User Entry
                     │
                     ▼
          Authentication Success
                     │
                     ▼
         RBAC checks permissions
```

---

# Step 1 - Verify Current Kubernetes Context

Before creating a new user, verify which Kubernetes cluster and context you are currently using.

### Check Current Context

```bash
kubectl config current-context
```

Example Output

```text
abhi.k8s.local
```

---

### List All Available Contexts

```bash
kubectl config get-contexts
```

Example Output

```text
CURRENT   NAME             CLUSTER          AUTHINFO
*         abhi.k8s.local   abhi.k8s.local   abhi.k8s.local
```

---

## Why Are We Checking the Context?

The **context** determines:

- Which Kubernetes Cluster to connect to.
- Which User credentials to use.
- Which Namespace to use by default.

If we create certificates for the wrong cluster, authentication will fail.

---

# Step 2 - Generate a Private Key

The first step is generating a private key for the new user.

Run:

```bash
openssl genrsa -out vishal.key 2048
```

Verify:

```bash
ls -lh
```

Example Output

```text
vishal.key
```

---

## Understanding the Command

```bash
openssl
```

OpenSSL utility for cryptographic operations.

---

```bash
genrsa
```

Generate an RSA private key.

---

```bash
-out vishal.key
```

Save the generated key into:

```
vishal.key
```

---

```bash
2048
```

Generate a **2048-bit RSA key**.

This is the commonly used secure key size.

---

## What is Inside the Private Key?

The file contains encrypted mathematical values used to prove ownership of the certificate.

Example:

```
-----BEGIN PRIVATE KEY-----

....

-----END PRIVATE KEY-----
```

Never edit or share this file.

---

## Why is the Private Key Important?

Think of the private key as your personal signature.

Only the owner of the private key can successfully authenticate using the certificate signed from it.

Without the matching private key:

```
Certificate Authentication

❌ Fails
```

---

# Step 3 - Generate a Certificate Signing Request (CSR)

Now create a CSR.

Run:

```bash
openssl req -new \
-key vishal.key \
-out vishal.csr \
-subj "/CN=vishal/O=developers"
```

Verify:

```bash
ls -lh
```

Example Output

```text
vishal.key
vishal.csr
```

---

## Understanding the Command

```bash
req
```

Creates a Certificate Signing Request.

---

```bash
-new
```

Generate a new CSR.

---

```bash
-key vishal.key
```

Use the previously generated private key.

---

```bash
-out vishal.csr
```

Save the request into:

```
vishal.csr
```

---

```bash
-subj
```

Provide subject information without interactive prompts.

---

```bash
CN=vishal
```

**CN (Common Name)** becomes the Kubernetes username after authentication.

Later, RBAC will identify this user as:

```
vishal
```

---

```bash
O=developers
```

Organization.

This can later be used for **Group-Based RBAC**.

In this practical, we are not using group permissions, but Kubernetes stores this information in the certificate.

---

# Understanding the CSR

The CSR does **not** authenticate the user.

It is simply a request asking the Kubernetes CA to issue a certificate.

It contains:

- User Identity
- Public Key
- Organization

The CA verifies and signs it.

---

# Step 4 - Download the Kubernetes CA Information

Every Kubernetes cluster created using **kOps** stores its Certificate Authority information inside the **State Store (Amazon S3)**.

Download the keyset file.

```bash
aws s3 cp \
s3://<YOUR-KOPS-STATE-STORE>/<CLUSTER-NAME>/pki/private/kubernetes-ca/keyset.yaml .
```

Example

```bash
aws s3 cp \
s3://abhis.kops.v1/abhi.k8s.local/pki/private/kubernetes-ca/keyset.yaml .
```

Verify:

```bash
ls
```

Example

```text
keyset.yaml
```

---

## What is keyset.yaml?

This file contains the Kubernetes CA material encoded in **Base64**.

It includes:

- CA Private Key
- CA Public Certificate

These values must be decoded before they can be used.

---

# Step 5 - Extract the Kubernetes CA Private Key

Run:

```bash
grep privateMaterial keyset.yaml \
| awk '{print $2}' \
| base64 -d > ca.key
```

Verify:

```bash
ls -lh ca.key
```

Example Output

```text
ca.key
```

---

## Understanding the Command

```bash
grep privateMaterial
```

Find the Base64 encoded private key.

---

```bash
awk '{print $2}'
```

Extract only the encoded value.

---

```bash
base64 -d
```

Decode the Base64 content.

---

```bash
> ca.key
```

Store the decoded key in:

```
ca.key
```

---

## Why Do We Need the Private Key?

A Certificate Authority signs certificates using its **private key**.

Without this key:

```
Signing

❌ Impossible
```

This file must always remain secret.

---

# Step 6 - Extract the Kubernetes CA Certificate

Run:

```bash
grep publicMaterial keyset.yaml \
| awk '{print $2}' \
| base64 -d > ca.crt
```

Verify:

```bash
ls -lh ca.crt
```

Example Output

```text
ca.crt
```

---

## Why publicMaterial?

The Kubernetes CA certificate is public.

Clients use it to verify signatures.

Unlike the private key, it can be safely shared.

---

# Step 7 - Sign the CSR

Now ask the Kubernetes CA to sign the request.

Run:

```bash
openssl x509 -req \
-in vishal.csr \
-CA ca.crt \
-CAkey ca.key \
-CAcreateserial \
-out vishal.crt \
-days 365
```

Example Output

```text
Certificate request self-signature ok
subject=CN=vishal, O=developers
```

Verify:

```bash
ls -lh vishal.crt
```

---

## Understanding the Command

```bash
x509
```

Creates an X.509 certificate.

---

```bash
-req
```

Input is a CSR.

---

```bash
-in vishal.csr
```

Use the CSR created earlier.

---

```bash
-CA ca.crt
```

Use the Kubernetes CA certificate.

---

```bash
-CAkey ca.key
```

Use the Kubernetes CA private key.

---

```bash
-CAcreateserial
```

Create a serial number for the issued certificate.

---

```bash
-out vishal.crt
```

Generate:

```
vishal.crt
```

---

```bash
-days 365
```

Certificate validity:

```
365 Days
```

---

# Files Created So Far

```
vishal.key
```

Private key owned by the user.

---

```
vishal.csr
```

Certificate Signing Request.

---

```
ca.key
```

Kubernetes CA Private Key.

---

```
ca.crt
```

Kubernetes CA Certificate.

---

```
vishal.crt
```

Signed client certificate.

---

# Authentication Flow Completed So Far

```
Generate Private Key

        │

        ▼

Generate CSR

        │

        ▼

Download Kubernetes CA

        │

        ▼

Extract CA Key

        │

        ▼

Extract CA Certificate

        │

        ▼

Sign CSR

        │

        ▼

Generate Client Certificate ✅
```

> At this point, we have successfully created a valid Kubernetes client certificate. In the next part, we will add this certificate to **kubeconfig**, create a new Kubernetes user, create a dedicated context, switch to that user, and observe the initial **Forbidden** error before configuring RBAC.

# Practical Lab - Configure kubeconfig User & Context

In the previous section, we successfully generated a valid Kubernetes client certificate.

At this point we have:

```
vishal.key
```

Private Key

```
vishal.csr
```

Certificate Signing Request

```
vishal.crt
```

Signed Client Certificate

Now Kubernetes must be informed that these credentials belong to a new user.

This is done by configuring **kubeconfig**.

---

# What is kubeconfig?

`kubeconfig` is a configuration file used by **kubectl** to connect to Kubernetes clusters.

By default it is stored at:

```
~/.kube/config
```

It stores three important things:

- Clusters
- Users
- Contexts

Think of it as a phonebook that tells kubectl:

- Which cluster should I connect to?
- Which user credentials should I use?
- Which namespace should I use?

---

# Understanding kubeconfig Structure

A kubeconfig file contains three major sections:

```
Clusters
    │
    ▼
Users
    │
    ▼
Contexts
```

Example:

```
Clusters

abhi.k8s.local

Users

abhi.k8s.local
vishal

Contexts

abhi.k8s.local
vishal-context
```

---

# Step 8 - Add the New User to kubeconfig

Now register the newly created certificate as a Kubernetes user.

Run:

```bash
kubectl config set-credentials vishal \
  --client-certificate=/root/vishal.crt \
  --client-key=/root/vishal.key
```

Example Output

```
User "vishal" set.
```

---

## Understanding the Command

```
kubectl config
```

Modify kubeconfig.

---

```
set-credentials
```

Create or update a user entry.

---

```
vishal
```

This is the name of the Kubernetes user.

Notice:

This user **does not exist in Linux**.

This user **does not exist in AWS IAM**.

It only exists inside Kubernetes authentication.

---

```
--client-certificate
```

Specify the client certificate.

```
vishal.crt
```

---

```
--client-key
```

Specify the matching private key.

```
vishal.key
```

Both files are required during authentication.

---

# Verify the User

Run:

```bash
kubectl config get-users
```

Example Output

```
NAME
abhi.k8s.local
vishal
```

---

## What Just Happened?

We did **not** create a Linux user.

We only created a kubeconfig entry.

Before:

```
Users

abhi.k8s.local
```

After:

```
Users

abhi.k8s.local
vishal
```

---

# Step 9 - Create a Namespace

We want Vishal to work only inside a dedicated namespace.

Create it.

```bash
kubectl create namespace dev
```

Verify:

```bash
kubectl get namespaces
```

Example

```
default

dev

kube-system
```

---

## Why Create a Namespace?

RBAC is commonly applied inside namespaces.

Instead of giving access to the entire cluster, we give access only to:

```
dev
```

This improves security.

---

# Step 10 - Create a Context

Now create a new context for Vishal.

Run:

```bash
kubectl config set-context vishal-context \
  --cluster=abhi.k8s.local \
  --user=vishal \
  --namespace=dev
```

Example Output

```
Context "vishal-context" created.
```

---

## Understanding the Command

```
set-context
```

Create a new context.

---

```
vishal-context
```

Context Name.

---

```
--cluster
```

Cluster to connect.

```
abhi.k8s.local
```

---

```
--user
```

Use:

```
vishal
```

for authentication.

---

```
--namespace
```

Default namespace:

```
dev
```

---

# Verify the Context

Run:

```bash
kubectl config get-contexts
```

Example

```
CURRENT   NAME             CLUSTER          AUTHINFO
*         abhi.k8s.local   abhi.k8s.local   abhi.k8s.local

          vishal-context   abhi.k8s.local   vishal
```

Notice:

Current context is still:

```
abhi.k8s.local
```

---

# Step 11 - Switch to the New User

Now become the new Kubernetes user.

Run:

```bash
kubectl config use-context vishal-context
```

Example Output

```
Switched to context "vishal-context"
```

Verify:

```bash
kubectl config current-context
```

Output

```
vishal-context
```

---

# What Changed?

Before

```
User

abhi.k8s.local
```

After

```
User

vishal
```

Kubectl is now sending every request using:

```
vishal.crt

+

vishal.key
```

The Kubernetes API Server authenticates the request as:

```
User = vishal
```

---

# Step 12 - Test Authentication

Run:

```bash
kubectl get pods
```

Expected Output

```
Error from server (Forbidden)
```

Example

```
pods is forbidden:
User "vishal" cannot list resource "pods"
```

---

# Why Did Authentication Succeed but Access Was Denied?

Many beginners think the certificate is incorrect.

Actually, the certificate is working perfectly.

Let's understand the sequence.

### Step 1

Kubectl sends:

```
vishal.crt

+

vishal.key
```

↓

### Step 2

API Server verifies:

```
Certificate signed by Kubernetes CA?

YES
```

↓

### Step 3

API Server identifies:

```
User = vishal
```

↓

### Step 4

RBAC checks permissions.

Question:

```
Does user vishal have permission
to list Pods?
```

Answer:

```
NO
```

↓

Result:

```
Forbidden
```

---

# Authentication vs Authorization

Authentication

```
Who are you?

↓

Vishal

SUCCESS
```

Authorization

```
What are you allowed to do?

↓

Nothing

↓

Forbidden
```

This proves an important concept:

> **Authentication and Authorization are two completely separate processes.**

Our certificate successfully authenticated the user, but because we have not yet created a **Role** or **RoleBinding**, Kubernetes denies every request.

---

# Current Authentication Flow

```
kubectl

      │

      ▼

vishal.crt

+

vishal.key

      │

      ▼

API Server

      │

Authentication

      │

User = vishal

      │

Authorization (RBAC)

      │

No RoleBinding Found

      │

Forbidden
```

---

# What Have We Achieved?

By this point we have successfully:

- Generated a Private Key
- Created a CSR
- Signed the CSR using the Kubernetes CA
- Generated a Client Certificate
- Added a new Kubernetes user to kubeconfig
- Created a dedicated namespace
- Created a dedicated context
- Switched to the new user
- Successfully authenticated using the client certificate
- Verified that the user has **no permissions** by default

> In the next section, we will create a **Role** and **RoleBinding** to grant controlled permissions to the `vishal` user, allowing access only to the resources we explicitly permit.

# Practical Lab - Role & RoleBinding (Authorization)

In the previous section, we successfully authenticated the user **vishal** using a Client Certificate.

However, when we executed:

```bash
kubectl get pods
```

Kubernetes returned:

```text
Error from server (Forbidden)
```

This proves an important point:

> **Authentication only verifies identity. Authorization decides permissions.**

Now we will authorize the authenticated user using **Role** and **RoleBinding**.

---

# What is a Role?

A **Role** is an RBAC object that defines **what actions can be performed on which resources inside a particular namespace**.

A Role **does not belong to any user**.

It only stores permissions.

Think of it like a **Job Description**.

Example:

```
Job Title

↓

Pod Reader

↓

Responsibilities

✔ View Pods

✔ List Pods

✔ Watch Pods
```

Notice that the job description does not mention **who** is doing the job.

Exactly the same applies to a Kubernetes Role.

---

# Role Characteristics

A Role is always limited to a **single namespace**.

It can define permissions for resources such as:

- Pods
- Services
- ConfigMaps
- Secrets
- Deployments
- PVCs

Example:

```
Namespace

↓

dev

↓

Role

↓

Read Pods
```

The Role cannot automatically grant access outside the namespace where it exists.

---

# Role Architecture

```
Namespace

dev

      │

      ▼

+--------------------+

Role

pod-reader

+--------------------+

Resources

Pods

Verbs

get
list
watch
```

---

# Step 13 - Switch Back to Admin Context

Currently we are logged in as:

```
vishal
```

Since Vishal has no permissions yet, we must switch back to the administrator context.

Run:

```bash
kubectl config use-context abhi.k8s.local
```

Verify:

```bash
kubectl config current-context
```

Output

```
abhi.k8s.local
```

---

## Why?

Only an administrator has permission to create RBAC resources.

Users cannot grant permissions to themselves.

---

# Step 14 - Create a Role

Create a file:

```bash
vim role.yaml
```

Paste the following YAML.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: pod-reader
  namespace: dev

rules:
- apiGroups:
  - ""

  resources:
  - pods

  verbs:
  - get
  - list
  - watch
```

Apply it.

```bash
kubectl apply -f role.yaml
```

Verify:

```bash
kubectl get role -n dev
```

Describe:

```bash
kubectl describe role pod-reader -n dev
```

---

# Understanding Every Field

## apiVersion

```yaml
apiVersion: rbac.authorization.k8s.io/v1
```

RBAC resources belong to the RBAC API group.

---

## kind

```yaml
kind: Role
```

Creates a namespace-scoped Role.

---

## metadata

```yaml
metadata:
```

Contains object information.

---

## name

```yaml
name: pod-reader
```

Role name.

This can be any meaningful name.

---

## namespace

```yaml
namespace: dev
```

This Role exists only inside the **dev** namespace.

---

## rules

```yaml
rules:
```

Rules define the permissions granted by the Role.

---

## apiGroups

```yaml
apiGroups:
- ""
```

Pods belong to the **Core API Group**.

The Core API Group is represented by an empty string.

Examples:

| Resource | API Group |
|-----------|-----------|
| Pods | "" |
| Services | "" |
| ConfigMaps | "" |
| Secrets | "" |

---

## resources

```yaml
resources:
- pods
```

The Role controls access only to Pods.

No permissions are granted for:

- Services
- Deployments
- ConfigMaps
- Secrets

---

## verbs

```yaml
verbs:
- get
- list
- watch
```

These are the operations allowed.

### get

Read a single Pod.

Example

```bash
kubectl get pod test
```

---

### list

List multiple Pods.

Example

```bash
kubectl get pods
```

---

### watch

Continuously watch for changes.

Example

```bash
kubectl get pods -w
```

---

# Current Situation

At this point Kubernetes has:

```
Role

↓

pod-reader

↓

Permissions

✔ get Pods

✔ list Pods

✔ watch Pods
```

But there is still one problem.

**Nobody is using this Role.**

---

# Why Doesn't the Role Work Yet?

Creating a Role only defines permissions.

It does **not** assign those permissions to any user.

Think about a company.

The HR department creates a new job position:

```
Pod Reader
```

Does that automatically mean someone is hired?

```
No
```

The company must assign an employee to that role.

Kubernetes follows the same principle.

---

# What is a RoleBinding?

A **RoleBinding** connects a **User**, **Group**, or **ServiceAccount** to a Role.

Think of it as an appointment letter.

```
Role

↓

Pod Reader

↓

Assigned To

↓

Vishal
```

Without a RoleBinding:

```
Role

↓

Nobody

↓

No Permissions
```

---

# Step 15 - Create the RoleBinding

Create a file:

```bash
vim rolebinding.yaml
```

Paste:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: pod-reader-binding
  namespace: dev

subjects:
- kind: User
  name: vishal
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply it.

```bash
kubectl apply -f rolebinding.yaml
```

Verify:

```bash
kubectl get rolebinding -n dev
```

Describe:

```bash
kubectl describe rolebinding pod-reader-binding -n dev
```

---

# Understanding Every Field

## subjects

```yaml
subjects:
```

Defines **who** receives the permissions.

---

## kind

```yaml
kind: User
```

Assign permissions to a Kubernetes User.

RBAC also supports:

- User
- Group
- ServiceAccount

---

## name

```yaml
name: vishal
```

This must match the authenticated username.

Remember our certificate?

```
CN=vishal
```

During authentication, Kubernetes extracted:

```
User = vishal
```

RBAC now searches for:

```
RoleBinding

↓

User

↓

vishal
```

Match found.

---

## roleRef

```yaml
roleRef:
```

Specifies **which Role** should be assigned.

---

## kind

```yaml
kind: Role
```

Assign a namespace Role.

---

## name

```yaml
name: pod-reader
```

Grant the permissions defined in the **pod-reader** Role.

---

# Internal Authorization Flow

```
kubectl

      │

      ▼

Authentication

      │

User = vishal

      │

RoleBinding Search

      │

Found

pod-reader-binding

      │

Role

pod-reader

      │

Permissions

✔ get

✔ list

✔ watch

      │

Request Allowed
```

---

# Step 16 - Test the Permissions

Switch back to Vishal.

```bash
kubectl config use-context vishal-context
```

Verify.

```bash
kubectl config current-context
```

Output

```
vishal-context
```

Now test the following commands.

---

## List Pods

```bash
kubectl get pods
```

Expected

```
Works
```

---

## Watch Pods

```bash
kubectl get pods -w
```

Expected

```
Works
```

Press **Ctrl + C** to stop watching.

---

## Describe a Pod

```bash
kubectl describe pod <pod-name>
```

Expected

```
Works
```

---

## Delete a Pod

```bash
kubectl delete pod <pod-name>
```

Expected

```
Forbidden
```

Reason:

The Role does not contain the **delete** verb.

---

## List Namespaces

```bash
kubectl get namespaces
```

Expected

```
Forbidden
```

Reason:

The Role grants permissions only on **Pods**.

---

## List Pods Across All Namespaces

```bash
kubectl get pods -A
```

Expected

```
Forbidden
```

Reason:

The Role exists only inside the **dev** namespace.

It cannot grant cluster-wide access.

---

# Authorization Flow Completed

```
Client Certificate

        │

Authentication

        │

User = vishal

        │

RoleBinding

        │

pod-reader-binding

        │

Role

        │

pod-reader

        │

Permissions

get
list
watch

        │

Request Allowed
```

---

# What We Learned

In this section, we learned:

- Why authentication alone is not enough.
- What a Role is.
- Why a Role does not belong to any user.
- How a Role defines permissions.
- What a RoleBinding is.
- How a RoleBinding connects a User to a Role.
- Why `kubectl get pods` succeeds after RoleBinding.
- Why `kubectl delete pod` still fails.
- Why `kubectl get namespaces` is still forbidden.
- Why `kubectl get pods -A` is also forbidden.

> **Important:** We intentionally stopped here. We have not discussed **ClusterRole** or **ClusterRoleBinding** because they are separate RBAC concepts and will be covered in the next notes file.

# Internal Working of Kubernetes RBAC

Now that we have completed the practical, let's understand what actually happens internally whenever we execute a Kubernetes command.

Suppose Vishal executes:

```bash
kubectl get pods
```

The request follows the sequence shown below.

---

# Complete Request Flow

```
                 kubectl

                    │

                    ▼

           Read kubeconfig File

                    │

                    ▼

Uses Context

vishal-context

                    │

                    ▼

Uses User

vishal

                    │

                    ▼

Uses

vishal.crt

+

vishal.key

                    │

                    ▼

Kubernetes API Server

                    │

                    ▼

Authentication

                    │

Certificate Verified?

                    │

          Yes               No
           │                │
           ▼                ▼

User = vishal         Request Rejected

           │

           ▼

Authorization (RBAC)

           │

Find RoleBinding

           │

Found?

           │

      Yes        No
       │         │
       ▼         ▼

Read Role     Forbidden

       │

       ▼

Permission Exists?

       │

   Yes        No
    │          │
    ▼          ▼

Allowed    Forbidden
```

---

# Understanding kubeconfig

The kubeconfig file stores all the information required by kubectl.

It contains three important sections.

```
Clusters

↓

Users

↓

Contexts
```

---

## Cluster

The Cluster section stores:

- Kubernetes API Server Address
- CA Certificate
- Cluster Information

Example

```
abhi.k8s.local
```

---

## User

The User section stores authentication credentials.

Examples

```
abhi.k8s.local

vishal
```

For Vishal, kubeconfig stores:

```
vishal.crt

+

vishal.key
```

---

## Context

A Context connects:

```
Cluster

+

User

+

Namespace
```

Example

```
Context

↓

vishal-context

↓

Cluster

abhi.k8s.local

↓

User

vishal

↓

Namespace

dev
```

Whenever we switch context,

kubectl automatically changes all three values.

---

# Why Didn't We Create a Linux User?

Many beginners ask this question.

We created:

```
vishal.crt
```

but we never executed:

```bash
useradd vishal
```

Why?

Because Kubernetes does **not** authenticate Linux users.

It authenticates certificates.

The API Server only checks:

```
Certificate

↓

Signed by trusted Kubernetes CA?

↓

Yes

↓

CN=vishal

↓

Authenticated User

↓

vishal
```

Whether a Linux user exists is completely irrelevant.

---

# Why Didn't We Create an AWS IAM User?

Similarly,

we never created:

```
IAM User

↓

vishal
```

because Kubernetes does not require IAM authentication.

AWS IAM authentication is only one of several supported authentication mechanisms.

Our practical uses **Certificate Authentication**.

Therefore IAM is not involved.

---

# Authentication vs Authorization

These two concepts should never be confused.

| Authentication | Authorization |
|---------------|---------------|
| Verifies identity | Verifies permissions |
| Who are you? | What can you do? |
| Certificate | RBAC |
| Happens first | Happens second |

---

# Understanding the Role

A Role answers one question.

```
What actions are allowed?
```

Example

```
Role

↓

pod-reader

↓

Resources

Pods

↓

Permissions

get

list

watch
```

Notice:

The Role does **not** mention any user.

---

# Understanding the RoleBinding

A RoleBinding answers another question.

```
Who receives this Role?
```

Example

```
RoleBinding

↓

User

vishal

↓

Role

pod-reader
```

Now Kubernetes understands:

```
User

↓

vishal

↓

Gets

↓

pod-reader
```

---

# Authentication + Authorization Together

```
User Executes

kubectl get pods

          │

          ▼

Client Certificate Sent

          │

          ▼

API Server

          │

Authentication

          │

User Identified

vishal

          │

Authorization

          │

RoleBinding Lookup

          │

pod-reader-binding

          │

Role Lookup

          │

pod-reader

          │

Allowed Verbs

get

list

watch

          │

Permission Found

          │

Pods Listed
```

---

# Why These Commands Worked

## Command

```bash
kubectl get pods
```

Reason

```
Verb

list

↓

Allowed
```

---

## Command

```bash
kubectl get pods -w
```

Reason

```
Verb

watch

↓

Allowed
```

---

## Command

```bash
kubectl describe pod test
```

Reason

Internally Kubernetes performs **GET** operations.

Since the Role allows:

```
get
```

the command succeeds.

---

# Why These Commands Failed

## Command

```bash
kubectl delete pod test
```

Reason

Our Role does not contain:

```
delete
```

Result

```
Forbidden
```

---

## Command

```bash
kubectl get namespaces
```

Reason

Our Role only grants access to:

```
Pods
```

No permission exists for:

```
Namespaces
```

Result

```
Forbidden
```

---

## Command

```bash
kubectl get pods -A
```

Reason

The Role exists only inside:

```
dev
```

The command requests Pods from **all namespaces**.

Result

```
Forbidden
```

---

# Common Interview Questions

## Q1. What is RBAC?

RBAC (Role-Based Access Control) is Kubernetes' authorization mechanism that controls what actions a user, group, or service account can perform on cluster resources.

---

## Q2. What is the difference between Authentication and Authorization?

Authentication verifies **who the user is**, while Authorization verifies **what the authenticated user is allowed to do**.

---

## Q3. Does Kubernetes authenticate Linux users?

No.

Kubernetes authenticates certificates, tokens, IAM identities, service accounts, etc., not Linux users.

---

## Q4. Why did we create a CSR?

A CSR (Certificate Signing Request) is used to request the Kubernetes Certificate Authority to issue a signed client certificate.

---

## Q5. Why do we need the Kubernetes CA?

The Kubernetes API Server trusts only certificates signed by its own Certificate Authority.

Without the trusted CA signature, authentication fails.

---

## Q6. Why was `kubectl get pods` initially forbidden?

Because the user was successfully authenticated but had no RBAC permissions assigned.

---

## Q7. Why did `kubectl delete pod` fail?

Because the Role granted only:

- get
- list
- watch

It did not grant the **delete** verb.

---

## Q8. Why did `kubectl get pods -A` fail?

Because the Role was created only for the **dev** namespace.

The request attempted to access Pods across the entire cluster.

---

## Q9. Can one Role be used by multiple users?

Yes.

A single Role can be referenced by multiple RoleBindings.

---

## Q10. Can one RoleBinding bind multiple users?

Yes.

The `subjects` section can contain multiple Users, Groups, or ServiceAccounts.

---

# Command Cheatsheet

## Generate Private Key

```bash
openssl genrsa -out vishal.key 2048
```

---

## Generate CSR

```bash
openssl req -new \
-key vishal.key \
-out vishal.csr \
-subj "/CN=vishal/O=developers"
```

---

## Extract CA Private Key

```bash
grep privateMaterial keyset.yaml | awk '{print $2}' | base64 -d > ca.key
```

---

## Extract CA Certificate

```bash
grep publicMaterial keyset.yaml | awk '{print $2}' | base64 -d > ca.crt
```

---

## Sign the CSR

```bash
openssl x509 -req \
-in vishal.csr \
-CA ca.crt \
-CAkey ca.key \
-CAcreateserial \
-out vishal.crt \
-days 365
```

---

## Add User

```bash
kubectl config set-credentials vishal \
--client-certificate=/root/vishal.crt \
--client-key=/root/vishal.key
```

---

## Create Context

```bash
kubectl config set-context vishal-context \
--cluster=abhi.k8s.local \
--user=vishal \
--namespace=dev
```

---

## Switch Context

```bash
kubectl config use-context vishal-context
```

---

## Create Role

```bash
kubectl apply -f role.yaml
```

---

## Create RoleBinding

```bash
kubectl apply -f rolebinding.yaml
```

---

## Verify Role

```bash
kubectl describe role pod-reader -n dev
```

---

## Verify RoleBinding

```bash
kubectl describe rolebinding pod-reader-binding -n dev
```

---

# Key Takeaways

- RBAC is Kubernetes' authorization mechanism.
- Authentication and Authorization are separate processes.
- Client certificates are one way to authenticate Kubernetes users.
- Kubernetes users created through certificates are **not Linux users**.
- A **Role** defines permissions but is not assigned to anyone.
- A **RoleBinding** assigns a Role to a User, Group, or ServiceAccount.
- Permissions are granted using **verbs** such as `get`, `list`, and `watch`.
- Without a matching RoleBinding, authenticated users receive a **Forbidden** error.
- Namespace-scoped Roles grant permissions only within their own namespace.

---

# Summary

In this chapter, we built a complete certificate-based authentication and RBAC workflow from scratch. We generated a private key, created a Certificate Signing Request (CSR), signed it using the Kubernetes Certificate Authority, configured a new user in `kubeconfig`, created a dedicated context, and verified successful authentication. We then implemented authorization by creating a namespace-scoped **Role** and **RoleBinding**, granting the user controlled access to Pods in the `dev` namespace. Through practical testing, we observed both successful and denied operations, clearly demonstrating how Kubernetes enforces the principle of least privilege using RBAC.

> **Next Topic:** **ClusterRole & ClusterRoleBinding** (Cluster-Wide Authorization)
> **Note:** This file focuses only on **Certificate Authentication, Role, and RoleBinding**. **ClusterRole** and **ClusterRoleBinding** will be covered separately in the next notes file.

---
