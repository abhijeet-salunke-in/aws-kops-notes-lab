# Kubernetes AWS IAM Authentication and RBAC Authorization

## Part 1: Introduction and Theory

---

# What Will We Learn?

In the previous section, we learned how to provide Kubernetes cluster access using **Client Certificate Authentication**. In that approach, we created a Kubernetes user by generating certificates signed by the Kubernetes CA and then controlled that user's permissions using RBAC.

Although this method works well, organizations that use AWS usually manage users through **AWS IAM (Identity and Access Management)** instead of creating Kubernetes certificates for every user.

In this section, we will learn how to authenticate AWS IAM users with a Kubernetes cluster created using **Kops**, and then authorize those users using Kubernetes RBAC.

---

# Learning Objectives

By the end of this guide, you will be able to:

- Enable AWS IAM Authentication in a Kops cluster.
- Create an IAM user in AWS.
- Map the IAM user to Kubernetes.
- Understand the role of `aws-iam-authenticator`.
- Configure RBAC using Roles and RoleBindings.
- Grant namespace-specific access to an IAM user.
- Troubleshoot common AWS Authentication issues.

---

# What is Authentication?

**Authentication** is the process of verifying **who the user is**.

It answers the question:

> **"Who are you?"**

If the identity is verified successfully, the user is considered authenticated.

Examples of authentication include:

- Username and Password
- Client Certificates
- AWS IAM User
- IAM Roles
- OIDC
- LDAP

Authentication **only verifies identity**. It does **not** decide what the user is allowed to do.

---

# What is Authorization?

After authentication is successful, Kubernetes performs **Authorization**.

Authorization answers the question:

> **"What are you allowed to do?"**

Kubernetes uses **RBAC (Role-Based Access Control)** to determine the permissions of an authenticated user.

Examples:

- Can the user list Pods?
- Can the user create Deployments?
- Can the user delete Services?
- Can the user access only one namespace?
- Can the user manage the entire cluster?

---

# Authentication vs Authorization

| Authentication | Authorization |
|---------------|---------------|
| Verifies identity | Verifies permissions |
| Answers **Who are you?** | Answers **What can you do?** |
| Happens first | Happens after authentication |
| AWS IAM, Certificates, OIDC | RBAC, Roles, ClusterRoles |

---

# Recap: Client Certificate Authentication

Previously, we created a Kubernetes user using **Client Certificates**.

The process was:

```
Generate Private Key
        │
        ▼
Generate CSR
        │
        ▼
Sign CSR using Kubernetes CA
        │
        ▼
Create kubeconfig
        │
        ▼
Authenticate to Kubernetes
        │
        ▼
RBAC decides permissions
```

Although this method is secure, managing certificates becomes difficult when many users join or leave an organization.

---

# Why AWS IAM Authentication?

Most organizations already manage their employees using **AWS IAM**.

Instead of creating Kubernetes certificates for every developer, DevOps engineer, or administrator, we can reuse their existing IAM identities.

This provides several advantages:

- Centralized user management
- No need to create Kubernetes certificates
- Easier onboarding and offboarding
- Better auditing using AWS CloudTrail
- Integration with existing AWS security policies

---

# Why Do We Need aws-iam-authenticator?

Kubernetes does **not** understand AWS IAM users directly.

When an IAM user sends a request, Kubernetes only receives an HTTPS request. It does not know whether that request came from a valid IAM user.

The **aws-iam-authenticator** acts as a bridge between AWS IAM and Kubernetes.

Its responsibilities are:

- Verify AWS IAM identity.
- Validate AWS authentication token.
- Map IAM users to Kubernetes usernames.
- Pass the authenticated username to the Kubernetes API Server.

Without the authenticator, Kubernetes cannot authenticate AWS IAM users.

---

# Authentication Flow

The complete authentication process looks like this:

```
IAM User
    │
    ▼
AWS CLI
    │
    ▼
Authentication Token
    │
    ▼
aws-iam-authenticator
    │
    ▼
Kubernetes API Server
    │
Authentication Successful
    │
    ▼
RBAC
    │
    ▼
Allow or Deny Request
```

---

# Authentication and Authorization Flow

```
                User Request
                     │
                     ▼
          AWS IAM Authentication
                     │
                     ▼
      aws-iam-authenticator verifies identity
                     │
                     ▼
          Kubernetes API Server
                     │
                     ▼
      RBAC (Role / RoleBinding)
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
      Access Allowed      Access Denied
```

---

# Real Lab Scenario

In this practical, we will perform the following tasks:

1. Enable AWS Authentication in a Kops cluster.
2. Create an IAM User.
3. Generate Access Keys.
4. Configure AWS CLI for the IAM user.
5. Configure the `aws-iam-authenticator`.
6. Map the IAM user to Kubernetes.
7. Create a Namespace.
8. Create a Role.
9. Create a RoleBinding.
10. Verify that the IAM user can access only the permitted namespace.

---

# Prerequisites

Before starting this practical, ensure the following:

- AWS Account
- Running Kops Kubernetes Cluster
- kubectl installed
- AWS CLI installed
- Kops installed
- Cluster administrator access
- RBAC enabled

---

# Key Components Used

| Component | Purpose |
|-----------|---------|
| AWS IAM | User Identity |
| aws-iam-authenticator | Authenticates IAM Users |
| ConfigMap | Maps IAM Users to Kubernetes Users |
| Role | Defines Namespace Permissions |
| RoleBinding | Assigns Role to User |
| Namespace | Restricts Resource Access |
| Kubernetes API Server | Processes Requests |

---

# Important Note

Remember the sequence:

```
Authentication
        ↓
Authorization
        ↓
Access Granted
```

If Authentication fails, Authorization is never executed.

Similarly, even if Authentication succeeds, the user still cannot access resources unless appropriate RBAC permissions are assigned.

---

## What's Next?

In **Part 2**, we will enable AWS IAM Authentication in the Kops cluster by updating the cluster configuration, performing a rolling update, and verifying that the `aws-iam-authenticator` component is deployed successfully.

# Part 2: Enable AWS IAM Authentication in Kops Cluster

---

# Objective

By default, a Kubernetes cluster created using Kops uses **Client Certificate Authentication** for accessing the cluster.

To allow AWS IAM Users to authenticate with Kubernetes, we must enable **AWS Authentication** in the cluster configuration.

After enabling it, Kops will deploy the **aws-iam-authenticator** component, which verifies AWS IAM identities before allowing them to communicate with the Kubernetes API Server.

---

# Step 1: Check Current Cluster Configuration

Before making any changes, verify whether AWS Authentication is already enabled.

```bash
kops get cluster abhi.k8s.local -o yaml
```

### Example Output

```yaml
spec:
  authorization:
    rbac: {}
```

If you do not see:

```yaml
authentication:
  aws: {}
```

then AWS Authentication is **not enabled**.

### Why?

We first inspect the current cluster configuration so that we know whether AWS Authentication already exists or not.

---

# Step 2: Edit the Cluster Configuration

Open the cluster configuration.

```bash
kops edit cluster abhi.k8s.local
```

Locate the **api** section.

Example:

```yaml
api:
  loadBalancer:
    class: Network
    type: Public
```

Add the following section just below it (or anywhere under `spec`):

```yaml
authentication:
  aws: {}
```

The configuration should now look similar to:

```yaml
spec:

  api:
    loadBalancer:
      class: Network
      type: Public

  authentication:
    aws: {}

  authorization:
    rbac: {}
```

Save and exit.

### Why?

This tells Kops to enable AWS IAM Authentication for this Kubernetes cluster.

Without this configuration, Kubernetes will continue accepting only certificate-based authentication.

---

# Step 3: Update the Cluster

After modifying the cluster configuration, update the cluster.

```bash
kops update cluster abhi.k8s.local --yes
```

### Sample Output

```
Cluster is starting rolling update.
```

### Why?

The configuration stored in the Kops State Store is only a desired configuration.

`kops update cluster` applies that configuration to AWS resources.

Without this command, the changes remain only in the State Store and are never applied to the running cluster.

---

# Step 4: Perform a Rolling Update

Apply the changes to the Kubernetes nodes.

```bash
kops rolling-update cluster abhi.k8s.local --yes
```

### Why?

Some cluster changes require recreating or updating the control plane and worker nodes.

A rolling update replaces the instances one by one so that the new configuration becomes active without completely shutting down the cluster.

---

# Note About Rolling Update

During our lab, we encountered the following error:

```text
Unable to reach the kubernetes API

dial tcp ... i/o timeout
```

This happened because the control plane was being recreated during the rolling update.

After waiting a few minutes for the new control plane instance to become Ready, the cluster became healthy again.

You can verify the cluster using:

```bash
kubectl get nodes
```

Example:

```text
NAME                    STATUS   ROLES
i-058707293523e5bc7     Ready    control-plane
i-04c1054d97d310728     Ready    node
```

---

# Step 5: Verify AWS Authentication is Enabled

Check the cluster configuration again.

```bash
kops get cluster abhi.k8s.local -o yaml
```

Expected output:

```yaml
authentication:
  aws: {}
```

### Why?

This confirms that AWS Authentication has been successfully enabled in the cluster configuration.

---

# Step 6: Verify aws-iam-authenticator Pod

List the pods in the `kube-system` namespace.

```bash
kubectl get pods -n kube-system
```

Initially, we observed:

```text
aws-iam-authenticator-xxxxx   0/1   ContainerCreating
```

### Why?

When AWS Authentication is enabled, Kops automatically deploys the **aws-iam-authenticator** DaemonSet.

This component is responsible for validating AWS IAM identities.

At this stage, the pod may still be starting because its required ConfigMap has not yet been created.

---

# Step 7: Investigate the Pending Pod

Describe the pod to identify the reason it is not running.

```bash
kubectl describe pod aws-iam-authenticator-xxxxx -n kube-system
```

In our lab, the Events section showed:

```text
MountVolume.SetUp failed for volume "config"

configmap "aws-iam-authenticator" not found
```

### Why?

The pod mounts a ConfigMap named:

```text
aws-iam-authenticator
```

Since the ConfigMap does not yet exist, Kubernetes cannot mount it into the container.

As a result:

- The container cannot start.
- The pod remains in the **ContainerCreating** state.

This is expected and will be fixed in the next part by creating the required ConfigMap.

---

# Summary

In this part, we:

- Checked the existing cluster configuration.
- Enabled AWS Authentication.
- Updated the cluster configuration.
- Performed a rolling update.
- Verified that AWS Authentication was enabled.
- Observed the `aws-iam-authenticator` pod.
- Investigated why the pod remained in `ContainerCreating`.
- Identified the root cause: **Missing ConfigMap**.

---

## What's Next?

In **Part 3**, we will:

- Create the `aws-auth.yml` ConfigMap.
- Apply the ConfigMap.
- Restart the `aws-iam-authenticator` pod.
- Verify that the pod changes from `ContainerCreating` to `Running`.
- Understand how IAM users are mapped to Kubernetes identities.

# Part 3: Configure aws-iam-authenticator (IAM User Mapping)

---

# Objective

In the previous part, we enabled AWS IAM Authentication in our Kops cluster.

During verification, we noticed that the **aws-iam-authenticator** pod was stuck in the **ContainerCreating** state.

After describing the pod, we discovered the root cause:

```text
MountVolume.SetUp failed for volume "config"

configmap "aws-iam-authenticator" not found
```

This happened because the authenticator pod requires a ConfigMap that contains the mapping between **AWS IAM Users** and **Kubernetes Users**.

In this part, we will create that ConfigMap and make the authenticator functional.

---

# Understanding the Problem

The authenticator pod mounts a ConfigMap named:

```text
aws-iam-authenticator
```

Inside this ConfigMap, Kubernetes stores information such as:

- Which AWS IAM User is allowed
- What Kubernetes username it becomes
- Which Kubernetes groups it belongs to

Without this ConfigMap, the authenticator has no information about valid IAM users.

Therefore, the pod cannot start.

---

# Authentication Flow

```
AWS IAM User
      │
      ▼
aws-iam-authenticator
      │
Reads ConfigMap
      │
      ▼
Maps IAM User
      │
      ▼
Kubernetes Username
      │
      ▼
RBAC Authorization
```

---

# Step 1: Create aws-auth.yml

Create the configuration file.

```bash
vim aws-auth.yml
```

---

# Step 2: Add the Configuration

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: aws-iam-authenticator
  namespace: kube-system

data:
  config.yaml: |
    clusterID: abhi.k8s.local

    mapUsers:
      - userarn: arn:aws:iam::144410074489:user/Ramdev
        username: Ramdev
        groups:
          - developers
```

---

# Understanding Every Field

## apiVersion

```yaml
apiVersion: v1
```

Specifies the Kubernetes API version.

---

## Kind

```yaml
kind: ConfigMap
```

Creates a ConfigMap object.

---

## Metadata

```yaml
metadata:
  name: aws-iam-authenticator
```

The name **must exactly match** the ConfigMap expected by the authenticator pod.

If the name is different, the pod cannot mount it.

---

## Namespace

```yaml
namespace: kube-system
```

The authenticator pod runs inside the **kube-system** namespace.

Therefore, the ConfigMap must also exist in the same namespace.

---

## clusterID

```yaml
clusterID: abhi.k8s.local
```

This must match your Kops cluster name.

Example:

```text
abhi.k8s.local
```

---

## mapUsers

```yaml
mapUsers:
```

This section maps AWS IAM Users to Kubernetes Users.

You can map one user or multiple users.

---

## userarn

```yaml
userarn: arn:aws:iam::144410074489:user/Ramdev
```

This is the AWS IAM User ARN.

Whenever this IAM User authenticates successfully, Kubernetes recognizes it.

---

## username

```yaml
username: Ramdev
```

This becomes the Kubernetes username.

Later, RBAC refers to this username.

For example:

```yaml
subjects:

- kind: User
  name: Ramdev
```

Notice that this name matches exactly.

---

## groups

```yaml
groups:
  - developers
```

This assigns the Kubernetes group.

Groups are useful when granting permissions to multiple users at once.

Although we will use **RoleBinding** directly with the user in this practical, group mapping is still considered a best practice.

---

# Step 3: Create the ConfigMap

Apply the configuration.

```bash
kubectl apply -f aws-auth.yml
```

Expected Output

```text
configmap/aws-iam-authenticator created
```

---

# Step 4: Verify ConfigMap

```bash
kubectl get configmap -n kube-system
```

Example

```text
NAME

aws-iam-authenticator
```

You can also inspect it.

```bash
kubectl get configmap aws-iam-authenticator -n kube-system -o yaml
```

Verify that the data contains:

```yaml
clusterID: abhi.k8s.local

mapUsers:
```

---

# Step 5: Check the Pod

```bash
kubectl get pods -n kube-system
```

Now the pod should become

```text
Running
```

Sometimes the pod still remains in the old state because it was created before the ConfigMap existed.

---

# Step 6: Restart the Pod

Delete the existing pod.

```bash
kubectl delete pod aws-iam-authenticator-xxxxx -n kube-system
```

Do not worry.

The pod belongs to a **DaemonSet**.

Therefore, Kubernetes automatically creates a new one.

---

# Step 7: Verify Again

```bash
kubectl get pods -n kube-system
```

Expected Output

```text
aws-iam-authenticator-xxxxx    1/1 Running
```

Now the authenticator is successfully running.

---

# What Changed?

Before:

```
aws-iam-authenticator

↓

ConfigMap Missing

↓

ContainerCreating
```

After:

```
ConfigMap Created

↓

Pod Restarted

↓

ConfigMap Mounted

↓

Pod Running
```

---

# Why Do We Need This ConfigMap?

Without this ConfigMap:

❌ Kubernetes does not know which IAM Users are allowed.

Without it:

- IAM User cannot authenticate.
- Kubernetes cannot map IAM User.
- RBAC cannot be executed.

This ConfigMap acts as the bridge between **AWS IAM** and **Kubernetes RBAC**.

---

# Summary

In this part, we:

- Understood why the authenticator pod was pending.
- Learned the purpose of the ConfigMap.
- Created the `aws-auth.yml` file.
- Mapped the AWS IAM User (`Ramdev`) to Kubernetes.
- Applied the ConfigMap.
- Restarted the authenticator pod.
- Verified that the pod was running successfully.

At this stage, AWS IAM Authentication is enabled, and the cluster now knows **which IAM user can be recognized**.

However, the IAM user still **does not have permission to perform Kubernetes operations**.

Authentication is complete, but Authorization is still pending.

---

## What's Next?

In **Part 4**, we will:

- Create a Namespace.
- Create a Role.
- Create a RoleBinding.
- Grant namespace-specific permissions to the IAM user.
- Understand how RBAC controls access after successful authentication.

# Part 4: Configure RBAC (Role & RoleBinding) for IAM User

---

# Objective

In the previous part, we configured the `aws-iam-authenticator` and mapped our AWS IAM User to Kubernetes.

At this point:

- Kubernetes can identify the IAM User.
- Authentication is complete.

However, the user still has **no permissions** inside the cluster.

This is because Kubernetes RBAC (Role-Based Access Control) has not yet been configured.

In this part, we will:

- Create a Namespace.
- Create a Role.
- Create a RoleBinding.
- Grant limited access to the IAM user.
- Understand how Kubernetes Authorization works.

---

# Authentication vs Authorization

Remember:

```
IAM User
    │
Authentication
    │
    ▼
Kubernetes knows who you are
    │
Authorization (RBAC)
    │
    ▼
Allowed / Denied
```

Authentication identifies the user.

Authorization decides what the user can do.

---

# Step 1: Create a Namespace

Create a namespace where the IAM user will have access.

```bash
kubectl create namespace devlop
```

Expected Output

```text
namespace/devlop created
```

Verify:

```bash
kubectl get ns
```

Example:

```text
NAME

default
devlop
kube-system
```

### Why?

Instead of giving access to the entire cluster, we restrict the user to a specific namespace.

This follows the **Principle of Least Privilege**, which grants only the permissions required to perform a task.

---

# Step 2: Create a Role

Create the Role definition.

```bash
vim role.yml
```

Add the following content:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: developer-role
  namespace: devlop

rules:
- apiGroups: [""]
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
```

---

# Understanding the Role

## apiVersion

```yaml
apiVersion: rbac.authorization.k8s.io/v1
```

Defines the RBAC API version.

---

## Kind

```yaml
kind: Role
```

A Role grants permissions **only within a single namespace**.

---

## Metadata

```yaml
metadata:
  name: developer-role
```

This is the name of the Role.

---

## Namespace

```yaml
namespace: devlop
```

The Role applies only inside the `devlop` namespace.

It has no effect on any other namespace.

---

## Rules

```yaml
rules:
```

Defines what actions are allowed.

---

## apiGroups

```yaml
apiGroups: [""]
```

An empty string (`""`) represents the **Core API Group**.

Pods belong to the Core API Group.

---

## Resources

```yaml
resources:
- pods
```

Specifies that the permissions apply only to Pods.

---

## Verbs

```yaml
verbs:
- get
- list
- watch
```

These permissions allow the user to:

- Get Pod details
- List Pods
- Watch Pod changes

Notice that we did **not** include:

- create
- delete
- update
- patch

So the user cannot modify Pods.

---

# Step 3: Create the Role

Apply the Role.

```bash
kubectl apply -f role.yml
```

Expected Output

```text
role.rbac.authorization.k8s.io/developer-role created
```

Verify:

```bash
kubectl get role -n devlop
```

---

# Step 4: Create the RoleBinding

Create the RoleBinding definition.

```bash
vim role-binding.yml
```

Add:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: developer-binding
  namespace: devlop

subjects:
- kind: User
  name: Ramdev
  apiGroup: rbac.authorization.k8s.io

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer-role
```

---

# Understanding the RoleBinding

## Kind

```yaml
kind: RoleBinding
```

A RoleBinding connects a Role with a User, Group, or ServiceAccount.

---

## Subject

```yaml
subjects:
```

Defines **who** receives the permissions.

---

### kind

```yaml
kind: User
```

The subject is a Kubernetes User.

---

### name

```yaml
name: Ramdev
```

This **must exactly match** the username defined in the `aws-auth.yml` ConfigMap.

Example:

```yaml
mapUsers:
- userarn: arn:aws:iam::144410074489:user/Ramdev
  username: Ramdev
```

If the names do not match, RBAC will not work.

---

## roleRef

```yaml
roleRef:
```

Specifies which Role should be assigned.

```yaml
kind: Role
name: developer-role
```

This binds the `developer-role` Role to the `Ramdev` user.

---

# Step 5: Create the RoleBinding

Apply it.

```bash
kubectl apply -f role-binding.yml
```

Expected Output

```text
rolebinding.rbac.authorization.k8s.io/developer-binding created
```

Verify:

```bash
kubectl get rolebinding -n devlop
```

---

# Step 6: Create a Test Pod

Create a Pod inside the `devlop` namespace.

```bash
kubectl run pod1 \
--image=nginx \
-n devlop
```

Verify:

```bash
kubectl get pods -n devlop
```

Example:

```text
NAME

pod1
```

### Why?

We need a resource inside the namespace so that the IAM user can test the granted permissions.

---

# Complete RBAC Flow

```
IAM User (Ramdev)
          │
          ▼
AWS IAM Authentication
          │
          ▼
aws-iam-authenticator
          │
          ▼
Mapped to Kubernetes User "Ramdev"
          │
          ▼
RoleBinding
          │
          ▼
developer-role
          │
          ▼
Permissions

GET Pods
LIST Pods
WATCH Pods
```

---

# Summary

In this part, we:

- Created a namespace.
- Created a Role.
- Granted Pod read permissions.
- Created a RoleBinding.
- Linked the IAM User (`Ramdev`) with the Role.
- Created a test Pod for permission verification.

At this stage, the Kubernetes RBAC configuration is complete.

The IAM user is now authenticated and has limited permissions within the `devlop` namespace.

---

# What's Next?

In **Part 5**, we will:

- Switch to the IAM User credentials.
- Configure the AWS CLI.
- Export an IAM-based kubeconfig.
- Verify the authenticated identity.
- Test whether the IAM user can access only the `devlop` namespace.
- Troubleshoot common authentication and authorization issues encountered during the lab.

# Part 5: Testing AWS IAM Authentication and Troubleshooting

---

# Objective

In the previous parts, we:

- Enabled AWS Authentication.
- Configured the `aws-iam-authenticator`.
- Created an IAM User.
- Mapped the IAM User to Kubernetes.
- Configured RBAC using Role and RoleBinding.

Now it's time to verify whether the IAM user can access the Kubernetes cluster.

---

# Step 1: Configure AWS CLI for the IAM User

Configure the AWS CLI using the IAM user's Access Key and Secret Key.

```bash
aws configure
```

Example:

```text
AWS Access Key ID: <IAM User Access Key>

AWS Secret Access Key: <IAM User Secret Key>

Default Region: ap-south-1

Output Format: json
```

---

# Step 2: Verify IAM Identity

Before testing Kubernetes access, verify that AWS CLI is using the correct IAM user.

```bash
aws sts get-caller-identity
```

Example Output

```json
{
  "UserId": "...",
  "Account": "144410074489",
  "Arn": "arn:aws:iam::144410074489:user/Ramdev"
}
```

### Why?

This confirms that AWS CLI is authenticated as the intended IAM user.

---

# Step 3: Generate IAM-Based kubeconfig

Export a kubeconfig using the authentication plugin.

```bash
kops export kubecfg \
abhi.k8s.local \
--auth-plugin \
--kubeconfig=/root/.kube/config.iam
```

### Why?

This generates a separate kubeconfig that uses the Kops authentication plugin instead of the default kubeconfig.

---

# Step 4: Use the New kubeconfig

```bash
export KUBECONFIG=/root/.kube/config.iam
```

Verify:

```bash
kubectl config view --minify
```

You should observe an **exec** section similar to:

```yaml
users:
- name: abhi.k8s.local
  user:
    exec:
      command: kops
```

This indicates that kubectl will execute the Kops authentication plugin whenever it needs credentials.

---

# Step 5: Verify the Authenticated User

Run:

```bash
kubectl auth whoami
```

---

# Expected Behaviour

If IAM Authentication is working correctly, the username should match the IAM user mapped inside the ConfigMap.

Example:

```text
Username: Ramdev
```

---

# What Happened in Our Lab?

Instead of the expected result, we observed:

```text
Username: kubecfg-root

Groups:
system:masters
```

This indicated that the request was still authenticated using a **temporary client certificate** instead of the AWS IAM user.

---

# How Did We Confirm It?

We executed:

```bash
kops helpers kubectl-auth \
--cluster=abhi.k8s.local \
--state=s3://abhis.kops.v1
```

The output contained:

```text
clientCertificateData

clientKeyData
```

This proved that the authentication plugin was returning an **X.509 Client Certificate**, not an AWS IAM authentication token.

Because of this, Kubernetes authenticated us as:

```text
kubecfg-root
```

instead of:

```text
Ramdev
```

---

# Why Is This Important?

Even though we had:

- Enabled AWS Authentication
- Created the ConfigMap
- Mapped the IAM User
- Configured RBAC

the authentication plugin generated a temporary Kubernetes client certificate.

Therefore, Kubernetes never evaluated the IAM user mapping.

---

# ConfigMap Verification

We verified that the ConfigMap was correctly configured.

```bash
kubectl get cm aws-iam-authenticator \
-n kube-system \
-o yaml
```

Configuration:

```yaml
mapUsers:
- userarn: arn:aws:iam::144410074489:user/Ramdev
  username: Ramdev
  groups:
  - developers
```

This confirmed that the server-side configuration was correct.

---

# Additional Issue Encountered

While testing the IAM user, we received:

```text
AccessDenied

s3:GetObject
```

The IAM user did not have permission to read the Kops State Store.

---

# Solution

We created an IAM Policy that granted read-only access to the Kops State Store and attached it to the IAM user.

After attaching the policy, the IAM user was able to access the State Store successfully.

---

# Important Learning

During this practical, we confirmed the following:

✅ AWS Authentication was enabled.

✅ aws-iam-authenticator was running.

✅ IAM User mapping existed.

✅ RBAC objects were created.

However, the Kops authentication plugin generated a temporary client certificate (`kubecfg-root`) instead of authenticating as the mapped IAM user.

As a result, RBAC for the IAM user was not exercised because Kubernetes authenticated the request as the cluster administrator.

This behavior may vary depending on the Kops version or authentication plugin implementation.

---

# Troubleshooting Checklist

If AWS IAM Authentication does not work, verify the following:

- Is `authentication: aws: {}` enabled in the cluster configuration?
- Is the `aws-iam-authenticator` pod running?
- Does the ConfigMap exist in the `kube-system` namespace?
- Is the IAM User ARN correct?
- Does the `username` in the ConfigMap match the RoleBinding subject?
- Does the IAM user have permission to read the Kops State Store?
- Is the correct kubeconfig being used?
- Does `kubectl auth whoami` show the expected username?

---

# Complete Authentication and Authorization Flow

```
AWS IAM User
        │
        ▼
AWS CLI
        │
        ▼
Authentication Plugin
        │
        ▼
aws-iam-authenticator
        │
        ▼
Kubernetes API Server
        │
Authentication Successful
        │
        ▼
RoleBinding
        │
        ▼
Role
        │
        ▼
Access Granted / Access Denied
```

---

# Summary

In this guide, we learned:

- Difference between Authentication and Authorization.
- How to enable AWS Authentication in a Kops cluster.
- Purpose of `aws-iam-authenticator`.
- How IAM users are mapped to Kubernetes.
- How RBAC controls access after successful authentication.
- How to configure Roles and RoleBindings for IAM users.
- Common troubleshooting steps for AWS IAM Authentication.
- Real-world debugging of authentication and kubeconfig issues encountered during the lab.

This completes the AWS IAM Authentication and RBAC Authorization practical.

# Part 6: Interview Questions, Best Practices & Final Revision

---

# Frequently Asked Interview Questions

## Q1. What is Authentication in Kubernetes?

Authentication is the process of verifying **who the user is**.

It answers the question:

> **Who are you?**

Examples:

- Client Certificate
- AWS IAM
- OIDC
- LDAP

---

## Q2. What is Authorization in Kubernetes?

Authorization determines **what an authenticated user is allowed to do**.

It answers the question:

> **What can you do?**

Kubernetes commonly uses **RBAC (Role-Based Access Control)** for authorization.

---

## Q3. What is the difference between Authentication and Authorization?

| Authentication | Authorization |
|---------------|---------------|
| Verifies identity | Verifies permissions |
| Happens first | Happens after authentication |
| Who are you? | What can you do? |
| IAM, Certificates | RBAC |

---

## Q4. What is aws-iam-authenticator?

`aws-iam-authenticator` is a component that allows Kubernetes to authenticate AWS IAM Users.

It acts as a bridge between:

```
AWS IAM
      │
      ▼
aws-iam-authenticator
      │
      ▼
Kubernetes API Server
```

---

## Q5. Why do we need aws-iam-authenticator?

Kubernetes cannot understand AWS IAM Users directly.

The authenticator:

- Validates the AWS IAM identity.
- Maps the IAM User to a Kubernetes username.
- Sends the authenticated username to the Kubernetes API Server.

---

## Q6. What is the purpose of the aws-iam-authenticator ConfigMap?

The ConfigMap stores the mapping between:

- AWS IAM User ARN
- Kubernetes Username
- Kubernetes Groups

Example:

```yaml
mapUsers:
- userarn: arn:aws:iam::144410074489:user/Ramdev
  username: Ramdev
  groups:
  - developers
```

---

## Q7. What happens if the ConfigMap is missing?

The `aws-iam-authenticator` Pod cannot mount its required configuration.

Typical error:

```text
MountVolume.SetUp failed

configmap "aws-iam-authenticator" not found
```

As a result, the Pod remains in the **ContainerCreating** state.

---

## Q8. Does Authentication provide permissions?

No.

Authentication only verifies identity.

Permissions are provided by **RBAC**.

---

## Q9. Which RBAC objects are used in this practical?

- Namespace
- Role
- RoleBinding

---

## Q10. Why did we create a Role instead of a ClusterRole?

Because we wanted to give access only inside a specific namespace (`devlop`).

A Role is namespace-scoped.

---

## Q11. Why did we create a RoleBinding?

A Role only defines permissions.

A RoleBinding assigns those permissions to a User, Group, or ServiceAccount.

---

## Q12. Why must the username in RoleBinding match the username in aws-auth.yml?

Example:

```yaml
mapUsers:
- username: Ramdev
```

RoleBinding:

```yaml
subjects:
- kind: User
  name: Ramdev
```

Both names must match exactly; otherwise, RBAC will not apply to that user.

---

## Q13. What is the purpose of groups in aws-auth.yml?

Groups allow multiple users to receive the same permissions.

Instead of binding every user individually, a RoleBinding can target a group.

---

## Q14. Why did the IAM user need permission to the Kops State Store?

The authentication plugin needed to read the Kops cluster configuration stored in the S3 State Store.

Without access, authentication failed with an `AccessDenied` error.

---

## Q15. How can you verify which identity Kubernetes is using?

```bash
kubectl auth whoami
```

This displays the authenticated username and groups.

---

# Best Practices

- Use IAM Users or IAM Roles instead of long-lived Kubernetes client certificates when possible.
- Follow the Principle of Least Privilege.
- Grant namespace-level access whenever possible.
- Use Groups to manage permissions for multiple users.
- Keep the `aws-iam-authenticator` ConfigMap secure.
- Regularly audit IAM users and RBAC permissions.
- Avoid granting `system:masters` unless absolutely necessary.

---

# Common Troubleshooting Checklist

If authentication or authorization does not work, verify:

- `authentication: aws: {}` is enabled.
- `aws-iam-authenticator` Pod is Running.
- ConfigMap exists in the `kube-system` namespace.
- IAM User ARN is correct.
- Username in ConfigMap matches the RoleBinding subject.
- IAM user has access to the Kops State Store (if required).
- Correct kubeconfig is in use.
- `kubectl auth whoami` shows the expected identity.

---

# Complete Request Flow

```
AWS IAM User
      │
      ▼
AWS CLI
      │
      ▼
Authentication Plugin
      │
      ▼
aws-iam-authenticator
      │
      ▼
Kubernetes API Server
      │
      ▼
Authentication Successful
      │
      ▼
RoleBinding
      │
      ▼
Role
      │
      ▼
Allow / Deny Request
```

---

# Key Takeaways

- Authentication verifies **who you are**.
- Authorization determines **what you can do**.
- AWS IAM Authentication integrates AWS identities with Kubernetes.
- `aws-iam-authenticator` bridges AWS IAM and Kubernetes.
- RBAC controls permissions after authentication.
- Roles define permissions, and RoleBindings assign them.
- Namespace-scoped access is achieved using **Role + RoleBinding**.
- Always verify your authenticated identity before troubleshooting RBAC.

---

# Final Revision

Remember the complete sequence:

```
Enable AWS Authentication
        │
        ▼
Deploy aws-iam-authenticator
        │
        ▼
Create aws-auth ConfigMap
        │
        ▼
Map IAM User
        │
        ▼
Create Namespace
        │
        ▼
Create Role
        │
        ▼
Create RoleBinding
        │
        ▼
Authenticate IAM User
        │
        ▼
RBAC Authorization
        │
        ▼
Access Granted / Denied
```

---

## Congratulations!

You have successfully completed the **AWS IAM Authentication and RBAC Authorization** topic. You now understand how AWS IAM identities are integrated with Kubernetes and how RBAC is used to control what authenticated users are allowed to do inside the cluster.
