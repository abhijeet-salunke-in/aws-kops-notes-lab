# KOPS Kubernetes Cluster Setup on AWS

## Overview

KOPS (Kubernetes Operations) is an open-source tool used to create, manage, upgrade, and maintain production-grade Kubernetes clusters on AWS.

In this lab, an EC2 instance acts as a **KOPS Management Server** from which KOPS and kubectl are installed and used to provision a Kubernetes cluster on AWS.

---

# Architecture Overview

## What is a Kubernetes Cluster?

A Kubernetes Cluster is a collection of machines working together to run containerized applications.

KOPS automatically provisions and manages the required AWS infrastructure.

### Components Created by KOPS

```text
Cluster
│
├── Control Plane (Master)
│
├── Worker Nodes
│
├── VPC
├── Subnets
├── Security Groups
├── Route Tables
├── Load Balancer
├── Auto Scaling Groups
│
└── S3 State Store
```

---

# Prerequisites

- AWS Account
- EC2 Instance (Management Server)
- IAM User with Administrator Access
- AWS CLI Installed
- S3 Bucket for KOPS State Store
- SSH Key Pair

---

# Step 1: Launch EC2 Instance

## Purpose

This EC2 instance serves as the KOPS Management Server.

Responsibilities:

- Install KOPS
- Install kubectl
- Create Kubernetes Cluster
- Manage Kubernetes Cluster

### Configuration

```text
Name            : kops-server
Region          : ap-south-1 (Mumbai)
AMI             : Amazon Linux 2023
Instance Type   : t3.medium
Storage         : 20 GB
```

Launch the instance with a key pair.

---

# Step 2: Connect to EC2

## Purpose

Access the Linux terminal of the management server.

### SSH Connection

```bash
ssh -i "aws-mumbai.pem" ec2-user@<public-ip>
```

---

# Step 3: Verify AWS CLI

## Purpose

KOPS uses AWS CLI credentials to communicate with AWS services.

### Verify Installation

```bash
aws --version
```

### Expected Output

```text
aws-cli/2.x.x
```

---

# Step 4: Create IAM User

## Purpose

KOPS requires permissions to provision AWS infrastructure.

### Navigate to IAM

```text
AWS Console
→ IAM
→ Users
→ Create User
```

### User Details

```text
Username : kops
```

Enable:

```text
Provide user access to AWS Management Console
```

Password:

```text
Custom Password
```

Disable:

```text
Users must create a new password at next sign-in
```

Click:

```text
Next
```

### Assign Permissions

Attach:

```text
AdministratorAccess
```

Complete:

```text
Next
→ Create User
```

---

# Step 5: Create Access Key

## Purpose

AWS CLI requires programmatic credentials.

### Navigate

```text
IAM
→ Users
→ kops
→ Security Credentials
→ Create Access Key
```

Choose:

```text
Third Party Service
```

or

```text
Command Line Interface (CLI)
```

AWS generates:

```text
Access Key ID
Secret Access Key
```

Store both securely.

---

# Step 6: Configure AWS CLI

## Purpose

Authenticate AWS CLI with your AWS account.

### Configure Credentials

```bash
aws configure
```

Provide:

```text
AWS Access Key ID
AWS Secret Access Key
Default Region : ap-south-1
Default Output : json
```

### Verify Configuration

```bash
aws sts get-caller-identity
```

### Expected Output

```text
User ID
Account ID
ARN
```

---

# Step 7: Create S3 Bucket

## Purpose

KOPS stores cluster state and configuration files in S3.

### Create Bucket

```bash
aws s3 mb s3://abhis.kops.v1
```

### Verify

```bash
aws s3 ls
```

### Expected Output

```text
abhis.kops.v1
```

---

# Step 8: Enable Bucket Versioning

## Purpose

Protect cluster configuration against accidental modification or deletion.

### Navigate

```text
S3
→ abhis.kops.v1
→ Properties
→ Bucket Versioning
→ Enable
```

Save changes.

---

# Step 9: Configure KOPS State Store

## Purpose

Specify where KOPS stores cluster state information.

### Set Environment Variable

```bash
export KOPS_STATE_STORE=s3://abhis.kops.v1
```

### Verify

```bash
echo $KOPS_STATE_STORE
```

### Expected Output

```text
s3://abhis.kops.v1
```

> Important: Use lowercase `s3://`.

Correct:

```text
s3://bucket-name
```

Incorrect:

```text
S3://bucket-name
```

---

# Step 10: Generate SSH Key

## Purpose

KOPS uses SSH keys to access cluster nodes.

### Generate Key Pair

```bash
ssh-keygen
```

Press:

```text
Enter
Enter
Enter
```

### Verify

```bash
ls ~/.ssh
```

### Expected Output

```text
id_rsa
id_rsa.pub
```

---

# Step 11: Install KOPS

## Purpose

Install KOPS CLI.

### Installation

```bash
curl -Lo kops https://github.com/kubernetes/kops/releases/latest/download/kops-linux-amd64

chmod +x kops

sudo mv kops /usr/local/bin/
```

### Verify

```bash
kops version
```

### Expected Output

```text
Client version: x.x.x
```

---

# Step 12: Install kubectl

## Purpose

kubectl is used to interact with Kubernetes clusters.

### Installation

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

### Verify

```bash
kubectl version --client
```

### Expected Output

```text
Client Version: v1.x.x
```

---

# Step 13: Create Cluster Definition

## Purpose

Create the Kubernetes cluster configuration.

This step does not create AWS resources.

### Create Cluster

```bash
kops create cluster \
--name abhi.k8s.local \
--zones ap-south-1a \
--control-plane-size c7i-flex.large \
--node-size t3.small
```

### Result

```text
Cluster Configuration
Stored in S3 Bucket
```

---

# Step 14: Verify Cluster Definition

### Verify

```bash
kops get cluster
```

### Expected Output

```text
abhi.k8s.local
```

---

# Step 15: Create AWS Resources

## Purpose

Provision the actual Kubernetes infrastructure on AWS.

### Execute

```bash
kops update cluster \
--name abhi.k8s.local \
--yes --admin
```

### Resources Created

```text
VPC
Subnets
Route Tables
Security Groups
Load Balancer
Control Plane Node
Worker Node
Auto Scaling Groups
IAM Roles
```

---

# Step 16: Validate Cluster

## Purpose

Confirm cluster health and readiness.

### Validate

```bash
kops validate cluster --wait 10m
```

### Possible Output

```text
CoreDNS Pending
```

This is normal while the cluster is starting.

Wait a few minutes and retry.

---

# Step 17: Check Nodes

### Command

```bash
kubectl get nodes
```

### Expected Output

```text
STATUS : Ready
```

Example:

```text
Control Plane Ready
Node Ready
```

---

# Step 18: Check System Pods

### Command

```bash
kubectl get pods -A
```

### Verify Core Components

```text
CoreDNS
API Server
Scheduler
Controller Manager
```

All should be running.

---

# Step 19: View Cluster Information

### Command

```bash
kops get all
```

### Displays

```text
Cluster
Instance Groups
Control Plane
Nodes
```

---

# Step 20: Verify Resources in AWS Console

## EC2

```text
Control Plane Instance
Worker Node Instance
```

## Auto Scaling Groups

```text
Control Plane ASG
Worker Node ASG
```

## VPC

```text
KOPS VPC
```

## Load Balancer

```text
Kubernetes API Load Balancer
```

## S3

```text
Cluster State Files
```

---

# Cluster Creation Workflow

```text
Launch EC2
↓
Connect to EC2
↓
Verify AWS CLI
↓
Create IAM User
↓
Create Access Key
↓
Configure AWS CLI
↓
Create S3 Bucket
↓
Enable Versioning
↓
Configure KOPS_STATE_STORE
↓
Generate SSH Key
↓
Install KOPS
↓
Install kubectl
↓
Create Cluster Definition
↓
Update Cluster
↓
Validate Cluster
↓
Check Nodes
↓
Check Pods
↓
View Cluster Information
↓
Cluster Ready
```

---

# Common Mistakes and Fixes

## Mistake 1: Permission Denied While Using PEM File

### Error

```text
Load key Permission Denied
```

### Fix

Adjust PEM file permissions before connecting.

---

## Mistake 2: Forgot AWS Region

### Fix

Navigate to:

```text
AWS Global View
→ Region Explorer
```

Locate the region where resources were created.

---

## Mistake 3: Incorrect KOPS State Store Syntax

### Incorrect

```text
S3://bucket-name
```

### Correct

```text
s3://bucket-name
```

Use lowercase `s3`.

---

## Mistake 4: Deprecated Parameter

### Deprecated

```bash
--master-size
```

### Recommended

```bash
--control-plane-size
```

---

# Best Practices

- Enable versioning on the KOPS state store bucket.
- Use unique bucket names globally.
- Restrict IAM permissions in production environments.
- Store SSH private keys securely.
- Validate clusters after every deployment.
- Monitor cluster resources using CloudWatch.
- Regularly upgrade KOPS and Kubernetes versions.

---

# Summary

This lab demonstrates the complete lifecycle of creating a Kubernetes cluster on AWS using KOPS:

1. Create a management server.
2. Configure AWS access.
3. Create an S3 state store.
4. Install KOPS and kubectl.
5. Define the cluster.
6. Provision AWS infrastructure.
7. Validate cluster health.
8. Manage the cluster using kubectl and KOPS.

KOPS automates the creation and management of production-grade Kubernetes clusters by provisioning and maintaining the required AWS resources.
