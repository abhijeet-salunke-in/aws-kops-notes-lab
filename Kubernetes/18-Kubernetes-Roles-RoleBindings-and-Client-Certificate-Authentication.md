# 18. Kubernetes Roles, RoleBindings & Client Certificate Authentication

## What We Will Learn

In the previous practical, we learned how to create a **Role** and **RoleBinding**. However, one important question remained unanswered:

> **How does Kubernetes know who the user is before checking RBAC?**

This document answers that question by introducing **Client Certificate Authentication**.

By the end of this guide, you will understand the complete Kubernetes security flow:

```text
Authentication
       │
       ▼
API Server identifies the user
       │
       ▼
Authorization (RBAC)
       │
       ▼
Allow / Deny
```

---

# Authentication vs Authorization

Many beginners confuse these two terms.

Authentication answers:

> **Who are you?**

Authorization answers:

> **What are you allowed to do?**

Example:

```text
Employee enters office
        │
        ▼
Shows ID Card
        │
(Authentication)
        │
        ▼
Security verifies identity
        │
        ▼
Checks Access Card Permissions
        │
(Authorization)
        │
        ▼
Allowed into Server Room
```

The same process happens inside Kubernetes.

```text
kubectl
     │
     ▼
Client Certificate
     │
     ▼
API Server
     │
Authentication
     │
     ▼
RBAC
     │
Authorization
     │
     ▼
Access Granted / Forbidden
```

---

# Why Do We Need Authentication?

Suppose anyone could run:

```bash
kubectl get pods
```

How would Kubernetes know:

- Who sent the request?
- Is the user really who they claim to be?
- Which RoleBinding belongs to that user?

Without authentication, RBAC would never know which user to check.

Therefore, **Authentication always happens before Authorization.**

---

# What is a Client Certificate?

A Client Certificate is a digital identity card.

Instead of telling Kubernetes:

> "Trust me, I am Abhi."

you present a certificate that says:

> "A trusted Certificate Authority has verified this identity."

The Kubernetes API Server trusts certificates that are signed by its trusted Certificate Authority (CA).

---

# Real Life Analogy

Imagine entering a company.

You don't simply say:

> "I work here."

Instead, you show your employee ID.

```text
Employee
     │
Shows Employee ID
     │
Security Guard
     │
Verifies ID
     │
Allows Entry
```

In Kubernetes:

```text
kubectl
     │
Client Certificate
     │
API Server
     │
Verifies Certificate
     │
Authentication Successful
```

The Employee ID is similar to a Client Certificate.

---

# What is a Certificate Authority (CA)?

A Certificate Authority (CA) is a trusted entity responsible for issuing certificates.

Only certificates signed by the trusted Kubernetes CA are accepted.

Real-life example:

```text
Government
      │
Issues Passport
      │
Citizen
```

Similarly,

```text
Kubernetes CA
        │
Signs Certificate
        │
User
```

If someone creates their own certificate, Kubernetes rejects it because it was **not signed by the trusted CA**.

---

# Components Used During Authentication

During this practical, we created four important files.

| File | Purpose |
|-------|----------|
| abhi.key | Private Key |
| abhi.csr | Certificate Signing Request |
| abhi.crt | Signed Client Certificate |
| abhi.kubeconfig | Login configuration used by kubectl |

---

# Purpose of Each File

## 1. abhi.key

This is the **Private Key**.

It proves ownership of the certificate.

Never share this file.

---

## 2. abhi.csr

CSR stands for:

> **Certificate Signing Request**

This file is sent to Kubernetes requesting a signed certificate.

---

## 3. abhi.crt

This is the signed Client Certificate.

The Kubernetes API Server reads this certificate to identify the user.

In our practical:

```text
CN = abhi

O = developers
```

Therefore,

Username becomes:

```text
abhi
```

Group becomes:

```text
developers
```

---

## 4. abhi.kubeconfig

This file contains:

- API Server address
- CA Certificate
- Client Certificate
- Private Key
- Context information

Whenever kubectl is executed, it reads this file to know:

- Which cluster to connect to
- Which certificate to use
- Which identity to present

---

# Complete Authentication Flow

```text
kubectl get pods
        │
        ▼
Reads kubeconfig
        │
        ▼
Loads Client Certificate
        │
        ▼
Sends request to API Server
        │
        ▼
API Server verifies:

✓ Trusted CA
✓ Certificate Validity
✓ Digital Signature
        │
        ▼
Authentication Successful
        │
        ▼
Username = abhi
Group = developers
        │
        ▼
RBAC starts checking Roles & RoleBindings
```

---

# Authentication vs Authorization Flow

```text
kubectl
     │
     ▼
Client Certificate
     │
     ▼
API Server
     │
Authentication
     │
     ▼
User = abhi
     │
     ▼
RoleBinding
     │
     ▼
Role
     │
     ▼
Allowed?
```

---

# Common Mistake

Many beginners think:

> RoleBinding authenticates users.

❌ Incorrect.

RoleBinding **never authenticates anyone**.

Correct flow:

```text
Certificate

↓

Authentication

↓

User Identified

↓

RoleBinding

↓

Role

↓

Permission Granted
```

Authentication and Authorization are completely different processes.

---

# Key Points

- Client Certificates are used for Authentication.
- RBAC is used for Authorization.
- Authentication always happens before Authorization.
- Kubernetes trusts only certificates signed by its trusted CA.
- The API Server extracts the username and group from the Client Certificate.
- RoleBindings work only after the user has been authenticated.

---

# Interview Questions

### Q1. What is the difference between Authentication and Authorization?

### Q2. What is a Client Certificate?

### Q3. Why is a Certificate Authority (CA) required?

### Q4. What information is stored inside a Client Certificate?

### Q5. What is a CSR?

### Q6. What is the purpose of kubeconfig?

### Q7. Which Kubernetes component authenticates the user?

### Q8. Does RoleBinding authenticate users?

### Q9. Which comes first, Authentication or Authorization?

### Q10. How does Kubernetes know the username "abhi" from the certificate?

---

**Next Part:** Practical implementation of Client Certificate Authentication, creating a Kubernetes user using CSR, generating `abhi.kubeconfig`, testing authentication, and integrating it with Roles and RoleBindings.

# Practical: Kubernetes Client Certificate Authentication

In this practical, we will create a real Kubernetes user named **abhi** using a **Client Certificate**.

Unlike the `kubectl auth can-i --as=abhi` command (which only impersonates a user), this practical performs **real authentication**.

---

# Practical Architecture

```text
Generate Private Key
        │
        ▼
Create Certificate Signing Request (CSR)
        │
        ▼
Submit CSR to Kubernetes
        │
        ▼
Approve CSR
        │
        ▼
Kubernetes CA signs certificate
        │
        ▼
Download Signed Certificate
        │
        ▼
Create kubeconfig
        │
        ▼
Login as User "abhi"
        │
        ▼
Authentication Successful
        │
        ▼
RBAC Checks Permissions
```

---

# Step 1 : Generate Private Key

Command

```bash
openssl genrsa -out abhi.key 2048
```

Verify

```bash
ls -l abhi.key
```

## Why?

Every certificate belongs to one unique private key.

Think of it like:

```text
House

↓

House Key

↓

Only Owner Can Open It
```

Similarly,

```text
Certificate

↓

Private Key

↓

Only Owner Can Use It
```

The private key must never be shared.

---

# Step 2 : Generate Certificate Signing Request (CSR)

Command

```bash
openssl req -new \
-key abhi.key \
-out abhi.csr \
-subj "/CN=abhi/O=developers"
```

Verify

```bash
openssl req -text -noout -verify -in abhi.csr
```

Expected

```text
CN = abhi

O = developers
```

## Why?

A CSR is simply a request asking Kubernetes:

> Please create a certificate for this user.

The CSR contains:

- Public Key
- Username
- Group

It does **NOT** contain the private key.

---

# Step 3 : Convert CSR to Base64

Command

```bash
cat abhi.csr | base64 | tr -d '\n'
```

Copy the complete output.

## Why?

The Kubernetes CSR API expects the certificate request in Base64 format.

---

# Step 4 : Create CSR YAML

Create

```text
abhi-csr.yaml
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest

metadata:
  name: abhi

spec:
  request: <BASE64_OUTPUT>

  signerName: kubernetes.io/kube-apiserver-client

  usages:
  - client auth
```

---

# Understanding Every YAML Field

## apiVersion

```yaml
apiVersion: certificates.k8s.io/v1
```

CSR belongs to the Kubernetes Certificates API.

---

## kind

```yaml
kind: CertificateSigningRequest
```

Creates a CSR object.

---

## metadata

```yaml
metadata:
  name: abhi
```

Name of the Kubernetes CSR object.

---

## request

```yaml
request:
```

Contains the Base64 encoded CSR.

---

## signerName

```yaml
signerName:
```

Tells Kubernetes which signer should sign the certificate.

Here,

```text
kubernetes.io/kube-apiserver-client
```

means

> Create a Client Certificate.

---

## usages

```yaml
usages:
- client auth
```

Specifies how the certificate will be used.

In our case,

Client Authentication.

---

# Step 5 : Create CSR Object

Command

```bash
kubectl apply -f abhi-csr.yaml
```

Verify

```bash
kubectl get csr
```

Expected

```text
Pending
```

## Why?

Creating a CSR does not automatically issue a certificate.

It waits for administrator approval.

---

# Step 6 : Approve CSR

Command

```bash
kubectl certificate approve abhi
```

Verify

```bash
kubectl get csr
```

Expected

```text
Approved,Issued
```

## Why?

Only trusted administrators should approve certificate requests.

Otherwise, anyone could request certificates.

---

# Step 7 : Download Signed Certificate

Command

```bash
kubectl get csr abhi \
-o jsonpath='{.status.certificate}' \
| base64 -d > abhi.crt
```

Verify

```bash
openssl x509 -text -noout -in abhi.crt
```

Expected

```text
Issuer

CN=kubernetes-ca

Subject

CN=abhi

O=developers
```

## Why?

Now Kubernetes has created a real Client Certificate.

Notice

Issuer

↓

Kubernetes CA

This proves Kubernetes signed the certificate.

---

# Step 8 : Extract Cluster CA Certificate

Command

```bash
kubectl config view --raw \
-o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
| base64 -d > ca.crt
```

Verify

```bash
openssl x509 -text -noout -in ca.crt
```

## Why?

Our Kops cluster embeds the CA certificate inside kubeconfig.

We extracted it to build another kubeconfig.

---

# Step 9 : Create Cluster Entry

```bash
kubectl config set-cluster abhi-cluster \
--server=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}') \
--certificate-authority=ca.crt \
--embed-certs=true \
--kubeconfig=abhi.kubeconfig
```

## Why?

Defines

- Cluster Name
- API Server
- Trusted CA

inside the new kubeconfig.

---

# Step 10 : Add User

```bash
kubectl config set-credentials abhi \
--client-certificate=abhi.crt \
--client-key=abhi.key \
--embed-certs=true \
--kubeconfig=abhi.kubeconfig
```

## Why?

Adds the authentication details of user **abhi**.

---

# Step 11 : Create Context

```bash
kubectl config set-context abhi-context \
--cluster=abhi-cluster \
--user=abhi \
--namespace=default \
--kubeconfig=abhi.kubeconfig
```

## Why?

A Context connects

```text
Cluster

↓

User

↓

Namespace
```

into one configuration.

---

# Step 12 : Use Context

```bash
kubectl config use-context abhi-context \
--kubeconfig=abhi.kubeconfig
```

Verify

```bash
kubectl config get-contexts \
--kubeconfig=abhi.kubeconfig
```

## Why?

Makes

```text
abhi-context
```

the active context.

---

# Step 13 : Test Authentication

```bash
kubectl --kubeconfig=abhi.kubeconfig get pods
```

Output

```text
Error from server (Forbidden)

User "abhi" cannot list resource "pods"
```

## Why?

Authentication

✅ Successful

Authorization

❌ Failed

No RoleBinding exists yet.

Notice

We received

```text
Forbidden
```

NOT

```text
Unauthorized
```

This proves Kubernetes recognized the user.

---

# Production Style Testing (Extra Practical)

Instead of typing

```bash
kubectl --kubeconfig=abhi.kubeconfig get pods
```

every time,

Production users keep their kubeconfig inside

```text
~/.kube/config
```

Create Linux User

```bash
useradd -m abhi
```

Create Kubernetes Folder

```bash
mkdir -p /home/abhi/.kube
```

Copy kubeconfig

```bash
cp abhi.kubeconfig /home/abhi/.kube/config
```

Change Ownership

```bash
chown -R abhi:abhi /home/abhi/.kube
```

Login

```bash
su - abhi
```

Now simply execute

```bash
kubectl get pods
```

No need to specify

```text
--kubeconfig
```

because kubectl automatically reads

```text
~/.kube/config
```

---

# Folder Structure

```text
/home/abhi

│

├── .kube

│      └── config

│

└── kubectl
```

---

# Authentication Flow

```text
kubectl

↓

Reads ~/.kube/config

↓

Loads Client Certificate

↓

API Server

↓

Certificate Verified

↓

User = abhi

↓

Authentication Successful

↓

RBAC Starts
```

---

# Common Mistakes

❌ Sharing the Private Key

Never share

```text
abhi.key
```

---

❌ Thinking CSR Automatically Creates Certificate

CSR

↓

Pending

↓

Administrator Approval Required

↓

Certificate Issued

---

❌ Thinking Forbidden Means Authentication Failed

```text
Unauthorized

↓

Authentication Failed
```

```text
Forbidden

↓

Authentication Successful

Authorization Failed
```

---

# Interview Questions

### What is CSR?

### Why is Base64 encoding required?

### What does signerName mean?

### What happens after approving CSR?

### What is stored inside kubeconfig?

### Why do we create a Context?

### Difference between Forbidden and Unauthorized?

### Where does kubectl search for kubeconfig by default?

### Why does Production use ~/.kube/config instead of --kubeconfig?

---

**Next Part:** Integrating Client Certificate Authentication with Kubernetes Roles and RoleBindings, complete RBAC workflow, production architecture, troubleshooting, interview questions, and summary.

# Integrating Client Certificate Authentication with RBAC

Until now, we have completed two different processes.

## Authentication

We successfully authenticated a real Kubernetes user named **abhi** using a Client Certificate.

Authentication answered:

> **Who are you?**

---

## Authorization

Next, Kubernetes uses RBAC to answer:

> **What are you allowed to do?**

Both processes work together.

---

# Complete Kubernetes Security Flow

```text
                kubectl

                    │

                    ▼

         ~/.kube/config

                    │

                    ▼

      Client Certificate + Private Key

                    │

                    ▼

              API Server

                    │

         Authentication

                    │

          User = abhi

                    │

                    ▼

             RoleBinding

                    │

                    ▼

                Role

                    │

                    ▼

          Permission Granted?
```

---

# What Happened During Our Practical?

Initially, we executed

```bash
kubectl get pods
```

using

```text
abhi.kubeconfig
```

Output

```text
Error from server (Forbidden)

User "abhi" cannot list resource "pods"
```

Many beginners think this means authentication failed.

That is incorrect.

Actually,

```text
Authentication

✓ Successful
```

because Kubernetes already identified the user as

```text
abhi
```

The request failed because no RoleBinding existed.

---

# Why Did Kubernetes Display "User abhi"?

Notice the error.

```text
User "abhi"
```

How did Kubernetes know the username?

Because our certificate contained

```text
Subject

CN = abhi

O = developers
```

The API Server extracts

```text
CN

↓

Username
```

and

```text
O

↓

Group
```

This information is then used by RBAC.

---

# Creating Role

Role YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: pod-reader
  namespace: default

rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs:
  - get
  - list
  - watch
```

Apply

```bash
kubectl apply -f role.yaml
```

Verify

```bash
kubectl get role
```

---

# Creating RoleBinding

RoleBinding YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: pod-reader-binding
  namespace: default

subjects:
- kind: User
  name: abhi
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply

```bash
kubectl apply -f rolebinding.yaml
```

Verify

```bash
kubectl get rolebinding
```

---

# Why Did It Work?

RoleBinding says

```text
User

↓

abhi
```

The certificate also says

```text
CN

↓

abhi
```

Both match.

Therefore,

RBAC grants the permissions defined inside the Role.

---

# Final Test

Login

```bash
su - abhi
```

Execute

```bash
kubectl get pods
```

Output

```text
No resources found in default namespace.
```

This proves

```text
Authentication

✓ Successful

Authorization

✓ Successful
```

The namespace simply contains no Pods.

---

# Unauthorized vs Forbidden

This is one of the most frequently asked interview questions.

## Unauthorized

Means

Authentication Failed

Examples

- Invalid Certificate
- Expired Certificate
- Unknown CA
- Invalid Token

```text
Client

↓

Authentication Failed

↓

Unauthorized
```

---

## Forbidden

Means

Authentication Successful

Authorization Failed

```text
Client

↓

Authentication Successful

↓

RBAC Denied

↓

Forbidden
```

During our practical

We intentionally observed

```text
Forbidden
```

before creating the RoleBinding.

---

# Authentication Mechanisms in Kubernetes

Kubernetes supports multiple authentication methods.

| Method | Common Usage |
|----------|-------------|
| Client Certificates | Self-managed clusters (Kops, kubeadm) |
| ServiceAccount Tokens | Pods |
| OIDC | Google, Azure AD, Keycloak |
| LDAP | Enterprise |
| Webhook Authentication | External systems |
| AWS IAM | Amazon EKS |

Authentication method changes.

RBAC does not.

RBAC always works the same.

---

# Kops vs kubeadm vs EKS

| Feature | kubeadm | Kops | Amazon EKS |
|----------|----------|------|------------|
| Cluster Management | Manual | Automated | Managed by AWS |
| Authentication Method | Client Certificates | Client Certificates | IAM / OIDC |
| RBAC | Yes | Yes | Yes |
| Uses CSR API | Yes | Yes | Yes |

Although authentication methods differ,

RBAC remains identical.

---

# Production Architecture

```text
Developer

↓

SSH

↓

Linux User

↓

~/.kube/config

↓

Client Certificate

↓

API Server

↓

Authentication

↓

RBAC

↓

RoleBinding

↓

Role

↓

Access Granted
```

---

# Common Mistakes

## Mistake 1

Thinking

```text
Role

↓

User
```

Incorrect.

Correct

```text
RoleBinding

↓

User

↓

Role
```

---

## Mistake 2

Thinking

Certificate provides permissions.

Incorrect.

Certificate only identifies the user.

Permissions always come from RBAC.

---

## Mistake 3

Thinking

Linux User

↓

Kubernetes User

Automatically

Incorrect.

Linux usernames and Kubernetes usernames are completely independent.

We intentionally used

```text
abhi
```

for both.

Production environments may use

```text
Linux User

↓

john
```

Certificate

```text
CN

↓

developer1
```

This still works.

---

## Mistake 4

Sharing

```text
abhi.key
```

Never share private keys.

Anyone possessing both

```text
abhi.key

+

abhi.crt
```

can authenticate as that user.

---

# Real Interview Flow

Interviewer:

How does Kubernetes authenticate a user?

Answer:

```text
kubectl reads kubeconfig.

↓

Client Certificate is sent to API Server.

↓

API Server verifies

• Trusted CA
• Certificate Validity
• Digital Signature

↓

Authentication Successful

↓

API Server extracts

CN

↓

Username

↓

RBAC checks RoleBinding

↓

Role

↓

Permission Granted or Denied
```

---

# Interview Questions

### What is Authentication?

### What is Authorization?

### Difference between Authentication and Authorization?

### What is a Client Certificate?

### What is a Certificate Authority?

### What is CSR?

### What is kubeconfig?

### Why does Kubernetes use Base64 CSR?

### What is signerName?

### Difference between Unauthorized and Forbidden?

### Does Role authenticate users?

### Does RoleBinding authenticate users?

### Which Kubernetes component authenticates users?

### How does Kubernetes determine the username?

### Where is username stored inside the certificate?

### What is stored in CN?

### What is stored in O?

### Why do we need a private key?

### Can anyone generate certificates?

### Why is CSR approval required?

### What is the default kubeconfig location?

### How does kubectl find kubeconfig?

### How does RBAC know which user is making the request?

### Why did our request fail before RoleBinding?

### Why did it succeed after RoleBinding?

---

# Summary

In this chapter, we completed the complete Kubernetes authentication and authorization workflow.

We learned

- Authentication
- Authorization
- Client Certificates
- Certificate Authority
- Private Key
- CSR
- Kubernetes CSR API
- kubeconfig
- Contexts
- Role
- RoleBinding
- Production Login
- Authentication Flow
- Authorization Flow
- Unauthorized vs Forbidden
- Kops-based Client Certificate Authentication

We also created a real Kubernetes user named **abhi**, authenticated using a Client Certificate, generated a dedicated kubeconfig, logged in using that identity, observed RBAC denying access before a RoleBinding existed, and finally granted access through RBAC.

This completes the entire Kubernetes security workflow from **authentication** to **authorization**, which is the foundation for securing access in self-managed Kubernetes clusters such as **Kops** and **kubeadm**.
