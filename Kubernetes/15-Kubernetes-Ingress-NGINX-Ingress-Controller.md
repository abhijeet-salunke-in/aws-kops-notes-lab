# Kubernetes Ingress and NGINX Ingress Controller

## Introduction

In our previous Kubernetes sessions, we learned almost all the core Kubernetes resources such as Pods, ReplicaSets, Deployments, Services, ConfigMaps, Secrets, Persistent Volumes, StatefulSets, Jobs, CronJobs, and DaemonSets.

After completing these topics, our trainer introduced **NGINX** before starting **Kubernetes Ingress**.

Many beginners directly start learning Ingress without understanding NGINX. However, understanding NGINX first makes learning Ingress much easier because the NGINX Ingress Controller internally uses NGINX as a reverse proxy.

This chapter covers:

- NGINX recap
- Why Kubernetes needed Ingress
- Problems solved by Ingress
- Complete practical implementation
- Troubleshooting
- Interview questions

---

# NGINX Recap (Before Learning Ingress)

Before starting Kubernetes Ingress, our trainer first taught NGINX.

This was important because the NGINX Ingress Controller internally behaves like an NGINX Reverse Proxy.

Therefore, before learning Ingress, we learned how NGINX works.

---

# What is NGINX?

NGINX is a lightweight, high-performance web server.

It can also work as:

- Web Server
- Reverse Proxy
- Forward Proxy
- Load Balancer
- HTTP Cache Server

Official Website:

https://nginx.org

---

# What is a Web Server?

A Web Server is software that listens for HTTP requests from browsers and returns web pages.

Example:

```
Browser

     |

HTTP Request

     |

NGINX

     |

index.html

     |

HTML Response

     |

Browser Displays Website
```

Popular Web Servers

- NGINX
- Apache
- IIS

---

# HTTP Request Flow

When a user opens a website,

Example

```
http://example.com
```

The browser sends an HTTP request.

NGINX receives the request.

NGINX looks for the requested file.

Example

```
index.html
```

If the file exists,

NGINX sends it back to the browser.

```
Browser

      |

HTTP Request

      |

NGINX

      |

Find index.html

      |

Return HTML

      |

Browser
```

---

# Common Network Ports

| Port | Purpose |
|-------|----------|
|22|SSH|
|80|HTTP|
|443|HTTPS|
|3306|MySQL|
|27017|MongoDB|

NGINX usually listens on:

- Port 80 (HTTP)
- Port 443 (HTTPS)

---

# Installing NGINX on CentOS

Install NGINX

```bash
dnf install nginx -y
```

Check version

```bash
nginx -v
```

Start service

```bash
systemctl start nginx
```

Check status

```bash
systemctl status nginx
```

Enable at boot

```bash
systemctl enable nginx
```

---

# Hosting Our First Website

Create website directory

```bash
mkdir -p /var/www/website1
```

Create HTML file

```bash
cd /var/www/website1

vim index.html
```

Initially, the website did not open.

Why?

Because NGINX did not know where our website files were located.

---

# Server Block

We created a Server Block.

Location

```
/etc/nginx/conf.d/
```

Example

```
website1.conf
```

The Server Block tells NGINX

- Which Port
- Which Domain
- Which Website Folder
- Which Index File

Without this configuration,

NGINX cannot locate our website.

---

# Test Configuration

Before restarting NGINX,

always test the configuration.

```bash
nginx -t
```

If everything is correct,

```
syntax is ok

test is successful
```

Then reload

```bash
systemctl reload nginx
```

---

# File Permissions

NGINX must have permission to read website files.

Ownership

```bash
chown -R nginx:nginx /var/www/website1
```

Permissions

```bash
chmod -R 755 /var/www/website1
```

Verify

```bash
ls -l /var/www/website1
```

---

# SELinux

During our practice,

SELinux prevented NGINX from accessing files.

Temporarily disable

```bash
setenforce 0
```

Reload

```bash
systemctl reload nginx
```

> **Note:** Never disable SELinux permanently in production. Configure proper SELinux policies instead.

---

# Important Features of NGINX

Our trainer discussed four important features.

- Forward Proxy
- Reverse Proxy
- Load Balancing
- HTTP Caching

---

# Forward Proxy

A Forward Proxy sits between the client and the internet.

```
Client

     |

Forward Proxy

     |

Internet
```

Purpose

- Hide Client Identity
- Restrict Internet Access
- Used in Schools
- Used in Companies

---

# Reverse Proxy

A Reverse Proxy sits in front of backend servers.

```
Client

      |

NGINX

      |

Backend Servers

 |      |      |

Node   Java   Python
```

Advantages

- Hides Backend Servers
- SSL Termination
- Load Balancing
- Security
- Routing

This Reverse Proxy concept is the foundation of Kubernetes Ingress.

---

# Load Balancing

NGINX can distribute incoming requests across multiple servers.

```
Users

      |

NGINX

 /     |      \

App1  App2   App3
```

Common Algorithms

- Round Robin
- Least Connections
- IP Hash

This is similar to how Kubernetes Services distribute traffic across Pods.

---

# HTTP Caching

NGINX can cache static files.

Examples

```
logo.png

style.css

script.js
```

Advantages

- Faster Website
- Lower Backend Load
- Better User Experience

---

# Why Did We Learn NGINX Before Ingress?

Because Kubernetes Ingress does not directly expose applications.

Instead,

an **Ingress Controller** (such as NGINX Ingress Controller) works as a Reverse Proxy.

The concepts we learned in NGINX are used internally by Kubernetes.

```
Browser

      |

AWS LoadBalancer

      |

NGINX Ingress Controller

      |

Service

      |

Pods
```

Now that we understand Reverse Proxy,

we can easily understand Kubernetes Ingress.

---

# What is Kubernetes Ingress?

Ingress is a Kubernetes API resource used to expose multiple HTTP/HTTPS applications using a single external entry point.

Instead of creating one LoadBalancer Service for every application,

Ingress allows us to expose multiple Services through a single LoadBalancer.

Ingress mainly provides

- Path-Based Routing
- Host-Based Routing
- SSL/TLS Termination
- Centralized Routing

---

# Why Do We Need Ingress?

Let's understand with a simple example.

Suppose we have only one application.

```
Internet

      |

LoadBalancer

      |

Service

      |

Pods
```

This works perfectly.

Now imagine the company grows.

Applications

- Website
- Mobile API
- HR Portal
- Payment Service
- Admin Portal
- Analytics

If every application uses

```
Service Type = LoadBalancer
```

AWS creates

```
One ELB

for

One Service
```

Therefore

```
6 Applications

↓

6 LoadBalancers

↓

6 Public IPs

↓

Higher AWS Cost

↓

Difficult Management
```

Imagine 100 applications.

That would require

- 100 LoadBalancers
- 100 Public Endpoints
- 100 Security Configurations

This is expensive and difficult to manage.

---

# Solution: Kubernetes Ingress

Instead of creating multiple LoadBalancers,

we create only one external LoadBalancer.

```
Internet

      |

AWS LoadBalancer

      |

NGINX Ingress Controller

      |

Ingress Rules

      |

Services

      |

Pods
```

Now,

all applications can be accessed through one public endpoint.

Example

```
company.com/

company.com/app

company.com/medical

company.com/portfolio
```

instead of

```
52.xx.xx.xx

54.xx.xx.xx

18.xx.xx.xx

13.xx.xx.xx
```

Ingress provides a single entry point for multiple applications.

---

# Problems Solved by Ingress

Ingress solves several real-world production problems.

### 1. Reduces AWS Cost

Only one external LoadBalancer is required.

---

### 2. Single Public Entry Point

Users access applications through one domain.

---

### 3. Centralized Routing

Routing rules are managed in one place.

---

### 4. Easier SSL Management

TLS certificates can be managed centrally.

---

### 5. Better Security

Only one public endpoint is exposed.

Internal Services remain private.

---

# Real World Analogy

Think of a hospital.

Without a receptionist,

patients would directly enter different departments.

This would create confusion.

Instead,

everyone first goes to the Reception Desk.

The receptionist checks the requirement and sends the patient to the correct department.

```
Patient

      |

Reception

 /      |       \

Cardiology

Orthopedics

Emergency
```

Ingress works exactly like a Receptionist.

Incoming requests first reach the Ingress Controller.

The controller checks the routing rules and forwards the request to the correct Kubernetes Service.

---

# Components of Kubernetes Ingress

There are three important components.

1. Ingress Resource

2. Ingress Controller

3. IngressClass

## 1. Ingress Resource

Ingress Resource is a Kubernetes object that contains routing rules.

It tells Kubernetes:

- Which URL should be accessed?
- Which Service should receive the request?
- Which Port should be used?

Example

```
/app

↓

app-service
```

```
/medical

↓

medical-service
```

An Ingress Resource **does not route traffic by itself.**

It only stores the routing rules inside the Kubernetes API Server.

---

## 2. Ingress Controller

The Ingress Controller is the software responsible for implementing the routing rules.

Popular Ingress Controllers

- NGINX Ingress Controller
- Traefik
- HAProxy
- AWS ALB Controller

In our lab,

we use

```
NGINX Ingress Controller
```

Think of it like this

```
Ingress Resource

↓

Rules

↓

NGINX Ingress Controller

↓

Traffic Routing
```

Without an Ingress Controller,

Ingress will never work.

---

## 3. IngressClass

A Kubernetes cluster may have multiple Ingress Controllers.

Example

```
NGINX

Traefik

AWS ALB
```

IngressClass tells Kubernetes

which controller should manage the Ingress.

Example

```yaml
ingressClassName: nginx
```

This means

Only the NGINX Ingress Controller should process this Ingress.

---

# Complete Request Flow

```
Browser

     |

HTTP Request

     |

AWS ELB

     |

NGINX Ingress Controller

     |

Ingress Rules

     |

ClusterIP Service

     |

Pods

     |

Response

     |

Browser
```

Notice

Ingress **never communicates directly with Pods.**

It always communicates with Services.

Services then forward requests to Pods.

---

# Practical Lab

Now we will perform the same practical demonstrated by our trainer.

---

# Step 1 - Create App Pod

File

```
app-pod.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  labels:
    app: app

spec:
  containers:
  - name: app-container
    image: aarushtechnologies/app
    ports:
    - containerPort: 80
```

Create Pod

```bash
kubectl apply -f app-pod.yml
```

Verify

```bash
kubectl get pods
```

Why?

This Pod hosts our first web application.

---

# Step 2 - Create Medical Pod

File

```
medical-pod.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: medical-pod
  labels:
    app: medical

spec:
  containers:
  - name: medical-container
    image: aarushtechnologies/medical
    ports:
    - containerPort: 80
```

Create

```bash
kubectl apply -f medical-pod.yml
```

Verify

```bash
kubectl get pods
```

Why?

This Pod hosts our second application.

---

# Step 3 - Create ClusterIP Services

Now the Pods need stable networking.

Remember

Pods are temporary.

Their IP addresses can change.

Therefore,

Ingress should never communicate directly with Pods.

Instead,

Ingress communicates with ClusterIP Services.

---

## Create App Service

File

```
app-service.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service

spec:
  selector:
    app: app

  ports:
  - port: 80
    targetPort: 80

  type: ClusterIP
```

Create

```bash
kubectl apply -f app-service.yml
```

---

## Create Medical Service

File

```
medical-service.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: medical-service

spec:
  selector:
    app: medical

  ports:
  - port: 80
    targetPort: 80

  type: ClusterIP
```

Create

```bash
kubectl apply -f medical-service.yml
```

Verify

```bash
kubectl get svc
```

Why ClusterIP?

Because Ingress is inside the cluster.

It can communicate with internal Services.

No need to expose every Service using LoadBalancer.

---

Current Architecture

```
               Kubernetes Cluster

     app-service

           |

       app-pod


 medical-service

           |

     medical-pod
```

Nothing is exposed to the Internet yet.

---

# Step 4 - Install NGINX Ingress Controller

Before creating the Ingress Resource,

we must install an Ingress Controller.

Without it,

Ingress cannot route traffic.

Install

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

This creates

- Namespace
- ServiceAccount
- Roles
- ClusterRoles
- Deployments
- Services
- Admission Webhooks
- Ingress Controller Pod

---

# Step 5 - Verify Installation

Check Pods

```bash
kubectl get pods -n ingress-nginx
```

Expected

```
ingress-nginx-controller

Running
```

Check Service

```bash
kubectl get svc -n ingress-nginx
```

Expected

```
TYPE

LoadBalancer
```

The external AWS ELB will be attached here.

---

# Step 6 - Create Ingress Resource

Create file

```
ingress.yml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: my-ingress

  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /

spec:

  ingressClassName: nginx

  rules:

  - http:

      paths:

      - path: /
        pathType: Prefix

        backend:
          service:
            name: app-service

            port:
              number: 80

      - path: /medical
        pathType: Prefix

        backend:
          service:
            name: medical-service

            port:
              number: 80
```

---

# Understanding the Ingress YAML

### apiVersion

Specifies the API group.

```
networking.k8s.io/v1
```

---

### kind

Creates an Ingress object.

```
Ingress
```

---

### metadata

Stores object information.

Example

```
name

annotations
```

---

### annotation

```
rewrite-target
```

This tells NGINX

how to rewrite incoming URLs before forwarding them to the backend application.

---

### spec

Contains the desired routing configuration.

---

### ingressClassName

```
nginx
```

This tells Kubernetes

which controller should manage this Ingress.

---

### rules

Contains all routing rules.

---

### path

```
/

/medical
```

Specifies which URL should match.

---

### backend

Specifies

which Service should receive the request.

---

### service

Backend Kubernetes Service.

---

### port

Backend Service Port.

---

# Step 7 - Apply the Ingress

Create the resource

```bash
kubectl apply -f ingress.yml
```

Verify

```bash
kubectl get ingress
```

Example

```
NAME

my-ingress

CLASS

nginx

ADDRESS

AWS ELB DNS
```

Once the ADDRESS appears,

our Ingress is ready.

# Step 8 - Test the Ingress

After creating the Ingress,

open your browser.

Example

```
http://<ELB-DNS>/
```

Expected Output

```
App Website
```

Now test

```
http://<ELB-DNS>/medical
```

Expected Output

```
Medical Website
```

Both applications are accessible using a single LoadBalancer.

This is the biggest advantage of Ingress.

---

# Complete Request Flow

When a user opens

```
http://<ELB>/medical
```

the request follows this path

```
Browser

     |

Internet

     |

AWS ELB

     |

NGINX Ingress Controller

     |

Ingress Rule

     |

medical-service

     |

medical-pod

     |

Response
```

Similarly,

```
http://<ELB>/
```

is routed to

```
app-service

↓

app-pod
```

---

# Step 9 - Add Another Application

Our trainer later added another application.

Image

```
abhisalunke16/portfolio:v3
```

---

## Create Portfolio Pod

File

```
port-pod.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: port-pod
  labels:
    app: port

spec:
  containers:
  - name: port-container
    image: abhisalunke16/portfolio:v3

    ports:
    - containerPort: 80
```

Create

```bash
kubectl apply -f port-pod.yml
```

---

## Create Portfolio Service

File

```
port-service.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: port-service

spec:
  selector:
    app: port

  ports:
  - port: 80
    targetPort: 80

  type: ClusterIP
```

Create

```bash
kubectl apply -f port-service.yml
```

---

# Update ingress.yml

Add another path

```yaml
- path: /portfolio
  pathType: Prefix

  backend:
    service:
      name: port-service

      port:
        number: 80
```

Apply

```bash
kubectl apply -f ingress.yml
```

Test

```
http://<ELB>/portfolio
```

Now three applications are available through a single LoadBalancer.

```
Internet

      |

AWS ELB

      |

NGINX Ingress Controller

      |

----------------------------

|        |          |

/      /medical   /portfolio

|        |          |

App   Medical   Portfolio
```

---

# rewrite-target Annotation

Our Ingress contains

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
```

Purpose

Rewrite the incoming URL before sending it to the backend Service.

Example

Browser Request

```
/medical
```

Ingress forwards

```
/
```

to the backend application.

This is useful when backend applications expect requests from the root path.

---

# Troubleshooting During Lab

While testing,

we encountered an interesting issue.

---

## Problem

Initially,

our Ingress looked like this

```
/

↓

medical-service

/app

↓

app-service
```

When we opened

```
http://ELB/app
```

The webpage loaded,

but

- CSS was missing
- JavaScript was not working
- Website design was broken

---

# Why Did This Happen?

The HTML page loaded successfully.

However,

the browser then requested

```
/css/style.css

/js/script.js
```

instead of

```
/app/css/style.css

/app/js/script.js
```

Since

```
/
```

was pointing to

```
medical-service
```

those CSS and JS requests were sent to the wrong application.

As a result,

the browser received incorrect responses and the webpage appeared broken.

---

# How We Fixed It

We changed our Ingress configuration.

Old

```
/

↓

medical-service
```

New

```
/

↓

app-service

/medical

↓

medical-service
```

Now,

all root requests including

```
/

/css

/js

/images
```

reached

```
app-service
```

The browser successfully loaded

- CSS
- JavaScript
- Images

The application started working correctly.

---

# Lesson Learned

Not every web application supports running inside

```
/app
```

Many frontend applications expect to run directly from

```
/
```

In production,

this is usually solved by

- configuring the application base path
- using subdomains
- or using proper rewrite rules

---

# Common Errors

## 1. Ingress Controller Not Running

Symptoms

```
Ingress Created

↓

No Response
```

Solution

```bash
kubectl get pods -n ingress-nginx
```

---

## 2. Wrong Service Name

If the Service name in

```
backend
```

does not exist,

Ingress cannot forward traffic.

Verify

```bash
kubectl get svc
```

---

## 3. Wrong Labels

If Service selectors do not match Pod labels,

Endpoints will not be created.

Check

```bash
kubectl get endpoints
```

---

## 4. IngressClass Missing

Example

```yaml
ingressClassName: nginx
```

Without the correct class,

the controller may ignore the Ingress.

---

## 5. Pending External Address

Sometimes

```
kubectl get ingress
```

shows

```
ADDRESS

<pending>
```

Wait for AWS to provision the ELB.

---

## 6. CSS and JavaScript Not Loading

Possible reasons

- Wrong path
- Wrong rewrite rule
- Application expects root path
- Incorrect frontend base URL

---

# Useful Commands

Check Pods

```bash
kubectl get pods
```

Check Services

```bash
kubectl get svc
```

Check Ingress

```bash
kubectl get ingress
```

Describe Ingress

```bash
kubectl describe ingress my-ingress
```

Check Ingress Controller

```bash
kubectl get pods -n ingress-nginx
```

Check Ingress Controller Service

```bash
kubectl get svc -n ingress-nginx
```

Check Endpoints

```bash
kubectl get endpoints
```

View Logs

```bash
kubectl logs -n ingress-nginx <controller-pod-name>
```

---

# Best Practices

- Use ClusterIP Services behind Ingress.
- Expose only one external LoadBalancer whenever possible.
- Always test Ingress after applying changes.
- Keep Ingress rules simple and organized.
- Use meaningful Service names.
- Monitor the Ingress Controller logs.
- Manage TLS certificates properly in production.
- Do not expose internal Services directly unless required.

# Important Interview Questions

### 1. What is Kubernetes Ingress?

Ingress is a Kubernetes resource that provides external access to multiple Services through a single entry point.  
It supports features like path-based routing, host-based routing, and SSL/TLS termination.

---

### 2. What is an Ingress Controller?

An Ingress Controller is the component that reads Ingress rules and routes incoming traffic to the correct Kubernetes Service.  
Without an Ingress Controller, an Ingress resource has no effect.

---

### 3. Why do we use Ingress instead of multiple LoadBalancer Services?

Using multiple LoadBalancer Services creates multiple public IPs and increases cloud costs.  
Ingress allows multiple applications to share a single LoadBalancer, making routing simpler and more cost-effective.

---

### 4. What is the difference between a Service and an Ingress?

A Service exposes Pods inside or outside the cluster, while an Ingress manages HTTP/HTTPS routing to multiple Services.  
Ingress always routes traffic to Services, not directly to Pods.

---

### 5. What is the purpose of the `rewrite-target` annotation?

The `rewrite-target` annotation modifies the request URL before forwarding it to the backend Service.  
It is commonly used when backend applications expect requests from the root (`/`) path.

# Summary

In this chapter, we learned how Kubernetes Ingress provides a single entry point for multiple applications.

We first understood NGINX because the NGINX Ingress Controller internally uses NGINX as a reverse proxy.

We created:

- App Pod
- Medical Pod
- Portfolio Pod
- ClusterIP Services
- NGINX Ingress Controller
- Ingress Resource

We learned how requests travel from the browser to the application through:

```
Browser

↓

AWS LoadBalancer

↓

NGINX Ingress Controller

↓

Ingress Rules

↓

ClusterIP Service

↓

Pod
```

Finally, we also diagnosed a real-world issue where CSS and JavaScript files failed to load because the application expected to run from the root (`/`) instead of a URL prefix (`/app`). This troubleshooting exercise reinforced how Ingress routing, path rewriting, and frontend asset paths work together.

Ingress is a key Kubernetes networking resource that enables path-based and host-based routing, simplifies external access, reduces cloud costs by minimizing the number of external load balancers, and provides a centralized entry point for applications running inside a Kubernetes cluster.

