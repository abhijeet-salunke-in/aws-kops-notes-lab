# Kubernetes ConfigMap

# Introduction

Applications require configuration values to run properly.

Examples:

- Environment Name
- Application Port
- API Endpoint
- Database Hostname
- Log Level
- Feature Flags

Example:

```text
ENV=dev
PORT=8080
LOG_LEVEL=INFO
```

These values may change frequently.

Kubernetes provides ConfigMap to manage such configurations separately from application code.

---

# What is a ConfigMap?

ConfigMap is a Kubernetes object used to store non-sensitive configuration data as key-value pairs.

It allows applications to consume configuration without rebuilding container images.

Definition:

> ConfigMap stores application configuration separately from container images and injects configuration into Pods as environment variables or files.

---

# Why ConfigMap Was Created

Consider an application running in three environments:

Development

```text
DB_HOST=dev-db
APP_COLOR=blue
DEBUG=true
```

Testing

```text
DB_HOST=test-db
APP_COLOR=yellow
DEBUG=true
```

Production

```text
DB_HOST=prod-db
APP_COLOR=green
DEBUG=false
```

Application code remains same.

Only configuration changes.

Without ConfigMap developers must:

1. Modify source code
2. Rebuild Docker image
3. Push image
4. Redeploy application

This violates the DevOps principle:

> Build Once, Deploy Anywhere

ConfigMap solves this problem.

---

# Problems Without ConfigMap

Suppose application contains:

```python
DB_HOST="dev-db"
```

When deploying to production:

- Code modification required
- New image build required
- New deployment required

This increases:

- Deployment time
- Human errors
- Maintenance effort

---

# Real Life Example

Think about a Television.

Television = Pod

Remote Control Settings = ConfigMap

Settings such as:

```text
Brightness
Volume
Language
Channel
```

can change without changing the TV itself.

Similarly:

Application remains same.

Configuration changes independently.

---

# Production Use Cases

ConfigMaps are commonly used to store:

```text
Application Name
Environment Name
URLs
API Endpoints
Ports
Feature Flags
Log Levels
Database Hostnames
```

Example:

```text
APP_NAME=payment-service
ENV=production
LOG_LEVEL=INFO
```

---

# ConfigMap vs Hardcoded Configuration

Hardcoded

```python
ENV="dev"
```

Problems:

- Requires rebuild
- Requires redeployment
- Difficult maintenance

ConfigMap

```yaml
data:
  ENV: dev
```

Benefits:

- No image rebuild
- Easy environment management
- Better DevOps practices

---

# ConfigMap vs Secret

ConfigMap stores:

```text
APP_NAME
PORT
ENVIRONMENT
URL
```

Secret stores:

```text
PASSWORD
TOKEN
API_KEY
CERTIFICATES
```

Never store passwords inside ConfigMap.

Use Kubernetes Secrets.

---

# Advantages of ConfigMap

1. Separates configuration from code

2. Supports Build Once Deploy Anywhere

3. Easy environment management

4. No image rebuild required

5. Supports GitOps workflows

6. Easy updates

7. Can be mounted as files

8. Can be injected as environment variables

---

# Limitations of ConfigMap

1. Not encrypted

2. Not suitable for passwords

3. Maximum size approximately 1 MB

4. Large ConfigMaps become difficult to manage

5. Changes may require pod restart depending on usage method

---

# Four Ways to Create ConfigMap

Kubernetes supports:

1. Literal Values
2. YAML File
3. Environment File
4. Folder

---

# Method 1 - Using Literal Values

## Why Use It

Useful for testing or very small configurations.

Example:

```bash
kubectl create configmap coursecm \
--from-literal=course=devops
```

Verify:

```bash
kubectl get cm
```

```bash
kubectl describe cm coursecm
```

Expected Output:

```text
course=devops
```

### Advantages

- Fast
- Simple

### Drawbacks

- Difficult to maintain
- Not Git-friendly
- Not used much in production

---

# Method 2 - Using YAML File

## Why Use It

Most common production method.

Create file:

configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: app-config

data:
  APP_NAME: nginx
  ENV: dev
  COLOR: blue
```

Apply:

```bash
kubectl apply -f configmap.yaml
```

Verify:

```bash
kubectl get cm
```

```bash
kubectl describe cm app-config
```

### Advantages

- Version controlled
- GitHub friendly
- CI/CD friendly
- GitOps compatible

### Drawbacks

- Requires YAML knowledge

---

# Method 3 - Using Environment File

## Why Use It

Many applications already use .env files.

Create file:

app.env

```text
APP_NAME=nginx
ENV=dev
COLOR=blue
```

Create ConfigMap:

```bash
kubectl create cm envcm \
--from-env-file=app.env
```

Verify:

```bash
kubectl describe cm envcm
```

### Advantages

- Easy migration from existing applications
- Developer friendly

### Drawbacks

- Less Kubernetes-native than YAML

---

# Method 4 - Using Folder

## Why Use It

Applications may contain multiple configuration files.

Example:

```text
configs/

├── nginx.conf
├── app.properties
└── logging.conf
```

Create ConfigMap:

```bash
kubectl create cm foldercm \
--from-file=configs/
```

Verify:

```bash
kubectl describe cm foldercm
```

### Advantages

- Manage multiple files together
- Useful for Nginx, Apache, Java applications

### Drawbacks

- Can become difficult to manage if folder contains many files

---

# Consuming ConfigMap in Pods

Creating ConfigMap alone is not enough.

Pods must consume ConfigMap.

Two major approaches exist:

1. Environment Variables
2. Volume Mount

---

# Method A - Environment Variables

Create ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: color-config

data:
  APP_COLOR: blue
```

Pod Manifest:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-pod

spec:
  containers:
  - name: nginx
    image: nginx

    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: color-config
          key: APP_COLOR
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Enter Pod:

```bash
kubectl exec -it nginx-pod -- sh
```

Check:

```bash
printenv
```

Output:

```text
APP_COLOR=blue
```

---

# Method B - Mount ConfigMap as Volume

ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: file-config

data:
  config.txt: |
    Hello from ConfigMap
```

Pod:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-pod

spec:
  containers:
  - name: nginx
    image: nginx

    volumeMounts:
    - name: config-volume
      mountPath: /config

  volumes:
  - name: config-volume
    configMap:
      name: file-config
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Enter Pod:

```bash
kubectl exec -it nginx-pod -- sh
```

Check:

```bash
cat /config/config.txt
```

Output:

```text
Hello from ConfigMap
```

---

# ConfigMap Update Behavior

Scenario:

Old Value

```text
APP_COLOR=blue
```

Updated Value

```text
APP_COLOR=green
```

Behavior depends on usage method.

### Environment Variable Method

Pod restart required.

Reason:

Environment variables are loaded only during container startup.

---

### Volume Mount Method

Usually updates automatically after some time.

Pod restart generally not required.

---

# Production Best Practices

1. Store only non-sensitive data

2. Use Secret for passwords

3. Use YAML manifests

4. Keep ConfigMaps in Git

5. Follow GitOps practices

6. Use meaningful names

Example:

```text
payment-config
frontend-config
logging-config
```

7. Avoid very large ConfigMaps

---

# Common Interview Questions

### What is ConfigMap?

A Kubernetes object used to store non-sensitive configuration data.

---

### Why use ConfigMap?

To separate configuration from application code.

---

### Can ConfigMap store passwords?

No.

Use Kubernetes Secrets.

---

### How can Pods consume ConfigMap?

1. Environment Variables

2. Volume Mounts

---

### What happens when ConfigMap changes?

Environment Variable Method:

- Pod restart required

Volume Mount Method:

- Automatically refreshed in most cases

---

### Maximum size of ConfigMap?

Approximately 1 MB.

---

### Which method is most common in production?

YAML-based ConfigMaps managed through GitOps workflows.

---

# Summary

ConfigMap is one of the most important Kubernetes objects.

It helps:

- Separate configuration from code
- Avoid image rebuilds
- Support multiple environments
- Simplify deployments
- Follow DevOps best practices

Creation Methods:

1. Literal
2. YAML File
3. Environment File
4. Folder

Consumption Methods:

1. Environment Variables
2. Volume Mounts

Sensitive information should always be stored in Kubernetes Secrets rather than ConfigMaps.
