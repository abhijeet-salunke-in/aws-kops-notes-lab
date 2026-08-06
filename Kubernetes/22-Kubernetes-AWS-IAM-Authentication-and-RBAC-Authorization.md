# AWS IAM Authentication in Kubernetes (kOps)

## 📌 Objective

The objective of this lab is to authenticate an **AWS IAM User** with a **Kubernetes cluster** created using **kOps**.

Instead of using Kubernetes client certificates, we will allow an AWS IAM user to authenticate with the Kubernetes API Server using **AWS IAM Authentication**. After successful authentication, Kubernetes **RBAC (Role & RoleBinding)** will be used to authorize what the user is allowed to do inside the cluster.

---

# What is AWS IAM Authentication?

AWS IAM Authentication allows AWS IAM Users or IAM Roles to log in to a Kubernetes cluster without creating Kubernetes certificates for every user.

The user proves their identity using AWS IAM credentials (Access Key ID and Secret Access Key). Kubernetes verifies this identity through the **aws-iam-authenticator** component and then applies Kubernetes RBAC permissions.

---

# Why Do We Need AWS IAM Authentication?

In the previous lab, we authenticated users using **Client Certificates**.

Although client certificates work well, they become difficult to manage when:

- The organization has many users.
- Users frequently join or leave the organization.
- Certificate rotation becomes difficult.
- The company already manages users in AWS IAM.

Instead of creating Kubernetes certificates for every employee, we can simply create an AWS IAM User and grant Kubernetes access.

---

# Authentication vs Authorization

Authentication and Authorization are two completely different processes.

## Authentication (Who are you?)

Authentication verifies the identity of the user.

Examples:

- Client Certificate Authentication
- AWS IAM Authentication
- OIDC Authentication
- Service Account Authentication

Example:

```
IAM User:
Ramdev
```

Kubernetes verifies whether this IAM User is genuine.

---

## Authorization (What are you allowed to do?)

After authentication succeeds, Kubernetes checks what the authenticated user is allowed to do.

Authorization is handled using Kubernetes RBAC objects such as:

- Role
- ClusterRole
- RoleBinding
- ClusterRoleBinding

Example:

```
Ramdev

↓

Can:
✔ Get Pods
✔ List Pods

Cannot:
✘ Delete Pods
✘ Create Deployments
```

---

# Authentication Flow

```
AWS IAM User
       │
       ▼
AWS Access Key
       │
       ▼
aws-iam-authenticator
       │
       ▼
Kubernetes API Server
       │
(Authentication Successful)
       │
       ▼
RBAC
(Role / RoleBinding)
       │
       ▼
Allowed Operations
```

---

# Lab Architecture

```
                 AWS IAM

             +----------------+
             | IAM User       |
             | Ramdev         |
             +----------------+
                     │
             Access Key / Secret Key
                     │
                     ▼
      +-------------------------------+
      | aws-iam-authenticator         |
      | (Running inside Kubernetes)   |
      +-------------------------------+
                     │
                     ▼
          Kubernetes API Server
                     │
          Authentication Successful
                     │
                     ▼
      Role + RoleBinding (RBAC)
                     │
                     ▼
          Namespace: devlop
                     │
                     ▼
           Allowed Kubernetes Actions
```

---

# Lab Workflow

During this lab we will perform the following steps:

1. Check whether AWS Authentication is enabled.
2. Create an AWS IAM User.
3. Generate Access Keys for the IAM User.
4. Enable AWS Authentication in the kOps cluster.
5. Update the Kubernetes cluster.
6. Verify the AWS IAM Authenticator Pod.
7. Troubleshoot the Pod if it is not running.
8. Create the Authentication ConfigMap.
9. Restart the Authenticator Pod.
10. Verify the Pod is running.
11. Create a Namespace.
12. Create a Role.
13. Create a RoleBinding.
14. Test authentication using the IAM User.
15. Configure AWS CLI for the IAM User.
16. Verify the authenticated identity.
17. Test RBAC permissions.

---

# Prerequisites

Before starting this lab, make sure you have:

- Running Kubernetes Cluster created using kOps
- AWS CLI installed
- kubectl installed
- kOps installed
- Root/Admin access to the cluster
- Access to the kOps State Store (S3 Bucket)
- Internet connectivity

---

> **Note**
>
> In this lab, **AWS IAM** is used only for **Authentication**, whereas **Kubernetes RBAC** is responsible for **Authorization**.
>
> Authentication answers:
>
> **"Who are you?"**
>
> Authorization answers:
>
> **"What are you allowed to do?"**

# Part 2: Enable AWS Authentication in the Cluster

In this section, we will prepare our Kubernetes cluster to use **AWS IAM Authentication**.

At this stage, users are still authenticated using **Client Certificates**. We will enable AWS Authentication so that IAM Users can authenticate to the Kubernetes API Server.

---

# Step 1: Check Whether AWS Authentication is Enabled

Before making any changes, first verify whether AWS Authentication is already enabled in the cluster.

### Command

```bash
kops get cluster abhi.k8s.local -o yaml
```

Example Output

```yaml
spec:
  authentication:
    aws: {}
```

If the `authentication` section is not present, AWS Authentication is **not enabled**.

---

## Why?

Before enabling any feature, we should always verify the current cluster configuration.

This prevents unnecessary configuration changes and helps us understand the current state of the cluster.

---

# Step 2: Create an AWS IAM User

Create a new IAM User that will later authenticate to the Kubernetes cluster.

### Command

```bash
aws iam create-user --user-name Ramdev
```

Example Output

```json
{
    "User": {
        "UserName": "Ramdev",
        "Arn": "arn:aws:iam::<ACCOUNT-ID>:user/Ramdev"
    }
}
```

---

## Why?

Kubernetes does not authenticate IAM users automatically.

First, an IAM User must exist in AWS before we can map that user to Kubernetes.

---

# Step 3: Generate Access Keys

Create an Access Key for the IAM User.

### Command

```bash
aws iam create-access-key --user-name Ramdev
```

Example Output

```json
{
    "AccessKey": {
        "AccessKeyId": "AKIAxxxxxxxx",
        "SecretAccessKey": "xxxxxxxxxxxxxxxx"
    }
}
```

> **Important**
>
> Save the **Access Key ID** and **Secret Access Key** safely.
>
> The Secret Access Key is shown **only once** by AWS.

---

## Why?

The IAM User will use these credentials to authenticate with AWS.

Later, the AWS CLI will use these credentials to obtain a temporary authentication token for Kubernetes.

---

# Step 4: Enable AWS Authentication

Edit the cluster configuration.

### Command

```bash
kops edit cluster abhi.k8s.local
```

Locate the API configuration and add the following section.

```yaml
spec:
  api:
    loadBalancer:
      type: Public

  authentication:
    aws: {}
```

Save and exit.

---

## Why?

By default, the Kubernetes API Server only accepts certificate-based authentication.

Adding

```yaml
authentication:
  aws: {}
```

tells **kOps** to enable AWS IAM Authentication.

After applying this configuration, kOps deploys the **aws-iam-authenticator** component inside the cluster.

This component is responsible for validating AWS IAM Users.

---

# Step 5: Update the Cluster

The changes made in the cluster configuration are only stored in the kOps State Store.

They are **not yet applied** to the running cluster.

Apply the changes.

### Command

```bash
kops update cluster abhi.k8s.local --yes
```

Example Output

```
Cluster is starting rolling update...
```

---

Next, perform a rolling update.

### Command

```bash
kops rolling-update cluster abhi.k8s.local --yes
```

This recreates the nodes one by one with the updated configuration.

---

## Verify the Cluster

After the rolling update completes, verify that all nodes are ready.

```bash
kubectl get nodes
```

Example Output

```text
NAME                  STATUS   ROLES           AGE
i-xxxxxxxxxxxx        Ready    control-plane
i-yyyyyyyyyyyy        Ready    node
```

---

You can also validate the cluster.

```bash
kops validate cluster
```

Initially, you may see validation errors similar to:

```text
VALIDATION ERRORS

Pod kube-system/aws-iam-authenticator is pending
```

This is expected because the **aws-iam-authenticator** pod has been created but is not yet fully configured.

We will fix this in the next section.

---

## Why Update the Cluster?

Editing the cluster configuration only updates the **desired state** stored in the kOps State Store (S3).

`kops update cluster` applies those changes to AWS resources.

`kops rolling-update cluster` recreates the instances so they start using the new configuration.

Without these commands, AWS Authentication will **not** be enabled.

---

## Summary

At this stage, we have completed the following:

- Verified the current authentication configuration.
- Created an AWS IAM User.
- Generated Access Keys.
- Enabled AWS Authentication in the cluster configuration.
- Updated the cluster.
- Performed a rolling update.
- Verified the cluster status.

In the next section, we will verify whether the **aws-iam-authenticator** pod is running and troubleshoot it if it remains in the **Pending** state.

# Part 3: Configure AWS IAM Authenticator

After enabling AWS Authentication and updating the cluster, kOps deploys the **aws-iam-authenticator** Pod.

Its job is to authenticate AWS IAM Users before allowing them to access the Kubernetes API Server.

However, immediately after deployment, the Pod may remain in the **Pending** or **ContainerCreating** state because the required ConfigMap has not yet been created.

In this section, we will troubleshoot the issue and configure the authenticator.

---

# Step 6: Check the AWS IAM Authenticator Pod

First, verify whether the AWS IAM Authenticator Pod is running.

### Command

```bash
kubectl get pods -n kube-system
```

Example Output

```text
NAME                                   READY   STATUS              AGE

aws-iam-authenticator-xxxxx            0/1     ContainerCreating   2m
```

or

```text
NAME                                   READY   STATUS     AGE

aws-iam-authenticator-xxxxx            0/1     Pending    2m
```

---

## Why?

Whenever a new component is deployed, we should verify whether it has started successfully.

If the Pod is not running, Kubernetes is indicating that something is preventing it from starting.

---

# Step 7: Describe the Pending Pod

If the Pod is stuck in the **Pending** or **ContainerCreating** state, inspect it to determine the root cause.

### Command

```bash
kubectl describe pod aws-iam-authenticator-xxxxx -n kube-system
```

Scroll to the **Events** section.

Example Output

```text
Events:

Warning  FailedMount

MountVolume.SetUp failed for volume "config":

configmap "aws-iam-authenticator" not found
```

---

## Root Cause

The Pod is trying to mount a ConfigMap named:

```text
aws-iam-authenticator
```

but Kubernetes cannot find it.

Without this ConfigMap, the authenticator has no configuration and therefore cannot start.

---

## Why?

The **aws-iam-authenticator** Pod reads its configuration from a ConfigMap.

That ConfigMap contains:

- Cluster Name
- IAM Users
- IAM Roles
- Kubernetes User Mapping
- Kubernetes Groups

Since the ConfigMap does not exist yet, Kubernetes cannot mount it inside the Pod.

As a result, the Pod remains stuck in the **ContainerCreating** state.

---

# Step 8: Create the Authentication ConfigMap

Now create the configuration file.

### Create File

```bash
vim aws-auth.yml
```

Add the following configuration.

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
      - userarn: arn:aws:iam::<ACCOUNT-ID>:user/Ramdev
        username: Ramdev
        groups:
          - developers
```

Save the file.

---

Apply the ConfigMap.

```bash
kubectl apply -f aws-auth.yml
```

Example Output

```text
configmap/aws-iam-authenticator created
```

Verify the ConfigMap.

```bash
kubectl get configmap -n kube-system
```

Example Output

```text
NAME

aws-iam-authenticator
```

---

## Why?

The ConfigMap provides the configuration required by the AWS IAM Authenticator Pod.

It tells Kubernetes:

- Which cluster this configuration belongs to.
- Which AWS IAM User is allowed.
- What Kubernetes username should be assigned.
- Which Kubernetes groups the user belongs to.

Without this ConfigMap, AWS IAM Authentication cannot work.

---

# Step 9: Verify the Pod Again

Check whether the Pod has started.

### Command

```bash
kubectl get pods -n kube-system
```

Sometimes the Pod is still not running because Kubernetes attempted to start it **before** the ConfigMap was created.

---

## Why?

Although the ConfigMap now exists, the existing Pod may still be using its previous failed state.

In such cases, restarting the Pod is the simplest solution.

---

# Step 10: Restart the AWS IAM Authenticator Pod

Delete the existing Pod.

```bash
kubectl delete pod aws-iam-authenticator-xxxxx -n kube-system
```

Example Output

```text
pod "aws-iam-authenticator-xxxxx" deleted
```

Since this Pod is managed by a **DaemonSet**, Kubernetes automatically creates a new one.

Verify the new Pod.

```bash
kubectl get pods -n kube-system
```

Example Output

```text
NAME                                   READY   STATUS    AGE

aws-iam-authenticator-yyyyy            1/1     Running   15s
```

---

## Why?

Deleting the Pod does **not** remove the application.

The DaemonSet controller immediately creates a new Pod.

This new Pod now finds the ConfigMap, mounts it successfully, loads the IAM User mappings, and starts normally.

---

# Summary

At this stage, we have successfully:

- Verified the AWS IAM Authenticator Pod.
- Identified why the Pod was stuck.
- Diagnosed the issue using `kubectl describe pod`.
- Found that the required ConfigMap was missing.
- Created the `aws-iam-authenticator` ConfigMap.
- Applied the ConfigMap.
- Restarted the Pod.
- Verified that the Pod is now running successfully.

The cluster is now capable of authenticating AWS IAM Users. In the next section, we will configure Kubernetes RBAC by creating a Namespace, Role, and RoleBinding for the IAM User.

# Part 4: Configure Kubernetes RBAC for the IAM User

AWS IAM Authentication only verifies the identity of the user.

It **does not grant any permissions** inside the Kubernetes cluster.

To allow the IAM User to perform Kubernetes operations, we must configure **Role-Based Access Control (RBAC)**.

In this section, we will:

- Create a Namespace.
- Create a Role.
- Create a RoleBinding.
- Create a test Pod.
- Switch to the IAM User.
- Verify the authenticated identity.
- Test the RBAC permissions.

---

# Step 11: Create a Namespace

Create a namespace where the IAM User will have access.

### Command

```bash
kubectl create namespace devlop
```

Verify the namespace.

```bash
kubectl get ns
```

Example Output

```text
NAME              STATUS

default           Active
devlop            Active
kube-system       Active
```

---

## Why?

Instead of granting permissions across the entire cluster, we are limiting the user's access to a specific namespace.

This follows the **Principle of Least Privilege**, giving users access only where it is required.

---

# Step 12: Create a Role and RoleBinding

## Create Role

Create the Role YAML.

```bash
vim role.yml
```

Example

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

## Create RoleBinding

Create the RoleBinding YAML.

```bash
vim role-binding.yml
```

Example

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
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

---

## Why?

The **Role** defines the permissions.

The **RoleBinding** assigns those permissions to a specific user.

Without a RoleBinding, the Role is never used.

---

# Step 13: Apply Both YAML Files

Create the Role.

```bash
kubectl apply -f role.yml
```

Create the RoleBinding.

```bash
kubectl apply -f role-binding.yml
```

Verify.

```bash
kubectl get role -n devlop

kubectl get rolebinding -n devlop
```

Example Output

```text
developer-role

developer-binding
```

---

## Create a Test Pod

Create a Pod inside the **devlop** namespace.

Example

```bash
kubectl run pod1 \
--image=nginx \
-n devlop
```

Verify.

```bash
kubectl get pods -n devlop
```

Example Output

```text
NAME

pod1
```

---

## Why?

The Pod is created so that we can later verify whether the IAM User can access it.

---

# Step 14: Verify the Current AWS Identity

Check which AWS identity is currently being used.

```bash
aws sts get-caller-identity
```

Example Output

```json
{
  "Arn":"arn:aws:iam::<ACCOUNT-ID>:user/kops"
}
```

or

```json
{
  "Arn":"arn:aws:iam::<ACCOUNT-ID>:user/Ramdev"
}
```

---

## Why?

Before testing permissions, always confirm which IAM User is currently authenticated.

Otherwise, you may unknowingly test using the wrong credentials.

---

# Step 15: Remove Existing AWS Credentials

Delete the current AWS CLI configuration.

```bash
rm -rf ~/.aws
```

---

## Why?

This removes any previously configured IAM credentials.

It allows us to configure the AWS CLI with the credentials of the IAM User we want to test.

---

# Step 16: Configure AWS CLI

Configure the AWS CLI using the Access Key created earlier.

```bash
aws configure
```

Enter:

```text
AWS Access Key ID

AWS Secret Access Key

Default Region

Output Format
```

Verify the identity.

```bash
aws sts get-caller-identity
```

Example Output

```json
{
    "Arn":"arn:aws:iam::<ACCOUNT-ID>:user/Ramdev"
}
```

---

## Why?

This confirms that all future AWS API requests will be performed using the IAM User **Ramdev**.

---

# Step 17: Test the RBAC Permissions

Now test whether the IAM User can access Kubernetes resources.

Check the namespace Pods.

```bash
kubectl get pods -n devlop
```

If the IAM User has been authenticated correctly and the RoleBinding is configured properly, the Pod list should be displayed.

Example Output

```text
NAME

pod1
```

Try accessing resources outside the assigned namespace.

Example

```bash
kubectl get pods -A
```

or

```bash
kubectl get ns
```

Depending on the RBAC permissions, these operations may return an error similar to:

```text
Error from server (Forbidden)
```

---

## Why?

This step verifies that:

- AWS IAM Authentication is working.
- The IAM User has been successfully mapped to a Kubernetes user.
- RBAC permissions are being enforced correctly.

Authentication confirms **who the user is**, while RBAC determines **what the user is allowed to do**.

---

# Summary

In this section, we:

- Created a dedicated namespace.
- Created a Role with limited permissions.
- Bound the Role to the IAM User using a RoleBinding.
- Created a test Pod.
- Verified the active AWS IAM identity.
- Reconfigured the AWS CLI for the IAM User.
- Tested Kubernetes access using RBAC.

At this point, the IAM User **Ramdev** can authenticate to the Kubernetes cluster using AWS IAM credentials and is authorized to perform only the actions permitted by the configured Role within the `devlop` namespace.

# Part 5: Troubleshooting IAM Authentication

During the practical, we observed an unexpected behavior.

Even after configuring the AWS IAM User, Role, and RoleBinding, the user was still able to perform administrator-level operations.

Instead of immediately assuming RBAC was incorrect, we investigated the authentication mechanism step by step.

---

# Problem Statement

After configuring the AWS IAM User and switching AWS credentials, we expected the IAM User to have only the permissions defined by the Role.

However, when we executed:

```bash
kubectl get pods -A
```

or

```bash
kubectl get ns
```

the commands succeeded instead of returning a **Forbidden** error.

This indicated that Kubernetes was **not authenticating us as the IAM User**.

---

# Step 18: Verify Current AWS Identity

First, verify which AWS IAM User is being used.

```bash
aws sts get-caller-identity
```

Example

```json
{
    "Arn":"arn:aws:iam::<ACCOUNT-ID>:user/Ramdev"
}
```

---

## Why?

This confirms that the AWS CLI is using the expected IAM User credentials.

If this is incorrect, Kubernetes authentication will also fail.

---

# Step 19: Verify the Current Kubernetes User

Run:

```bash
kubectl auth whoami
```

Example Output

```text
Username

kubecfg-root

Groups

system:masters
system:authenticated
```

---

## Expected Result

We expected:

```text
Username

Ramdev
```

because the IAM User **Ramdev** was mapped inside the ConfigMap.

---

## Actual Result

Instead of:

```
Ramdev
```

Kubernetes authenticated us as:

```
kubecfg-root
```

which belongs to the **system:masters** group.

This explains why every command was successful.

---

# Why Did This Happen?

Authentication was **not using AWS IAM**.

Instead, kubectl was still authenticating using a Kubernetes **Client Certificate**.

Since that certificate belonged to **kubecfg-root**, Kubernetes treated the request as coming from the cluster administrator.

RBAC for the IAM User was never evaluated.

---

# Step 20: Verify the Current kubeconfig

Display the active kubeconfig.

```bash
kubectl config view --minify
```

Example

```yaml
users:

- name: abhi.k8s.local

  user:

    client-certificate-data:

    client-key-data:
```

---

## Observation

The kubeconfig still contained:

- Client Certificate
- Client Key

instead of an authentication plugin.

This proved that kubectl was still using **certificate-based authentication**.

---

# Step 21: Backup the Existing kubeconfig

Before making changes, create a backup.

```bash
cp ~/.kube/config ~/.kube/config.admin
```

---

## Why?

This preserves the administrator kubeconfig.

If anything goes wrong, it can easily be restored.

---

# Step 22: Generate an IAM-Based kubeconfig

Generate a new kubeconfig using the kOps authentication plugin.

```bash
kops export kubecfg \
abhi.k8s.local \
--auth-plugin \
--kubeconfig=/root/.kube/config.iam
```

---

## Why?

Unlike the default kubeconfig, this configuration does not store client certificates.

Instead, it uses an **authentication plugin (exec plugin)**.

---

# Step 23: Verify the New kubeconfig

Open the generated file.

```bash
cat ~/.kube/config.iam
```

Example

```yaml
users:

- name: abhi.k8s.local

  user:

    exec:

      command: kops
```

---

## Observation

The new kubeconfig uses an **exec plugin**.

Whenever kubectl needs credentials, it executes:

```text
kops helpers kubectl-auth
```

instead of using a client certificate stored in the kubeconfig.

---

# Step 24: Switch to the New kubeconfig

Use the IAM kubeconfig.

```bash
export KUBECONFIG=/root/.kube/config.iam
```

Verify.

```bash
kubectl config view --minify
```

The output should display an **exec** section instead of **client-certificate-data**.

---

## Why?

This ensures kubectl uses the authentication plugin rather than the administrator certificate.

---

# Step 25: Verify the Authentication Plugin

Execute the helper manually.

```bash
kops helpers kubectl-auth \
--cluster=abhi.k8s.local \
--state=s3://<STATE-STORE>
```

Example Output

```text
clientCertificateData

clientKeyData
```

---

## Observation

Although we expected an AWS IAM authentication token, the plugin generated a **temporary client certificate**.

As a result, Kubernetes still authenticated the request as:

```text
kubecfg-root
```

instead of the mapped IAM User.

---

# Important Learning

Simply enabling AWS Authentication is **not enough**.

Always verify:

- Which AWS IAM User is active.
- Which kubeconfig is being used.
- Whether kubectl is using client certificates or an authentication plugin.
- Which username Kubernetes ultimately authenticates.

The command:

```bash
kubectl auth whoami
```

is one of the most useful troubleshooting commands because it shows the exact identity recognized by the Kubernetes API Server.

---

# Summary

During troubleshooting, we discovered:

- AWS CLI was using the correct IAM User.
- Kubernetes was **not** using the IAM User.
- kubectl was authenticating with a client certificate.
- The authenticated Kubernetes user was **kubecfg-root**.
- Since **kubecfg-root** belongs to **system:masters**, RBAC restrictions were bypassed.
- We generated an IAM-based kubeconfig using the kOps authentication plugin to continue the investigation.

In the next section, we will troubleshoot another issue encountered during the lab: **AccessDenied while reading the kOps State Store (S3)** and understand why the IAM User required additional S3 permissions.

# Part 6: Troubleshooting S3 Access & Final Verification

While testing AWS IAM Authentication, another issue was encountered.

Even after generating an IAM-based kubeconfig, Kubernetes authentication failed because the IAM User did not have permission to read the **kOps State Store**.

In this section, we will troubleshoot and resolve this issue.

---

# Step 26: Attempt to Access the Cluster

After switching to the IAM User and using the IAM kubeconfig, execute:

```bash
kubectl auth whoami
```

or

```bash
kubectl get pods -n devlop
```

Example Error

```text
AccessDenied

User:
arn:aws:iam::<ACCOUNT-ID>:user/Ramdev

is not authorized to perform:

s3:GetObject

on resource:

arn:aws:s3:::<STATE-STORE>/cluster/config
```

---

## Why Did This Happen?

The kubeconfig generated by kOps uses an authentication plugin.

Whenever kubectl sends a request, the plugin first reads the cluster configuration from the **kOps State Store (Amazon S3)**.

Since the IAM User had no permission to access the S3 bucket, authentication failed before Kubernetes could even verify the user.

---

# Step 27: Create an IAM Policy

Create a JSON file.

```bash
vim s3-read-policy.json
```

Example

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<STATE-STORE>",
        "arn:aws:s3:::<STATE-STORE>/*"
      ]
    }
  ]
}
```

Save the file.

---

## Why?

This policy grants the IAM User read-only access to the kOps State Store.

The user does **not** require write permissions because it only needs to read the cluster configuration.

---

# Step 28: Create the IAM Policy

Execute:

```bash
aws iam create-policy \
--policy-name KopsStateStoreReadOnly \
--policy-document file://s3-read-policy.json
```

Example Output

```text
PolicyName

KopsStateStoreReadOnly
```

---

## Why?

This creates a reusable IAM policy that can later be attached to any IAM User requiring Kubernetes authentication.

---

# Step 29: Attach the Policy to the IAM User

Attach the policy.

```bash
aws iam attach-user-policy \
--user-name Ramdev \
--policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/KopsStateStoreReadOnly
```

Verify.

```bash
aws iam list-attached-user-policies \
--user-name Ramdev
```

Example Output

```text
KopsStateStoreReadOnly
```

---

## Why?

Without this policy, the authentication plugin cannot read the cluster configuration stored in Amazon S3.

---

# Step 30: Configure AWS CLI Again

Switch back to the IAM User credentials if required.

```bash
aws configure
```

Verify the identity.

```bash
aws sts get-caller-identity
```

Expected Output

```json
{
  "Arn":"arn:aws:iam::<ACCOUNT-ID>:user/Ramdev"
}
```

---

# Step 31: Use the IAM kubeconfig

Switch to the kubeconfig generated with the authentication plugin.

```bash
export KUBECONFIG=/root/.kube/config.iam
```

Verify.

```bash
kubectl config view --minify
```

The kubeconfig should contain:

```yaml
exec:
  command: kops
```

instead of

```yaml
client-certificate-data
```

---

# Step 32: Verify Authentication

Execute

```bash
kubectl auth whoami
```

During our lab, the output was:

```text
Username

kubecfg-root

Groups

system:masters
```

---

## Important Observation

Even after:

- Enabling AWS Authentication
- Creating the ConfigMap
- Mapping the IAM User
- Creating the Role
- Creating the RoleBinding
- Using the authentication plugin
- Granting S3 access

the authentication plugin still generated a **temporary client certificate**, causing Kubernetes to authenticate the request as **kubecfg-root**.

As a result, the IAM User mapping was not exercised in our environment.

This behavior depends on the **kOps version**, cluster configuration, and authentication plugin implementation.

---

# Lessons Learned

During this practical, we learned:

- AWS IAM Authentication must be enabled in the cluster.
- The `aws-iam-authenticator` Pod must be running.
- The ConfigMap must correctly map IAM Users.
- IAM Users require valid AWS credentials.
- RBAC controls authorization after successful authentication.
- The kOps authentication plugin may require access to the S3 State Store.
- Always verify the authenticated Kubernetes identity using:

```bash
kubectl auth whoami
```

---

# Complete Authentication Flow

```text
AWS IAM User
        │
        ▼
AWS CLI Credentials
        │
        ▼
Authentication Plugin
        │
        ▼
Read Cluster Configuration
from kOps State Store (S3)
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
RBAC (Role / RoleBinding)
        │
        ▼
Access Granted / Access Denied
```

---

# Summary

In this lab, we successfully explored the complete workflow of AWS IAM Authentication with Kubernetes.

We learned how to:

- Enable AWS Authentication in a kOps cluster.
- Deploy and configure the `aws-iam-authenticator`.
- Map an AWS IAM User to a Kubernetes identity.
- Configure RBAC using Role and RoleBinding.
- Troubleshoot missing ConfigMaps and pending Pods.
- Resolve S3 `AccessDenied` issues by granting read-only access to the kOps State Store.
- Verify authenticated identities using `kubectl auth whoami`.
- Understand the interaction between AWS IAM Authentication and Kubernetes RBAC.

This concludes the practical implementation of AWS IAM Authentication with Kubernetes.

# Part 7: Interview Questions, Best Practices & Revision

This section summarizes the most important concepts covered in this lab and includes commonly asked interview questions.

---

# Frequently Asked Interview Questions

## 1. What is AWS IAM Authentication in Kubernetes?

AWS IAM Authentication allows AWS IAM Users or IAM Roles to authenticate to a Kubernetes cluster without using Kubernetes client certificates.

Instead of verifying certificates, Kubernetes verifies the user's AWS IAM identity through the **aws-iam-authenticator**.

---

## 2. What is the difference between Authentication and Authorization?

| Authentication | Authorization |
|----------------|---------------|
| Verifies the identity of the user | Determines what the user can do |
| Happens first | Happens after authentication |
| Example: AWS IAM | Example: RBAC |

---

## 3. Why do we need aws-iam-authenticator?

Kubernetes does not understand AWS IAM identities directly.

The **aws-iam-authenticator** acts as a bridge between AWS IAM and Kubernetes.

It:

- Validates AWS IAM credentials.
- Maps the IAM User to a Kubernetes username.
- Sends the authenticated identity to the Kubernetes API Server.

---

## 4. What is the purpose of the aws-auth ConfigMap?

The ConfigMap maps AWS IAM Users or IAM Roles to Kubernetes users and groups.

Example:

```yaml
mapUsers:
- userarn: arn:aws:iam::ACCOUNT-ID:user/Ramdev
  username: Ramdev
  groups:
  - developers
```

Without this mapping, Kubernetes does not know how to identify the IAM User.

---

## 5. Why do we create a Role?

A Role defines what actions are allowed within a specific namespace.

Example:

- Get Pods
- List Pods
- Watch Pods

---

## 6. Why do we create a RoleBinding?

A Role only defines permissions.

A RoleBinding assigns those permissions to a User, Group, or ServiceAccount.

Without a RoleBinding, the Role has no effect.

---

## 7. Why did we create a separate namespace?

To follow the **Principle of Least Privilege**.

Instead of granting cluster-wide permissions, the IAM User receives access only to the required namespace.

---

## 8. What happens if the ConfigMap is missing?

The aws-iam-authenticator Pod cannot start.

Typical error:

```text
MountVolume.SetUp failed

configmap "aws-iam-authenticator" not found
```

---

## 9. Why did we create an S3 Read-Only Policy?

The kOps authentication plugin reads the cluster configuration from the kOps State Store (Amazon S3).

Without permission, authentication fails with:

```text
AccessDenied

s3:GetObject
```

---

## 10. Which command tells you the Kubernetes identity being used?

```bash
kubectl auth whoami
```

This command displays:

- Username
- Groups
- Authentication details

It is one of the most useful commands for troubleshooting authentication issues.

---

# Best Practices

- Use AWS IAM Authentication instead of distributing Kubernetes client certificates to users.
- Grant only the minimum permissions required.
- Use namespace-scoped Roles whenever possible.
- Prefer Groups over assigning permissions directly to individual users.
- Keep the `aws-auth` ConfigMap updated.
- Regularly review IAM policies and RBAC permissions.
- Backup the administrator kubeconfig before making authentication changes.
- Verify the authenticated identity before troubleshooting authorization issues.

---

# Common Troubleshooting Checklist

If authentication is not working, verify the following:

- Is AWS Authentication enabled in the cluster?
- Is the `aws-iam-authenticator` Pod running?
- Does the `aws-auth` ConfigMap exist?
- Is the IAM User ARN correct?
- Does the mapped Kubernetes username match the RoleBinding subject?
- Is the IAM User using the correct AWS credentials?
- Does the IAM User have access to the kOps State Store?
- Is the correct kubeconfig being used?
- What does `kubectl auth whoami` display?

---

# Frequently Used Commands

## Check Cluster Configuration

```bash
kops get cluster abhi.k8s.local -o yaml
```

---

## Edit Cluster

```bash
kops edit cluster abhi.k8s.local
```

---

## Update Cluster

```bash
kops update cluster abhi.k8s.local --yes
```

---

## Rolling Update

```bash
kops rolling-update cluster abhi.k8s.local --yes
```

---

## Validate Cluster

```bash
kops validate cluster
```

---

## Check aws-iam-authenticator Pod

```bash
kubectl get pods -n kube-system
```

---

## Describe Pod

```bash
kubectl describe pod <pod-name> -n kube-system
```

---

## Apply ConfigMap

```bash
kubectl apply -f aws-auth.yml
```

---

## Verify AWS Identity

```bash
aws sts get-caller-identity
```

---

## Configure AWS CLI

```bash
aws configure
```

---

## Verify Kubernetes Identity

```bash
kubectl auth whoami
```

---

# Complete Authentication & Authorization Flow

```text
AWS IAM User
       │
       ▼
AWS CLI Credentials
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
Allow / Deny Request
```

---

# Key Takeaways

- AWS IAM Authentication verifies **who the user is**.
- Kubernetes RBAC determines **what the user can do**.
- The `aws-iam-authenticator` bridges AWS IAM and Kubernetes.
- The `aws-auth` ConfigMap maps IAM identities to Kubernetes users.
- Roles define permissions.
- RoleBindings assign permissions.
- Namespace-scoped Roles provide least-privilege access.
- Always verify the authenticated identity using `kubectl auth whoami`.
- Troubleshoot methodically by checking authentication before authorization.

---

# Conclusion

In this lab, we successfully explored how AWS IAM Authentication integrates with Kubernetes in a kOps-managed cluster.

We learned how to:

- Enable AWS Authentication.
- Configure the `aws-iam-authenticator`.
- Map AWS IAM Users using the `aws-auth` ConfigMap.
- Configure RBAC with Roles and RoleBindings.
- Troubleshoot common authentication issues.
- Understand the interaction between AWS IAM Authentication and Kubernetes Authorization.

By combining **AWS IAM Authentication** with **Kubernetes RBAC**, we can securely control access to cluster resources while leveraging centralized identity management provided by AWS.
