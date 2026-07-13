# Kubernetes StatefulSet, Persistent Volume, MongoDB ReplicaSet and Complete Application Deployment

---

# Table of Contents

1. Introduction
2. What is Stateful and Stateless Application?
3. Stateless Applications
4. Stateful Applications
5. Why Kubernetes Introduced StatefulSet
6. What is a StatefulSet?
7. Features of StatefulSet
8. Stateful vs Stateless Comparison
9. Deployment vs StatefulSet
10. Pod Identity
11. Stable Network Identity
12. Stable Storage
13. Ordered Deployment
14. Ordered Scaling
15. Ordered Termination
16. When Should We Use StatefulSet?
17. When Should We NOT Use StatefulSet?
18. Real World Examples
19. StatefulSet Architecture
20. Summary

---

# Introduction

Before learning StatefulSet, we first need to understand why Kubernetes introduced it.

Kubernetes was originally designed to run containerized applications. Most early applications deployed on Kubernetes were **stateless** applications such as web servers, REST APIs, frontend applications, reverse proxies, and load balancers.

These applications do not permanently store user data inside the container. Therefore, Kubernetes could freely create, delete, or replace Pods whenever needed.

For example,

```
Nginx

Apache

React

Angular

NodeJS API

Spring Boot API
```

All of these applications can be restarted without losing important information because the actual data is usually stored somewhere else.

As Kubernetes adoption increased, organizations also wanted to run databases inside Kubernetes.

Examples include:

- MongoDB
- MySQL
- PostgreSQL
- Cassandra
- Elasticsearch
- Kafka
- ZooKeeper
- Redis with Persistence

Unlike web servers, databases cannot simply lose their identity or storage.

If a database Pod disappears and a completely new Pod is created with a different name and empty storage, all application data may be lost.

This became a major problem.

To solve this problem, Kubernetes introduced **StatefulSet**.

StatefulSet provides:

- Stable Pod Names
- Stable Network Identity
- Persistent Storage
- Ordered Deployment
- Ordered Scaling
- Ordered Deletion

These features make StatefulSet the ideal controller for databases and other stateful applications.

---

# What is a Stateful Application?

A Stateful Application is an application that stores information about its previous state.

Simply put,

> The application remembers previous data even after restarting.

For example,

Suppose you have a MongoDB database.

Inside MongoDB, you have:

```
Database:
Company

Collection:
Employees

Documents:
Abhijeet
Vishal
Aarush
```

Now imagine the MongoDB Pod crashes.

If Kubernetes recreates the Pod,

the database **must still contain**

```
Company Database

Employees Collection

All Documents
```

If everything disappears after restart,

the application becomes useless.

Therefore databases require persistent identity and storage.

These applications are called Stateful Applications.

---

# What is State?

State means information that must survive application restarts.

Examples:

- Database Records
- User Accounts
- Passwords
- Uploaded Files
- Transaction History
- Orders
- Logs
- Messages
- Replicated Data
- Cluster Membership
- Configuration

If losing this information breaks the application,

the application is Stateful.

---

# Stateless Applications

Stateless applications do not store user data inside the Pod.

Each request is independent.

The server never needs to remember previous requests.

Example:

Suppose we deploy a React application.

```
Browser

↓

React Pod

↓

Response
```

Tomorrow the Pod crashes.

Kubernetes creates

```
React Pod (New)
```

Nothing is lost because React files are already present inside the Docker Image.

Another example is Nginx.

```
User

↓

Nginx

↓

Returns HTML
```

Nginx doesn't need to remember anything.

If the Pod is deleted,

another Pod immediately starts serving traffic.

No user data is lost.

Examples of Stateless Applications:

- React
- Angular
- Vue
- Nginx
- Apache
- HAProxy
- Traefik
- NodeJS API
- Spring Boot API
- Python Flask API
- Go REST API

These applications usually run inside

```
Deployment
```

instead of StatefulSet.

---

# Characteristics of Stateless Applications

A stateless application generally has the following properties:

- No permanent data inside Pod
- Easy Horizontal Scaling
- Any Pod can handle any request
- Pod names are not important
- Storage is optional
- Pods are interchangeable
- Easy rolling updates
- Easy rollback
- Faster deployment
- Easier maintenance

---

# Example of Stateless Application

Suppose Kubernetes creates

```
frontend-8hdf6

frontend-a82jj

frontend-p28dk
```

Tomorrow Pod 2 crashes.

Kubernetes creates

```
frontend-x91ka
```

Nothing breaks.

Users never notice.

This is because Pod identity is not important.

---

# Stateful Applications

A Stateful Application stores information that must survive restarts.

Each Pod has its own identity.

Each Pod owns its own storage.

Each Pod has its own network identity.

Deleting the Pod should NOT delete the data.

Examples include

- MongoDB
- MySQL
- PostgreSQL
- Oracle Database
- Cassandra
- Kafka
- Elasticsearch
- ZooKeeper
- Redis (Persistent Mode)

---

# Example of Stateful Application

Suppose MongoDB has

```
Employee Database
```

Inside

```
Employees Collection
```

Containing

```
Abhijeet

Vishal

Rahul

Aarush
```

Now MongoDB crashes.

If Kubernetes creates

```
mongodb-new-8dhsj
```

with empty storage,

the entire database disappears.

This is unacceptable.

Therefore MongoDB requires:

- Stable Name
- Stable Storage
- Stable Network Identity

---

# Why Kubernetes Needed StatefulSet

Deployment works perfectly for stateless applications.

However,

Deployment has several limitations.

Whenever a Pod dies,

Deployment creates another Pod with

- Different Name
- Different Identity
- Different Storage

For example,

```
mongodb-x82ja
```

crashes.

Deployment creates

```
mongodb-bd72k
```

This Pod is completely different.

Its storage is different.

Its hostname is different.

Its DNS is different.

Its identity is different.

Databases cannot work like this.

StatefulSet solves this problem.

---

# What is StatefulSet?

StatefulSet is a Kubernetes workload controller that manages stateful applications.

It guarantees:

- Stable Pod Identity
- Stable Network Identity
- Stable Persistent Storage
- Ordered Deployment
- Ordered Scaling
- Ordered Updates
- Ordered Deletion

Unlike Deployment,

Pods are NOT identical.

Each Pod becomes a unique member of the application cluster.

For example,

```
mongodb-0

mongodb-1

mongodb-2
```

Each Pod permanently owns its identity.

---

# Features of StatefulSet

StatefulSet provides several powerful features.

## 1. Stable Pod Names

Pods always keep their names.

Example

```
mongodb-0

mongodb-1

mongodb-2
```

Even after restart,

the Pod names remain unchanged.

---

## 2. Stable Hostname

Every Pod gets a permanent hostname.

Example

```
mongodb-0.mongodb-svc.default.svc.cluster.local

mongodb-1.mongodb-svc.default.svc.cluster.local
```

Applications always know where to connect.

---

## 3. Stable Storage

Every Pod owns one Persistent Volume Claim.

Example

```
mongodb-0

↓

PVC-0

↓

PV-0

↓

AWS EBS Volume
```

Even if Pod is deleted,

the volume remains attached.

---

## 4. Ordered Deployment

Pods are created sequentially.

```
mongodb-0

↓

Ready

↓

mongodb-1

↓

Ready

↓

mongodb-2
```

Next Pod starts only after previous Pod becomes Ready.

---

## 5. Ordered Scaling

Scaling from

```
2

↓

5
```

creates

```
mongodb-2

↓

mongodb-3

↓

mongodb-4
```

in order.

---

## 6. Ordered Termination

Scaling down

```
5

↓

2
```

deletes

```
mongodb-4

↓

mongodb-3

↓

mongodb-2
```

The primary Pods remain safe.

---

## 7. Persistent Identity

Even after restart,

```
mongodb-0
```

always remains

```
mongodb-0
```

It never becomes

```
mongodb-x837d
```

like Deployment Pods.

---

# Why Stable Identity Matters

Many distributed databases identify nodes using hostnames.

For example,

MongoDB ReplicaSet may contain

```
mongodb-0.mongodb-svc

mongodb-1.mongodb-svc

mongodb-2.mongodb-svc
```

If hostnames keep changing,

replication breaks.

Therefore StatefulSet guarantees permanent hostnames.

---

# Real World Applications Using StatefulSet

Some of the most popular Stateful Applications are:

Database Systems

- MongoDB
- MySQL
- PostgreSQL
- MariaDB
- Oracle Database

Streaming Platforms

- Kafka
- Pulsar

Distributed Storage

- Cassandra
- HBase

Search Engines

- Elasticsearch
- OpenSearch

Coordination Services

- ZooKeeper
- etcd

Caching Systems

- Redis (Persistent Mode)

Monitoring

- VictoriaMetrics
- Prometheus (with persistent storage)

---

# Summary

In this section, we learned why StatefulSet exists and why it is one of the most important Kubernetes workload controllers.

We understood that Deployments are excellent for stateless applications but are not suitable for applications that require stable identity, stable networking, and persistent storage.

StatefulSet solves these challenges by providing predictable Pod names, stable DNS records, persistent volumes, and ordered lifecycle management.

In the next section, we will dive deeper into one of the most important concepts required by StatefulSets:

- Headless Services
- Stable DNS
- Persistent Volumes (PV)
- Persistent Volume Claims (PVC)
- StorageClasses
- Dynamic Volume Provisioning
- How StatefulSet automatically creates storage for each Pod

# Part 2 - Headless Service, Stable DNS, Persistent Volumes (PV), Persistent Volume Claims (PVC) and StorageClass

---

# Why Do We Need a Headless Service?

In Kubernetes, Pods are **temporary resources**.

Whenever a Pod crashes, Kubernetes may recreate it.

Although StatefulSet gives Pods stable names like

```
mongodb-0

mongodb-1

mongodb-2
```

applications still need a way to discover these Pods over the network.

Imagine a MongoDB Replica Set.

```
MongoDB Primary

↓

MongoDB Secondary

↓

MongoDB Secondary
```

Each MongoDB instance must know how to communicate with the other members.

MongoDB cannot communicate using Pod IP addresses because Pod IPs can change after recreation.

Instead, MongoDB communicates using DNS names.

For example,

```
mongodb-0.mongodb-svc

mongodb-1.mongodb-svc

mongodb-2.mongodb-svc
```

These DNS names never change.

That is why StatefulSets always require a Headless Service.

---

# What is a Headless Service?

A Headless Service is a Service that **does not allocate a ClusterIP**.

Instead of load balancing traffic,

it returns the DNS records of every Pod individually.

Normal Service

```
Client

↓

ClusterIP

↓

Load Balancer

↓

Pod-1

Pod-2

Pod-3
```

Headless Service

```
Client

↓

DNS Query

↓

Pod-1

Pod-2

Pod-3
```

Notice that there is **no ClusterIP**.

---

# How to Create a Headless Service?

The only difference between a normal Service and a Headless Service is

```yaml
clusterIP: None
```

Example

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mongodb-svc

spec:
  clusterIP: None

  selector:
    app: mongodb

  ports:
  - port: 27017
```

This single line

```yaml
clusterIP: None
```

converts the Service into a Headless Service.

---

# Difference Between ClusterIP and Headless Service

## ClusterIP Service

```
Service

↓

One Virtual IP

↓

Load Balancing

↓

Multiple Pods
```

All requests first reach the ClusterIP.

The Service decides which Pod should receive the request.

---

## Headless Service

```
Service

↓

DNS

↓

Individual Pods
```

No load balancing occurs.

The application receives the address of every Pod.

---

# Why MongoDB Needs Headless Service

MongoDB Replica Set requires every member to know the hostname of every other member.

Example

```
mongodb-0.mongodb-svc.default.svc.cluster.local

mongodb-1.mongodb-svc.default.svc.cluster.local
```

When MongoDB initializes the Replica Set,

it stores these hostnames.

Later,

all communication happens using these DNS names.

If the hostname changes,

replication breaks.

Therefore,

StatefulSet + Headless Service always work together.

---

# Stable DNS

One of the biggest advantages of StatefulSet is stable DNS.

Suppose Kubernetes creates

```
mongodb-0

mongodb-1
```

Each Pod automatically receives

```
mongodb-0.mongodb-svc.default.svc.cluster.local

mongodb-1.mongodb-svc.default.svc.cluster.local
```

These names never change.

Even after Pod recreation,

the DNS remains the same.

---

# DNS Hierarchy

Consider the following hostname

```
mongodb-0.mongodb-svc.default.svc.cluster.local
```

Let's break it down.

```
mongodb-0
```

Pod Name

---

```
mongodb-svc
```

Headless Service Name

---

```
default
```

Namespace

---

```
svc
```

Service

---

```
cluster.local
```

Cluster Domain

Combined

```
mongodb-0.mongodb-svc.default.svc.cluster.local
```

---

# How DNS Resolution Works

Suppose the NodeJS application tries to connect to

```
mongodb-0.mongodb-svc
```

The DNS server performs

```
mongodb-0.mongodb-svc.default.svc.cluster.local
```

↓

Find EndpointSlice

↓

Find Pod IP

↓

Return Pod Address

Example

100.96.1.113

The application connects successfully.

---

# Our Real Project

During our MongoDB project,

we configured

```
serviceName: mongodb-svc
```

inside the StatefulSet.

Our Pods became

```
mongodb-0

mongodb-1
```

The DNS automatically became

```
mongodb-0.mongodb-svc

mongodb-1.mongodb-svc
```

Later,

our NodeJS backend connected using

```javascript
mongodb://mongodb-0.mongodb-svc:27017,mongodb-1.mongodb-svc:27017/?replicaSet=rs0
```

No Pod IP was used.

Everything worked using DNS.

---

# EndpointSlice

Many beginners think the Service stores Pod IP addresses.

Actually,

Kubernetes creates an EndpointSlice.

Example

```
kubectl get endpointslices
```

Output

```
mongodb-svc-5jpnt
```

Inside the EndpointSlice

```
Hostname

mongodb-0

↓

IP

100.96.1.113

--------------------

Hostname

mongodb-1

↓

IP

100.96.1.195
```

The DNS server reads this information.

---

# Verifying DNS

Useful commands

```
kubectl get svc

kubectl get endpoints

kubectl get endpointslices

kubectl describe svc mongodb-svc

kubectl describe endpointslice mongodb-svc-xxxxx
```

---

# DNS Testing

Inside BusyBox

```
kubectl run dns --rm -it --image=busybox -- sh
```

Test DNS

```
nslookup mongodb-svc
```

Output

```
100.96.1.113

100.96.1.195
```

Test FQDN

```
nslookup mongodb-0.mongodb-svc.default.svc.cluster.local
```

Output

```
100.96.1.113
```

---

# Our DNS Issue

During our project,

we faced an interesting DNS problem.

Initially,

we tried

```
mongodb-0.mongodb
```

NodeJS produced

```
ENOTFOUND mongodb-0.mongodb
```

Reason

There was no Service called

```
mongodb
```

Our actual Headless Service name was

```
mongodb-svc
```

After correcting the connection string

```javascript
mongodb://mongodb-0.mongodb-svc:27017,mongodb-1.mongodb-svc:27017/?replicaSet=rs0
```

DNS started working correctly.

This was one of the major issues we solved.

---

# Persistent Storage

Another important requirement for Stateful Applications is storage.

Suppose MongoDB stores

```
Employee Data
```

Now,

the Pod crashes.

If the storage disappears,

the database becomes empty.

This is unacceptable.

Therefore,

Stateful Applications always require Persistent Storage.

---

# Types of Storage in Kubernetes

Kubernetes provides several storage options.

```
emptyDir

hostPath

Persistent Volume

Persistent Volume Claim

StorageClass

CSI Drivers

Cloud Storage
```

StatefulSet generally uses

```
Persistent Volume Claims
```

---

# What is a Persistent Volume (PV)?

A Persistent Volume (PV) is a piece of storage inside the Kubernetes cluster.

Think of it as a virtual hard disk.

The storage may come from

- AWS EBS
- Azure Disk
- Google Persistent Disk
- NFS
- Ceph
- Local Storage

Example

```
AWS EBS Volume

↓

Persistent Volume

↓

Kubernetes
```

---

# What is a Persistent Volume Claim (PVC)?

A PVC is a request for storage.

Example

Application

↓

Needs 5 GB

↓

Creates PVC

↓

PVC finds matching PV

↓

Storage attached

The application never directly uses the PV.

It always uses the PVC.

---

# Why PVC Instead of PV?

Suppose you buy a hotel room.

You don't own the building.

You only reserve one room.

Similarly,

Applications don't use the entire storage infrastructure.

They simply request storage through PVC.

PVC acts as the reservation.

PV acts as the actual storage.

---

# StorageClass

Modern Kubernetes clusters rarely create PVs manually.

Instead,

they use a StorageClass.

StorageClass automatically creates storage whenever a PVC is created.

This process is called

```
Dynamic Provisioning
```

---

# Dynamic Provisioning

Without StorageClass

```
Admin

↓

Create PV

↓

Application uses PVC
```

With StorageClass

```
Application creates PVC

↓

StorageClass

↓

Automatically creates PV

↓

Application starts
```

Much easier.

---

# AWS Example

Our Kubernetes cluster was running on AWS.

Whenever StatefulSet created

```
mongo-data-mongodb-0
```

Kubernetes automatically provisioned

```
AWS EBS Volume
```

Similarly

```
mongo-data-mongodb-1
```

received another EBS volume.

Each MongoDB Pod had its own dedicated storage.

---

# volumeClaimTemplates

One of the most powerful StatefulSet features is

```
volumeClaimTemplates
```

Instead of manually creating PVCs,

StatefulSet automatically creates one PVC for every Pod.

Example

```yaml
volumeClaimTemplates:

- metadata:
    name: mongo-data

  spec:

    accessModes:

    - ReadWriteOnce

    resources:

      requests:

        storage: 3Gi
```

Suppose

```
replicas: 2
```

StatefulSet automatically creates

```
mongo-data-mongodb-0

mongo-data-mongodb-1
```

Each Pod receives its own PVC.

No manual work is required.

---

# Complete Storage Architecture

```
MongoDB-0

↓

PVC

↓

PV

↓

AWS EBS Volume

---------------------------------

MongoDB-1

↓

PVC

↓

PV

↓

AWS EBS Volume
```

Each Pod has independent storage.

Deleting one Pod does not affect the storage of another Pod.

---

# Why StatefulSet Uses Separate PVCs

Imagine both MongoDB Pods shared one disk.

```
MongoDB-0

↓

Shared Disk

↑

MongoDB-1
```

This could easily corrupt the database.

Instead,

each MongoDB Pod gets its own volume.

```
MongoDB-0

↓

PVC-0

↓

PV-0

--------------------

MongoDB-1

↓

PVC-1

↓

PV-1
```

This is much safer.

---

# Summary

In this section we learned:

- Why StatefulSet requires a Headless Service.
- How Headless Service differs from a normal ClusterIP Service.
- How Kubernetes provides stable DNS records for StatefulSet Pods.
- How DNS names like `mongodb-0.mongodb-svc.default.svc.cluster.local` are formed.
- The role of EndpointSlices in DNS resolution.
- The purpose of Persistent Volumes (PV) and Persistent Volume Claims (PVC).
- How StorageClasses enable dynamic provisioning of storage.
- How StatefulSet uses `volumeClaimTemplates` to automatically create a dedicated PVC for every Pod.

With these networking and storage concepts in place, we are now ready to build our first MongoDB StatefulSet and understand every step of the practical deployment in detail.

# Part 3 - Practical 1 : Deploying MongoDB using StatefulSet (Without ReplicaSet)

---

# Objective

In this practical, we will deploy MongoDB using Kubernetes StatefulSet.

Our goal is to understand:

- How StatefulSet creates Pods
- Stable Pod Names
- Stable DNS
- Headless Service
- Pod Identity
- Creating Database
- Creating Collection
- Inserting Data
- Data Persistence
- Why StatefulSet alone does NOT replicate data

**Important**

This practical does **NOT** configure MongoDB Replication.

We are only deploying MongoDB as a Stateful Application.

Replication will be configured in the next practical.

---

# Lab Architecture

Initially our architecture is very simple.

```
                Kubernetes Cluster
-----------------------------------------------------

          Headless Service
            mongodb-svc
                 |
      -------------------------
      |                       |
  mongodb-0              mongodb-1

```

At this stage,

both Pods are completely independent.

They are NOT sharing database records.

They only have

- Stable Identity
- Stable Hostname
- Stable Storage

---

# Step 1 - Create StatefulSet YAML

First create a StatefulSet manifest.

```
vim mongosts.yml
```

Inside this file,

we define

- MongoDB Image
- Replica Count
- Container Port
- StatefulSet Name
- volumeClaimTemplates

Example

```yaml
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: mongodb

spec:
  serviceName: mongodb-svc

  replicas: 2

  selector:
    matchLabels:
      app: mongodb

  template:

    metadata:
      labels:
        app: mongodb

    spec:

      containers:

      - name: mongodb

        image: mongo:7.0

        ports:

        - containerPort: 27017

        volumeMounts:

        - name: mongo-data
          mountPath: /data/db

  volumeClaimTemplates:

  - metadata:
      name: mongo-data

    spec:

      accessModes:
      - ReadWriteOnce

      resources:

        requests:
          storage: 3Gi
```

---

# Understanding This YAML

Let us understand each important field.

## replicas

```
replicas: 2
```

This creates

```
mongodb-0

mongodb-1
```

Unlike Deployment,

Pods are numbered.

---

## serviceName

```
serviceName: mongodb-svc
```

This connects StatefulSet with the Headless Service.

Without this,

stable DNS will not work.

---

## volumeClaimTemplates

```
volumeClaimTemplates
```

Automatically creates

```
mongo-data-mongodb-0

mongo-data-mongodb-1
```

Each Pod receives its own PVC.

---

# Step 2 - Deploy StatefulSet

Apply the YAML.

```
kubectl apply -f mongosts.yml
```

Expected Output

```
statefulset.apps/mongodb created
```

---

# Step 3 - Verify StatefulSet

Check StatefulSet

```
kubectl get sts
```

Output

```
NAME

mongodb
```

---

# Step 4 - Verify Pods

```
kubectl get pods
```

Output

```
mongodb-0

mongodb-1
```

Notice

Pods are created sequentially.

```
mongodb-0

↓

Running

↓

mongodb-1

↓

Running
```

This demonstrates OrderedReady Policy.

---

# Step 5 - Verify PVC

```
kubectl get pvc
```

Output

```
mongo-data-mongodb-0

mongo-data-mongodb-1
```

Each Pod has its own storage.

---

# Step 6 - Verify PV

```
kubectl get pv
```

Output

```
pv-xxxxx

pv-yyyyy
```

Since our cluster is on AWS,

these PVs are backed by AWS EBS Volumes.

---

# Step 7 - Create Headless Service

Create another file

```
vim sts-headless-svc.yml
```

Example

```yaml
apiVersion: v1

kind: Service

metadata:
  name: mongodb-svc

spec:

  clusterIP: None

  selector:
    app: mongodb

  ports:

  - port: 27017
```

Apply

```
kubectl apply -f sts-headless-svc.yml
```

---

# Step 8 - Verify Service

```
kubectl get svc
```

Output

```
mongodb-svc

ClusterIP

None
```

Notice

ClusterIP is

```
None
```

This confirms

it is a Headless Service.

---

# Step 9 - Verify DNS

```
kubectl exec -it mongodb-0 -- hostname
```

Output

```
mongodb-0
```

Now

```
kubectl exec -it mongodb-0 -- hostname -f
```

Output

```
mongodb-0.mongodb-svc.default.svc.cluster.local
```

Similarly

```
mongodb-1.mongodb-svc.default.svc.cluster.local
```

Each Pod has a permanent hostname.

---

# Step 10 - Connect to MongoDB

Enter Mongo Shell

```
kubectl exec -it mongodb-0 -- mongosh
```

Mongo Shell opens.

```
test>
```

---

# Step 11 - Create Database

Create a database.

```
use vpcollege
```

Output

```
switched to db vpcollege
```

Many beginners think

the database is now permanently created.

This is NOT true.

At this moment,

MongoDB has only changed the current database context.

No data exists yet.

---

# Step 12 - Verify Database

Run

```
show dbs
```

Surprisingly,

our new database

```
vpcollege
```

is NOT visible.

Many beginners become confused here.

---

# Why Isn't Database Visible?

MongoDB only displays databases that contain data.

An empty database is not shown.

For example

```
use vpcollege
```

does NOT create

```
vpcollege
```

physically.

Only after inserting the first document

does MongoDB actually create the database.

This is an important MongoDB interview question.

---

# Step 13 - Create Collection

Collections in MongoDB are similar to tables in relational databases.

Example

```
Employees
```

---

# Step 14 - Insert Data

Insert the first document.

```
db.employees.insertOne({

    name:"Abhi",

    age:20,

    role:"DevOps Engineer"

})
```

Output

```
acknowledged: true
```

Now

MongoDB creates

```
Database

↓

Collection

↓

Document
```

---

# Step 15 - Verify Data

Run

```
db.employees.find()
```

Output

```
Abhi

20

DevOps Engineer
```

Our document has been inserted successfully.

---

# Step 16 - Verify Database

Now execute

```
show dbs
```

This time

```
vpcollege
```

appears.

Reason

The database now contains data.

---

# Step 17 - Exit Mongo Shell

```
exit
```

---

# Step 18 - Login to Second Pod

Now enter

```
kubectl exec -it mongodb-1 -- mongosh
```

Again

```
show dbs
```

Result

```
vpcollege

NOT FOUND
```

---

# Observation

This surprises many beginners.

We created

```
2 MongoDB Pods
```

Still

our data exists only in

```
mongodb-0
```

Why?

---

# Important Concept

A StatefulSet does NOT replicate application data.

StatefulSet only provides

- Stable Identity
- Stable DNS
- Stable Storage
- Ordered Deployment

It never copies MongoDB data between Pods.

Replication is always handled by the application itself.

For MongoDB,

this means

```
MongoDB Replica Set
```

must be configured.

---

# Why This Happens

Current Architecture

```
mongodb-0

↓

Own Database

↓

Own PVC

↓

Own EBS

----------------------------

mongodb-1

↓

Own Database

↓

Own PVC

↓

Own EBS
```

Notice

Each Pod owns completely separate storage.

StatefulSet intentionally isolates storage.

It does NOT synchronize data.

---

# Common Misconception

Many beginners believe

```
StatefulSet

=

Replication
```

This is completely incorrect.

StatefulSet only manages Pods.

It does NOT understand MongoDB,

MySQL,

PostgreSQL,

or Kafka.

Each application is responsible for its own replication mechanism.

Examples

MongoDB

↓

Replica Set

--------------------

MySQL

↓

Group Replication

--------------------

PostgreSQL

↓

Streaming Replication

--------------------

Kafka

↓

Broker Replication

---

# Conclusion

At the end of this practical, we successfully learned:

- How to deploy MongoDB using StatefulSet.
- How StatefulSet creates Pods with stable identities.
- How Headless Services provide stable DNS.
- How Persistent Volume Claims are automatically created.
- How to create a MongoDB database and collection.
- Why an empty MongoDB database is not shown in `show dbs`.
- Why each StatefulSet Pod has independent storage.
- Most importantly, we proved that **StatefulSet does not replicate application data**.

This observation becomes the motivation for the next practical, where we will configure a **MongoDB Replica Set** to enable automatic data replication between `mongodb-0` and `mongodb-1`.

# Part 4 - Configuring MongoDB Replica Set on Kubernetes StatefulSet

---

# Introduction

In the previous practical, we successfully deployed MongoDB using StatefulSet.

Our StatefulSet provided:

- Stable Pod Names
- Stable DNS
- Stable Storage
- Persistent Volumes

However, one major problem still existed.

We inserted data into

```
mongodb-0
```

but the same data was **not available** inside

```
mongodb-1
```

This proved that StatefulSet alone does **not** replicate application data.

StatefulSet only manages Pods.

Data replication must always be handled by the application itself.

For MongoDB, this feature is called a **Replica Set**.

---

# What is a MongoDB Replica Set?

A Replica Set is a group of MongoDB instances that maintain the same data automatically.

Instead of having independent databases,

all MongoDB instances become members of one database cluster.

Example

```
             Replica Set (rs0)

        -------------------------
        |                       |
    mongodb-0              mongodb-1
     Primary              Secondary

```

Whenever data is written to the Primary,

MongoDB automatically copies it to every Secondary.

---

# Why Do We Need Replica Set?

Suppose our application stores employee information.

Without Replica Set

```
mongodb-0

↓

Insert Employee

↓

Stored only in mongodb-0
```

If mongodb-0 crashes,

all recent writes are unavailable.

---

With Replica Set

```
Application

↓

Primary

↓

Secondary

↓

Secondary
```

Every write is copied automatically.

This provides:

- High Availability
- Data Redundancy
- Automatic Failover
- Data Replication

---

# Components of a Replica Set

A Replica Set generally consists of three types of members.

## Primary

The Primary is the main database.

Responsibilities

- Accepts Read Requests
- Accepts Write Requests
- Replicates Data
- Handles Client Connections

Only one Primary exists.

Example

```
mongodb-0
```

---

## Secondary

Secondary servers continuously copy data from the Primary.

Responsibilities

- Receive replicated data
- Can serve read requests
- Participate in elections

Example

```
mongodb-1
```

---

## Arbiter (Optional)

An Arbiter does not store data.

It only participates in voting.

Example

```
Primary

↓

Secondary

↓

Arbiter
```

Useful when an odd number of votes is required.

Our project used only

```
Primary

Secondary
```

No Arbiter.

---

# Replica Set Architecture

```
                rs0

         -----------------

        Primary

      mongodb-0

           |

Replication

           |

      mongodb-1

      Secondary
```

---

# Important Difference

Many beginners think

```
StatefulSet

=

Replication
```

This is incorrect.

StatefulSet provides

- Stable Pod Names
- Stable Storage
- Stable DNS

Replica Set provides

- Data Replication
- Primary Election
- Automatic Failover

These are completely different technologies.

---

# Enabling Replica Set in MongoDB

MongoDB must start with

```
--replSet
```

parameter.

Inside StatefulSet,

we modified

```yaml
containers:

- name: mongodb

  image: mongo:7.0

  command:

  - mongod

  - --bind_ip_all

  - --replSet

  - rs0
```

---

# Why --bind_ip_all?

By default,

MongoDB only listens on localhost.

Replica members must communicate over the Kubernetes network.

Therefore,

we enabled

```
--bind_ip_all
```

This allows MongoDB to accept connections from other Pods.

---

# Apply Updated StatefulSet

After modifying the StatefulSet,

apply it again.

```
kubectl apply -f mongosts.yml
```

Verify Pods

```
kubectl get pods
```

Both Pods should restart successfully.

---

# Verify Replica Set Status

Login

```
kubectl exec -it mongodb-0 -- mongosh
```

Check status

```
rs.status()
```

Initially,

MongoDB returns

```
NotYetInitialized
```

Reason

Replica Set is enabled,

but not initialized.

---

# Getting Full Hostname

Before initializing Replica Set,

MongoDB requires the full hostname of every member.

Execute

```
kubectl exec -it mongodb-0 -- hostname -f
```

Output

```
mongodb-0.mongodb-svc.default.svc.cluster.local
```

Similarly

```
kubectl exec -it mongodb-1 -- hostname -f
```

Output

```
mongodb-1.mongodb-svc.default.svc.cluster.local
```

These names remain permanent because StatefulSet provides stable identity.

---

# Why Did We Use hostname -f?

Many people manually type

```
mongodb-0.mongodb-svc
```

Although it often works,

the safest approach is

```
hostname -f
```

because MongoDB stores the exact hostname inside Replica Set configuration.

Using incorrect names can cause Replica Set communication failures.

---

# Initialize Replica Set

Inside mongosh

Run

```javascript
rs.initiate({

_id:"rs0",

members:[

{

_id:0,

host:"mongodb-0.mongodb-svc.default.svc.cluster.local:27017"

},

{

_id:1,

host:"mongodb-1.mongodb-svc.default.svc.cluster.local:27017"

}

]

})
```

Expected Output

```
ok:1
```

Replica Set has now been created.

---

# Check Replica Set Status

Execute

```
rs.status()
```

Output becomes much larger.

Important observations

```
set : rs0
```

```
PRIMARY
```

```
SECONDARY
```

You should see

```
mongodb-0

PRIMARY
```

and

```
mongodb-1

SECONDARY
```

---

# Verify Replica Set Configuration

Run

```
rs.conf()
```

This displays

- Replica Set Name
- Members
- Hostnames
- Votes
- Priorities

Example

```
mongodb-0

Priority

1

----------------

mongodb-1

Priority

1
```

---

# Test Replication

Create a database.

```
use codingwale
```

Insert data.

```
db.students.insertOne({

name:"Vishal",

age:20,

role:"DevOps Engineer"

})
```

Verify

```
db.students.find()
```

Document appears.

---

# Check Secondary

Open another terminal.

Login

```
kubectl exec -it mongodb-1 -- mongosh
```

Execute

```
show dbs
```

Database appears.

Now

```
use codingwale
```

Run

```
db.students.find()
```

The same document appears.

This confirms

Replication is working successfully.

---

# What Happened Behind the Scenes?

Application

↓

Primary

↓

MongoDB Replication Engine

↓

Secondary

↓

Storage

Unlike StatefulSet,

MongoDB itself copied the data.

---

# Our Observation

Before Replica Set

```
mongodb-0

↓

Employee

Visible

---------------

mongodb-1

↓

Employee

Not Visible
```

After Replica Set

```
mongodb-0

↓

Employee

↓

Replication

↓

mongodb-1

↓

Employee Visible
```

Problem solved.

---

# Important Interview Question

**Question**

Does StatefulSet replicate database records?

**Answer**

No.

StatefulSet only provides

- Stable Identity
- Stable Storage
- Stable Network

Database replication is handled by the application.

For MongoDB,

Replication is provided by Replica Set.

---

# Another Common Interview Question

**Question**

Why do MongoDB Pods require stable DNS?

**Answer**

Replica Set members communicate using hostnames.

If hostnames change,

Replica Set communication breaks.

StatefulSet guarantees permanent hostnames.

---

# Real Issues We Faced During This Practical

While configuring Replica Set, we encountered several problems.

### Issue 1

Data inserted into mongodb-0 was not visible in mongodb-1.

Reason

Replica Set was not configured.

Solution

Enable

```
--replSet rs0
```

and initialize using

```
rs.initiate()
```

---

### Issue 2

Incorrect MongoDB hostname.

Initially,

we tried using incorrect DNS names.

MongoDB could not communicate correctly.

Solution

Use

```
hostname -f
```

and copy the exact FQDN into

```
rs.initiate()
```

---

### Issue 3

Confusion between StatefulSet and Replica Set.

Initially,

it appeared that StatefulSet should automatically replicate data.

After testing,

we confirmed that StatefulSet only manages infrastructure.

MongoDB Replica Set manages data replication.

---

# Summary

In this practical, we successfully converted two independent MongoDB Pods into a highly available MongoDB Replica Set.

We learned:

- Why StatefulSet alone is not enough.
- Difference between StatefulSet and Replica Set.
- Primary and Secondary architecture.
- Importance of `--replSet`.
- Why `--bind_ip_all` is required.
- Why stable DNS is critical.
- How to initialize a Replica Set using `rs.initiate()`.
- How to verify the configuration with `rs.status()` and `rs.conf()`.
- How MongoDB automatically replicates data between members.

At this point, our MongoDB cluster is production-ready from a replication perspective and can safely serve applications running inside Kubernetes.

# Part 5 - Building a Complete NodeJS + MongoDB ReplicaSet Application on Kubernetes

---

# Introduction

So far, we have learned:

- StatefulSet
- Headless Service
- Persistent Volume
- Persistent Volume Claim
- Stable DNS
- MongoDB Replica Set

At this stage, we have only deployed MongoDB.

Now we need an application that can actually use the database.

Instead of manually opening Mongo Shell every time,

we will create a backend application.

Our backend will

- Connect to MongoDB Replica Set
- Insert Data
- Read Data
- Return JSON to clients

Later,

our React Frontend will consume this API.

---

# Complete Project Architecture

```
                Browser
                    │
                    │ HTTP Request
                    ▼
        React Frontend (NodePort)
                    │
                    │ Fetch API
                    ▼
         NodeJS Backend (Deployment)
                    │
                    │ MongoDB Driver
                    ▼
       MongoDB Replica Set (StatefulSet)
             │                  │
             ▼                  ▼
          mongodb-0         mongodb-1
           PRIMARY         SECONDARY
             │                  │
             ▼                  ▼
          PVC + EBS         PVC + EBS
```

Notice that the browser never talks directly to MongoDB.

Instead,

the communication always follows

```
Browser

↓

Frontend

↓

Backend

↓

Database
```

This is how almost every real-world application works.

---

# Why Do We Need a Backend?

Many beginners ask,

> Why can't React directly connect to MongoDB?

Answer:

Because MongoDB should never be exposed to users.

Imagine this architecture

```
Browser

↓

MongoDB
```

Anyone could

- Delete Data
- Modify Data
- Drop Database
- Read Sensitive Information

This is a huge security risk.

Instead,

we use

```
Browser

↓

Backend

↓

MongoDB
```

The backend controls

- Authentication
- Authorization
- Validation
- Business Logic
- Database Access

---

# Backend Responsibilities

Our NodeJS backend performs the following tasks:

- Connect to MongoDB
- Insert Employees
- Read Employees
- Return JSON
- Hide Database Credentials
- Handle Business Logic

---

# Backend Source Code

Our backend was written using

- ExpressJS
- MongoDB Node Driver

Example

```javascript
const express = require("express");
const { MongoClient } = require("mongodb");

const app = express();

app.use(express.json());
```

---

# Connecting to MongoDB Replica Set

One of the most important parts of the application is the MongoDB Connection String.

```javascript
const url =
"mongodb://mongodb-0.mongodb-svc:27017,mongodb-1.mongodb-svc:27017/?replicaSet=rs0";
```

Let's understand every part.

---

## mongodb://

Specifies MongoDB protocol.

---

## mongodb-0.mongodb-svc

Stable DNS of Primary Pod.

---

## mongodb-1.mongodb-svc

Stable DNS of Secondary Pod.

---

## replicaSet=rs0

This tells the MongoDB driver

"This database is a Replica Set."

The driver automatically discovers

- Primary
- Secondary
- Elections
- Failover

---

# Why Not Use Pod IP?

Suppose we use

```
100.96.1.113
```

instead of

```
mongodb-0.mongodb-svc
```

Tomorrow

Pod crashes.

New Pod receives

```
100.96.4.81
```

Application immediately breaks.

DNS never changes.

Therefore

always use DNS.

Never use Pod IP.

---

# Connecting to MongoDB

After creating MongoClient

```javascript
const client = new MongoClient(url);
```

Connection

```javascript
await client.connect();
```

If successful

```
Connected to MongoDB Replica Set
```

appears in Pod Logs.

Verify

```
kubectl logs node-app
```

---

# Creating Database

After connecting

```javascript
const db = client.db("codingwale");
```

MongoDB automatically creates

```
codingwale
```

when the first document is inserted.

---

# Creating Collection

```javascript
const collection =
db.collection("students");
```

If the collection doesn't exist,

MongoDB automatically creates it.

---

# Insert API

Our backend provides

```
/add
```

Example

```javascript
app.get("/add", async(req,res)=>{

await collection.insertOne({

name:"Aarush",

age:25,

created:new Date()

});

});
```

Whenever

```
http://NodeIP:31557/add
```

is opened,

a new document is inserted.

---

# Read API

Our second endpoint

```
/
```

returns every student.

```javascript
app.get("/",async(req,res)=>{

const students=

await collection.find().toArray();

res.json(students);

});
```

Output

```json
[
{
"name":"Vishal",
"age":20
}
]
```

---

# Why Return JSON?

Frontend applications understand JSON.

Instead of returning HTML,

our backend returns

```
JSON
```

React reads

```
JSON

↓

JavaScript Object

↓

Display Data
```

---

# Dockerizing the Backend

Instead of installing NodeJS inside Kubernetes,

we package everything into a Docker Image.

Dockerfile

```Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["node","server.js"]
```

---

# Building Docker Image

```
docker build -t sts-back-img:v1 .
```

Verify

```
docker images
```

---

# Tag Image

```
docker tag

sts-back-img:v1

username/sts-back-img:v1
```

---

# Push Image

```
docker push username/sts-back-img:v1
```

Now Kubernetes can download it.

---

# Creating Deployment

Example

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:
  name: node-app

spec:

  replicas: 1

  selector:

    matchLabels:

      app: node-app

  template:

    metadata:

      labels:

        app: node-app

    spec:

      containers:

      - name: node-app

        image: username/sts-back-img:v1

        ports:

        - containerPort:3000
```

Apply

```
kubectl apply -f nodejs-deploy.yml
```

---

# Verify Pod

```
kubectl get pods
```

Output

```
node-app

Running
```

---

# Expose Backend

Create Service

```yaml
apiVersion: v1

kind: Service

metadata:

  name: node-service

spec:

  selector:

    app: node-app

  ports:

  - port:3000

    targetPort:3000

  type: NodePort
```

Apply

```
kubectl apply -f nodejs-svc.yml
```

---

# Verify Service

```
kubectl get svc
```

Example

```
NodePort

31557
```

Application becomes accessible

```
http://WorkerNodeIP:31557
```

---

# Testing Backend

Open Browser

```
http://WorkerNodeIP:31557
```

Output

```json
[
{

"name":"Vishal",

"age":20

}
]
```

Backend successfully connected.

---

# Testing Insert API

Open

```
http://WorkerNodeIP:31557/add
```

Refresh

```
http://WorkerNodeIP:31557
```

New document appears.

---

# Verifying MongoDB

Login

```
kubectl exec -it mongodb-0 -- mongosh
```

Switch Database

```
use codingwale
```

Verify

```
db.students.find().pretty()
```

Output

```
Aarush

Vishal

...
```

Everything matches.

---

# Data Flow

```
Browser

↓

NodeJS

↓

Mongo Driver

↓

Primary

↓

MongoDB Replication

↓

Secondary

↓

Storage
```

Notice

NodeJS only communicates with

Replica Set.

It does not know

which server is Primary.

MongoDB Driver automatically discovers

the Primary.

---

# Automatic Failover

Suppose

```
mongodb-0

PRIMARY
```

crashes.

Replica Set elects

```
mongodb-1

PRIMARY
```

NodeJS continues working.

No code changes required.

The MongoDB Driver automatically reconnects.

This is one of the biggest advantages of using Replica Sets.

---

# Common Mistakes We Faced

During this project, we encountered several real-world problems.

## Wrong DNS

Initially

```javascript
mongodb://mongodb-0.mongodb
```

Error

```
ENOTFOUND
```

Solution

```
mongodb://mongodb-0.mongodb-svc
```

---

## Wrong Replica Set Name

Using

```
replicaSet=test
```

instead of

```
rs0
```

causes connection failure.

Always match

```
rs.initiate()

↓

_id

↓

rs0
```

---

## Old Docker Image

We modified

```
server.js
```

but forgot to build and push a new image.

Kubernetes kept running the old code.

Solution

```
docker build

↓

docker tag

↓

docker push

↓

kubectl rollout restart deployment node-app
```

---

## CrashLoopBackOff

The application crashed because

```
cors
```

package was missing.

Solution

```
npm install cors
```

Rebuild image.

Push.

Restart Deployment.

---

# Summary

In this practical, we successfully connected a NodeJS backend to a MongoDB Replica Set running inside Kubernetes.

We learned:

- Why applications should never connect directly from the browser to MongoDB.
- How to use stable Kubernetes DNS instead of Pod IP addresses.
- How the MongoDB driver automatically discovers the Primary node in a Replica Set.
- How to containerize a NodeJS application with Docker.
- How to deploy the backend using a Kubernetes Deployment.
- How to expose the backend using a NodePort Service.
- How to verify that data inserted through the API is actually stored in MongoDB and replicated to the Replica Set.

At this stage, our backend is production-ready and exposes REST APIs that can be consumed by any frontend application. In the next section, we will deploy a React frontend, connect it to the backend, and build a complete three-tier application running entirely on Kubernetes.

# Part 6 - Deploying React Frontend and Building a Complete Three-Tier Kubernetes Application

---

# Introduction

At this stage, we have successfully deployed:

- MongoDB StatefulSet
- MongoDB Replica Set
- NodeJS Backend

The only missing component is the Frontend.

Until now,

we were accessing the backend directly from the browser.

Example

```
http://WorkerNodeIP:31557
```

The backend returned JSON.

Although this works,

real users do not interact with raw JSON.

Instead,

they use a frontend application.

In our project,

we used ReactJS as the frontend.

---

# Three-Tier Architecture

A modern web application is usually divided into three layers.

```
Presentation Layer

↓

Business Layer

↓

Database Layer
```

In our project,

these layers are

```
React

↓

NodeJS

↓

MongoDB
```

---

# Complete Architecture

```
                 User

                  │

                  ▼

          React Frontend

           Deployment

               │

         NodePort Service

               │

               ▼

        NodeJS Backend

          Deployment

               │

         NodePort Service

               │

               ▼

      MongoDB Replica Set

         StatefulSet

       mongodb-0 (PRIMARY)

       mongodb-1 (SECONDARY)

               │

               ▼

         Persistent Storage

              PVC

               │

               ▼

             AWS EBS
```

This is the exact architecture we built during our practical sessions.

---

# Why React Uses Deployment Instead of StatefulSet

This is a very common interview question.

React is a **Stateless Application**.

It does not store user data.

Whenever a React Pod starts,

it simply serves static HTML,

JavaScript,

and CSS files.

If the Pod crashes,

Kubernetes creates another Pod.

Nothing is lost.

Therefore,

React does NOT require

- Stable Storage
- Stable Identity
- Stable Hostname

Deployment is sufficient.

---

# Deployment Comparison

React

```
Deployment
```

NodeJS

```
Deployment
```

MongoDB

```
StatefulSet
```

Reason

React

↓

No State

NodeJS

↓

No State

MongoDB

↓

Stores Data

Needs Stable Storage

Needs Stable Identity

Needs Replica Set

---

# React Source Code

Our React application fetches data from the backend.

Example

```javascript
useEffect(()=>{

fetch("http://WorkerNodeIP:31557")

.then(res=>res.json())

.then(data=>{

setEmployees(data);

});

},[])
```

Whenever the page loads,

React sends an HTTP request to the backend.

---

# Request Flow

```
Browser

↓

React

↓

Fetch()

↓

NodeJS API

↓

MongoDB

↓

NodeJS

↓

React

↓

Browser
```

The browser never communicates directly with MongoDB.

---

# React Components

Our frontend contains

```
App.jsx
```

Responsibilities

- Fetch Employee Data
- Store Response
- Display Table
- Handle Loading State

---

# Employee Table

The response received from NodeJS is

```json
[
{

"name":"Vishal",

"age":20,

"role":"DevOps Engineer"

}
]
```

React converts this JSON into

```
HTML Table
```

using

```javascript
employees.map(...)
```

---

# Dockerizing React

Like every application,

React is packaged inside a Docker Image.

Example Dockerfile

```Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

EXPOSE 4173

CMD ["npm","run","preview","--","--host"]
```

> **Note:** For a Vite application, the production image should run `npm run build` and then serve the built files (for example with `vite preview` or, more commonly in production, with Nginx). In our classroom project we initially experimented with a different Dockerfile before correcting it.

---

# Build Image

```
docker build -t react-img:v1 .
```

---

# Push Image

```
docker tag react-img:v1 username/react-img:v1

docker push username/react-img:v1
```

---

# React Deployment

Example

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: react-app

spec:

  replicas:1

  selector:

    matchLabels:

      app:react-app

  template:

    metadata:

      labels:

        app:react-app

    spec:

      containers:

      - name:react

        image:username/react-img:v1
```

Deploy

```
kubectl apply -f react-deploy.yml
```

---

# Verify Deployment

```
kubectl get pods
```

Output

```
react-app

Running
```

---

# Expose React

Create Service

```yaml
kind: Service

type: NodePort
```

Apply

```
kubectl apply -f react-service.yml
```

Verify

```
kubectl get svc
```

Example

```
30314
```

Browser

```
http://WorkerNodeIP:30314
```

React application loads successfully.

---

# Request Lifecycle

Suppose the browser opens

```
http://WorkerNodeIP:30314
```

The sequence becomes

```
Browser

↓

React Page

↓

Fetch()

↓

http://WorkerNodeIP:31557

↓

NodeJS

↓

MongoDB

↓

NodeJS

↓

JSON

↓

React

↓

HTML Table
```

---

# Our First Problem

Initially,

the page loaded,

but

```
No Data
```

appeared.

Why?

The backend API worked perfectly.

```
curl

↓

Success
```

React

↓

Failed.

---

# Root Cause

The browser blocked the request.

Reason

```
CORS
```

(Cross-Origin Resource Sharing)

React

```
http://WorkerNode:30314
```

Backend

```
http://WorkerNode:31557
```

Different Origins.

Modern browsers block these requests by default.

---

# Solution

Install

```
cors
```

package.

```
npm install cors
```

Then

```javascript
const cors=require("cors");

app.use(cors());
```

Rebuild Docker Image.

Push Image.

Restart Deployment.

Problem solved.

---

# Our Second Problem

The application still crashed.

Error

```
Cannot find module

cors
```

Reason

We modified

```
server.js
```

but forgot

```
npm install cors
```

Solution

```
npm install cors

↓

docker build

↓

docker push

↓

kubectl rollout restart
```

---

# Our Third Problem

NodeJS still used

old code.

Reason

Docker Image

```
v1
```

was still running.

We forgot

```
docker push
```

or

```
kubectl rollout restart
```

Solution

```
docker build

↓

docker push

↓

kubectl rollout restart deployment node-app
```

---

# Our Fourth Problem

Initially,

the backend could not connect to MongoDB.

Error

```
ENOTFOUND
```

Reason

Wrong DNS.

Incorrect

```
mongodb-0.mongodb
```

Correct

```
mongodb-0.mongodb-svc
```

or

```
mongodb-0.mongodb-svc.default.svc.cluster.local
```

After updating the connection string,

the backend connected successfully.

---

# Our Fifth Problem

React loaded,

but browser displayed

```
Loading...
```

forever.

Backend worked.

MongoDB worked.

Reason

CORS.

After enabling

```
app.use(cors())
```

everything started working.

---

# Our Sixth Problem

NodePort was inaccessible.

Browser

↓

Timeout

Reason

AWS Security Group.

Initially,

we opened the NodePort

on the Control Plane.

NodePort traffic actually reaches the **Worker Node**.

Solution

Open

```
30314

31557
```

on the Worker Node Security Group.

After that,

everything worked correctly.

---

# Complete Data Flow

When a user visits the application,

the following sequence occurs.

```
User

↓

React NodePort

↓

React Pod

↓

Fetch API

↓

NodeJS NodePort

↓

NodeJS Pod

↓

MongoDB Driver

↓

Replica Set

↓

Primary

↓

MongoDB

↓

Secondary

↓

NodeJS

↓

React

↓

Browser
```

This is exactly how our complete project functions.

---

# Final Production Architecture

```
                    Browser

                        │

                        ▼

              React Frontend

                 Deployment

                        │

                        ▼

               NodePort Service

                        │

                        ▼

               NodeJS Backend

                 Deployment

                        │

                        ▼

               NodePort Service

                        │

                        ▼

           MongoDB Replica Set

        mongodb-0 (PRIMARY)

        mongodb-1 (SECONDARY)

                        │

                Automatic Replication

                        │

                        ▼

             Persistent Volume Claims

                        │

                        ▼

                 AWS EBS Volumes
```

---

# Kubernetes Objects Used

During this project,

we used the following Kubernetes resources.

| Resource | Purpose |
|----------|----------|
| Deployment | React Frontend |
| Deployment | NodeJS Backend |
| StatefulSet | MongoDB |
| Headless Service | Stable DNS |
| NodePort Service | Browser Access |
| PersistentVolumeClaim | Storage Request |
| PersistentVolume | Storage |
| ReplicaSet (MongoDB) | Database Replication |

---

# What We Learned From This Project

By completing this project,

we learned how to build a complete production-style Kubernetes application.

We learned:

- Dockerizing applications.
- Deploying stateless applications using Deployments.
- Deploying stateful applications using StatefulSets.
- Creating Headless Services.
- Configuring MongoDB Replica Sets.
- Using Persistent Volumes and Persistent Volume Claims.
- Connecting applications using Kubernetes DNS.
- Exposing applications using NodePort Services.
- Handling browser CORS restrictions.
- Debugging DNS issues.
- Debugging CrashLoopBackOff.
- Rebuilding and updating Docker images.
- Rolling out Deployment updates.
- Verifying database replication.
- Building an end-to-end three-tier architecture.

This practical combined almost every major Kubernetes concept learned so far into one complete working project and demonstrated how stateless and stateful workloads work together inside a Kubernetes cluster.

# Part 7 - Troubleshooting, Production Best Practices and Common Mistakes

---

# Introduction

Building a Stateful Application on Kubernetes is much more challenging than deploying a simple stateless application.

Unlike Deployments,

StatefulSets involve multiple Kubernetes components working together.

For example

- StatefulSet
- Headless Service
- Persistent Volume Claim (PVC)
- Persistent Volume (PV)
- StorageClass
- DNS
- MongoDB Replica Set
- Docker Images
- NodeJS Backend
- React Frontend

If any one of these components is misconfigured,

the entire application may fail.

During our project,

we encountered many real-world problems.

Understanding these problems is just as important as understanding the deployment itself because troubleshooting is a major part of a DevOps Engineer's daily work.

---

# Problem 1 - NodeJS Could Not Connect to MongoDB

## Symptoms

NodeJS Pod entered

```
CrashLoopBackOff
```

Logs showed

```
MongoServerSelectionError

ENOTFOUND mongodb-0.mongodb
```

---

## Root Cause

Our application was trying to connect to

```
mongodb-0.mongodb
```

However,

our Headless Service was actually named

```
mongodb-svc
```

Therefore,

Kubernetes DNS could not resolve

```
mongodb-0.mongodb
```

---

## Wrong Connection String

```javascript
mongodb://mongodb-0.mongodb:27017
```

---

## Correct Connection String

```javascript
mongodb://mongodb-0.mongodb-svc:27017,mongodb-1.mongodb-svc:27017/?replicaSet=rs0
```

---

## Lesson Learned

Always use the Headless Service name inside StatefulSet connection strings.

Never invent DNS names.

---

# Problem 2 - MongoDB Replica Set Was Not Working

## Symptoms

Data inserted into

```
mongodb-0
```

never appeared inside

```
mongodb-1
```

---

## Root Cause

StatefulSet was working correctly.

Replica Set was not configured.

Many beginners incorrectly believe

```
StatefulSet

=

Replication
```

This is false.

---

## Solution

Start MongoDB using

```
--replSet rs0
```

Initialize Replica Set

```
rs.initiate(...)
```

Verify

```
rs.status()
```

---

## Lesson Learned

StatefulSet manages infrastructure.

MongoDB Replica Set manages database replication.

These are completely different responsibilities.

---

# Problem 3 - Wrong Replica Set Hostnames

Initially,

we manually entered hostnames.

Some of them were incorrect.

Replica Set communication failed.

---

## Solution

Always verify the hostname using

```
hostname -f
```

Example

```
mongodb-0.mongodb-svc.default.svc.cluster.local
```

Use these exact values inside

```
rs.initiate()
```

---

# Problem 4 - React Could Not Display Data

React loaded successfully.

Browser opened correctly.

However,

Employee table remained empty.

---

## Root Cause

Backend API was working.

MongoDB was working.

Browser blocked the request.

Reason

```
CORS
```

---

## Solution

Install

```
cors
```

Enable

```javascript
const cors=require("cors");

app.use(cors());
```

Rebuild Docker Image.

Push Image.

Restart Deployment.

---

## Lesson Learned

Whenever Frontend and Backend run on different origins,

CORS must be configured.

---

# Problem 5 - Cannot Find Module 'cors'

After adding

```javascript
require("cors")
```

the application crashed.

Logs

```
Cannot find module

cors
```

---

## Root Cause

Package was never installed.

---

## Solution

```
npm install cors
```

Rebuild Docker Image.

Push.

Restart Deployment.

---

# Lesson Learned

Editing source code is not enough.

Every new dependency must also be installed.

---

# Problem 6 - Kubernetes Continued Running Old Code

We edited

```
server.js
```

but Kubernetes continued running the previous version.

---

## Root Cause

Docker Image was never rebuilt.

Kubernetes simply downloaded the old image.

---

## Solution

```
docker build

↓

docker tag

↓

docker push

↓

kubectl rollout restart deployment node-app
```

---

## Lesson Learned

Whenever application code changes,

always rebuild the Docker image.

---

# Problem 7 - Image Updated But Pod Still Used Old Version

Sometimes

```
docker push
```

was successful.

Still,

Pod continued using the old image.

---

## Root Cause

Image tag

```
v1
```

was unchanged.

Kubernetes assumed

the image already existed.

---

## Solution 1

Use a new tag

```
v1

↓

v2

↓

v3
```

---

## Solution 2

Use

```yaml
imagePullPolicy: Always
```

---

## Lesson Learned

Never reuse the same image tag during development.

---

# Problem 8 - NodePort Not Accessible

Browser

↓

Timeout

---

## Root Cause

AWS Security Group was configured incorrectly.

We opened

```
31557
```

on

```
Control Plane
```

instead of

```
Worker Node
```

NodePort traffic always reaches Worker Nodes.

---

## Solution

Open NodePort

```
30314

31557
```

inside Worker Node Security Group.

---

## Lesson Learned

NodePort belongs to Worker Nodes.

Control Plane never serves application traffic.

---

# Problem 9 - BusyBox Could Not Resolve DNS

Command

```
nslookup mongodb-0.mongodb-svc
```

returned

```
NXDOMAIN
```

---

## Confusion

We thought DNS was broken.

Later

```
nslookup

mongodb-0.mongodb-svc.default.svc.cluster.local
```

worked perfectly.

---

## Explanation

BusyBox's `nslookup` does not always use the Kubernetes search domains the same way that application libraries do.

The fully qualified domain name (FQDN) resolved correctly, which confirmed that Kubernetes DNS and the Headless Service were configured properly.

---

## Lesson Learned

When troubleshooting Kubernetes DNS,

always test the full hostname.

---

# Problem 10 - MongoDB Database Not Visible

Command

```
use codingwale
```

worked.

But

```
show dbs
```

did not display

```
codingwale
```

---

## Root Cause

MongoDB only creates a database after the first document is inserted.

---

## Solution

Insert

```
db.students.insertOne(...)
```

Now

```
show dbs
```

shows

```
codingwale
```

---

## Lesson Learned

An empty MongoDB database does not physically exist.

---

# Problem 11 - StatefulSet Does Not Share Data

Initially,

we assumed

```
mongodb-0

↓

Data

↓

mongodb-1
```

would happen automatically.

It did not.

---

## Root Cause

Each StatefulSet Pod owns its own storage.

```
mongodb-0

↓

PVC-0

↓

EBS-0

-------------------

mongodb-1

↓

PVC-1

↓

EBS-1
```

No shared storage exists.

---

## Solution

MongoDB Replica Set.

---

## Lesson Learned

Storage persistence

≠

Database replication.

---

# Problem 12 - CrashLoopBackOff

Pods repeatedly restarted.

---

## Troubleshooting Steps

Always follow the same order.

```
kubectl get pods
```

↓

```
kubectl describe pod
```

↓

```
kubectl logs
```

↓

Fix

↓

Restart

---

Never guess.

Always inspect the logs first.

---

# Problem 13 - Wrong DNS vs Wrong Application

Sometimes

developers blame Kubernetes.

Actually

the application itself is misconfigured.

Example

Wrong

```
mongodb://mongodb
```

Correct

```
mongodb://mongodb-0.mongodb-svc
```

---

# Problem 14 - Pod Recreation

Many beginners panic when Pods restart.

Remember

StatefulSet recreates

```
mongodb-0
```

not

```
mongodb-a92jd
```

Identity remains

the same.

Storage reconnects automatically.

---

# Production Best Practices

---

## Use Separate Storage For Every Database Pod

Never share one Persistent Volume among multiple database Pods unless the storage technology explicitly supports it.

Each StatefulSet member should own its own Persistent Volume Claim.

---

## Always Use Headless Service

Stateful applications should communicate using stable DNS names.

Never use Pod IP addresses.

---

## Never Hardcode Pod IPs

Bad

```
100.96.1.113
```

Good

```
mongodb-0.mongodb-svc
```

---

## Enable Health Probes

Use

```
Readiness Probe

Liveness Probe

Startup Probe
```

These improve availability and recovery.

---

## Resource Requests and Limits

Always define

```yaml
resources:

  requests:

  limits:
```

This prevents resource starvation.

---

## Use Secrets

Never store

```
Database Passwords

API Keys

JWT Secrets
```

inside YAML files.

Use Kubernetes Secrets.

---

## Use ConfigMaps

Move configuration outside the container.

Examples

```
Environment

Application Config

URLs

Feature Flags
```

---

## Backup Databases

Persistent Volumes reduce the risk of data loss,

but they are **not backups**.

Always implement a proper backup strategy.

---

## Monitor Stateful Applications

Use monitoring tools such as

- Prometheus
- Grafana
- MongoDB Exporter

Monitor

- CPU
- Memory
- Disk Usage
- Replication Lag
- PVC Usage

---

## Prefer Ingress Over Multiple NodePorts

NodePort is useful for learning.

In production,

Ingress is usually a better choice because it provides

- TLS termination
- Virtual hosts
- Path-based routing
- Centralized traffic management

---

# Debugging Checklist

Whenever a Stateful Application fails,

follow this sequence.

```
Pods Running?

↓

Services Created?

↓

PVC Bound?

↓

PV Available?

↓

DNS Working?

↓

Replica Set Healthy?

↓

Application Logs?

↓

NodePort Accessible?

↓

Security Group Open?

↓

Browser Errors?
```

This systematic approach avoids random guessing.

---

# Key Takeaways

By completing this project and troubleshooting every issue, we learned several important lessons:

- StatefulSet is responsible for **stable identity, networking, and storage**, not database replication.
- MongoDB Replica Set is responsible for **replicating data and electing a Primary**.
- Headless Services provide the stable DNS names required by stateful applications.
- PersistentVolumeClaims ensure that data survives Pod recreation.
- Docker images must be rebuilt and redeployed whenever application code changes.
- Browser applications often require CORS configuration when communicating with APIs.
- AWS Security Groups must allow traffic to the **Worker Node** NodePort, not just the Control Plane.
- Effective troubleshooting starts with `kubectl logs` and `kubectl describe` rather than guessing.

These lessons reflect real production scenarios and are valuable not only for interviews but also for day-to-day Kubernetes administration.

# Part 8 - Interview Questions, Revision Notes, Commands Cheat Sheet and Summary

---

# StatefulSet Interview Questions

## Q1. What is a StatefulSet?

Answer:

A StatefulSet is a Kubernetes workload controller used to deploy stateful applications.

It provides:

- Stable Pod names
- Stable DNS
- Stable Persistent Storage
- Ordered Deployment
- Ordered Scaling
- Ordered Deletion

Examples:

- MongoDB
- MySQL
- PostgreSQL
- Cassandra
- Kafka
- ZooKeeper
- Elasticsearch

---

## Q2. Why can't we use Deployment for databases?

Deployment creates interchangeable Pods.

Example

```
mysql-x81ab

↓

Crash

↓

mysql-93kjd
```

The Pod name changes.

The identity changes.

A database cannot tolerate this because other database members identify each other by hostname.

StatefulSet guarantees

```
mysql-0

↓

Crash

↓

mysql-0
```

Identity remains unchanged.

---

## Q3. Does StatefulSet replicate data?

No.

StatefulSet only manages Kubernetes objects.

Replication is handled by the database itself.

Examples

MongoDB

↓

Replica Set

MySQL

↓

Group Replication

PostgreSQL

↓

Streaming Replication

Kafka

↓

Broker Replication

---

## Q4. What is a Headless Service?

A Headless Service is a Service whose ClusterIP is

```
None
```

Example

```yaml
clusterIP: None
```

Instead of providing one virtual IP,

it exposes the DNS of every Pod individually.

Example

```
mongodb-0.mongodb-svc

mongodb-1.mongodb-svc
```

---

## Q5. Why does StatefulSet require a Headless Service?

Because Stateful applications need stable DNS.

Deployment Pods communicate through Services.

Stateful Pods communicate directly with each other.

Example

```
mongodb-0.mongodb-svc

↓

mongodb-1.mongodb-svc
```

---

## Q6. What happens when mongodb-0 crashes?

1.

StatefulSet recreates

```
mongodb-0
```

2.

Same PVC reconnects.

3.

Same EBS reconnects.

4.

Same DNS returns.

5.

MongoDB Replica Set synchronizes missing data.

---

## Q7. What happens if the entire Worker Node crashes?

Scheduler chooses another node.

↓

StatefulSet recreates Pod.

↓

PVC reconnects.

↓

PV reconnects.

↓

EBS attaches.

↓

Replica Set synchronizes.

Application continues.

---

## Q8. Why does each StatefulSet Pod get its own PVC?

Because every Pod owns its own storage.

Example

```
mongodb-0

↓

PVC-0

↓

PV-0

↓

EBS-0

-------------------

mongodb-1

↓

PVC-1

↓

PV-1

↓

EBS-1
```

This prevents storage conflicts.

---

## Q9. Can two Pods use the same EBS?

Normally,

No.

AWS EBS uses

```
ReadWriteOnce
```

Only one node can mount it at a time.

---

## Q10. Difference between PV and PVC?

PVC

↓

Storage Request

PV

↓

Actual Storage

Example

```
PVC

"I need 5GB"

↓

PV

"Here is 5GB"

↓

EBS
```

---

## Q11. What is StorageClass?

StorageClass automatically creates Persistent Volumes.

Without StorageClass

Administrator creates PV manually.

With StorageClass

PVC automatically creates PV.

---

## Q12. Why is MongoDB Stateful?

Because it stores

- Data
- Users
- Collections
- Indexes
- Transactions

The data must survive Pod recreation.

---

## Q13. Is NodeJS Stateful?

No.

NodeJS stores nothing permanently.

It simply processes requests.

If it crashes,

another Pod continues.

---

## Q14. Is React Stateful?

No.

React only serves static files.

No important information is stored inside the Pod.

---

## Q15. Can StatefulSet scale?

Yes.

Example

```
Replicas

2

↓

3
```

Creates

```
mongodb-2
```

Each Pod receives

its own PVC.

---

## Q16. Can StatefulSet perform Rolling Updates?

Yes.

Updates occur one Pod at a time,

starting from the highest ordinal.

---

## Q17. What happens if mongodb-1 is deleted?

StatefulSet recreates

```
mongodb-1
```

Same hostname

Same PVC

Same EBS

MongoDB synchronizes missing data.

---

## Q18. What happens if PVC is deleted?

StatefulSet creates a new empty PVC (if allowed by the storage configuration).

The previous data is lost unless the underlying Persistent Volume is retained and reattached.

Deleting a PVC should always be done carefully.

---

## Q19. Does StatefulSet guarantee application consistency?

No.

It guarantees Kubernetes object identity.

Application consistency is the responsibility of the application itself.

---

## Q20. Why does StatefulSet create Pods in order?

Because many distributed databases depend on ordered startup.

Example

```
mongodb-0

↓

mongodb-1

↓

mongodb-2
```

Not random order.

---

# Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
|----------|------------|-------------|
| Pod Names | Random | Fixed |
| DNS | Shared Service | Stable Per Pod |
| Storage | Shared/Optional | Dedicated Per Pod |
| Identity | No | Yes |
| Ordered Deployment | No | Yes |
| Ordered Deletion | No | Yes |
| Best For | Stateless Apps | Databases |

---

# Deployment vs StatefulSet vs DaemonSet

| Feature | Deployment | StatefulSet | DaemonSet |
|----------|------------|-------------|------------|
| Stateless Apps | Yes | No | Sometimes |
| Databases | No | Yes | No |
| One Pod Per Node | No | No | Yes |
| Stable Identity | No | Yes | No |
| PVC Support | Optional | Yes | Optional |

---

# Headless Service vs ClusterIP

| ClusterIP | Headless |
|-----------|----------|
| One Virtual IP | No Virtual IP |
| Load Balancing | No Load Balancing |
| Client Sees One Address | Client Sees Every Pod |
| Deployment | StatefulSet |

---

# PV vs PVC

| PVC | PV |
|-----|----|
| Request | Actual Storage |
| Created by User | Created by Admin or StorageClass |
| Consumed by Pod | Backed by Storage |

---

# StorageClass Flow

```
PVC

↓

StorageClass

↓

PV

↓

AWS EBS
```

---

# StatefulSet Flow

```
StatefulSet

↓

Pod

↓

PVC

↓

PV

↓

EBS
```

---

# Complete Application Flow

```
Browser

↓

React

↓

NodeJS

↓

MongoDB Primary

↓

PVC

↓

PV

↓

EBS

↓

Replica Set

↓

Secondary

↓

PVC

↓

PV

↓

EBS
```

---

# kubectl Commands Cheat Sheet

Create resources

```bash
kubectl apply -f file.yml
```

Delete resources

```bash
kubectl delete -f file.yml
```

View Pods

```bash
kubectl get pods
```

Wide Output

```bash
kubectl get pods -o wide
```

Describe Pod

```bash
kubectl describe pod POD_NAME
```

Logs

```bash
kubectl logs POD_NAME
```

Interactive Shell

```bash
kubectl exec -it POD_NAME -- sh
```

Mongo Shell

```bash
kubectl exec -it mongodb-0 -- mongosh
```

Restart Deployment

```bash
kubectl rollout restart deployment node-app
```

Restart StatefulSet

```bash
kubectl rollout restart statefulset mongodb
```

View Services

```bash
kubectl get svc
```

View PVC

```bash
kubectl get pvc
```

View PV

```bash
kubectl get pv
```

View StatefulSet

```bash
kubectl get sts
```

View Endpoints

```bash
kubectl get endpoints
```

View EndpointSlices

```bash
kubectl get endpointslices
```

---

# MongoDB Commands

Replica Status

```javascript
rs.status()
```

Replica Configuration

```javascript
rs.conf()
```

Current Primary

```javascript
db.hello()
```

Switch Database

```javascript
use codingwale
```

View Collections

```javascript
show collections
```

Insert Document

```javascript
db.students.insertOne({
name:"Vishal",
age:20
})
```

Read Data

```javascript
db.students.find().pretty()
```

Delete

```javascript
db.students.deleteMany({})
```

---

# Docker Commands

Build

```bash
docker build -t app:v1 .
```

Tag

```bash
docker tag app:v1 username/app:v1
```

Push

```bash
docker push username/app:v1
```

Images

```bash
docker images
```

Remove

```bash
docker rmi IMAGE_ID
```

---

# AWS Commands We Used

Open Worker Node Security Group

↓

Add

```
TCP

31557

30314
```

Without this,

NodePort is inaccessible from outside the cluster.

---

# Memory Tricks

## Deployment

Think

```
Replace Anything

No Identity
```

---

## StatefulSet

Think

```
Identity

Storage

DNS
```

---

## PVC

Think

```
Storage Request
```

---

## PV

Think

```
Actual Disk
```

---

## Headless Service

Think

```
Direct DNS

No Load Balancer
```

---

## Replica Set

Think

```
Database Replication

Not Kubernetes
```

---

# Common Beginner Mistakes

❌ Believing StatefulSet replicates data.

Correct:

Database replicates data.

---

❌ Believing PVC stores data.

Correct:

PV/EBS stores the data.

PVC is only a request.

---

❌ Believing Headless Service stores data.

Correct:

It only provides stable DNS.

---

❌ Connecting applications using Pod IP.

Correct:

Always use Service DNS.

---

❌ Forgetting Docker Build.

Always

```
Build

↓

Tag

↓

Push

↓

Restart Deployment
```

---

❌ Forgetting CORS.

React

↓

Browser

↓

NodeJS

Needs

```
cors
```

---

❌ Opening NodePort on Control Plane.

Correct

Worker Node Security Group.

---

# One Page Revision

Remember only these six points.

```
Deployment

↓

Stateless

------------------------

StatefulSet

↓

Stateful

------------------------

Headless Service

↓

Stable DNS

------------------------

PVC

↓

Storage Request

------------------------

PV

↓

Actual Storage

------------------------

Replica Set

↓

Data Replication
```

If you remember these six concepts, you already understand 90% of StatefulSet.

---

# Final Summary

Throughout this chapter, we built a complete production-style three-tier application on Kubernetes.

We learned how to:

- Differentiate between stateless and stateful workloads.
- Deploy MongoDB using StatefulSets.
- Use Headless Services for stable networking.
- Persist data using PVCs, PVs, and AWS EBS.
- Configure a MongoDB Replica Set.
- Deploy a NodeJS backend with Deployments.
- Connect the backend to MongoDB using Kubernetes DNS.
- Deploy a React frontend.
- Expose applications using NodePort Services.
- Resolve real-world issues such as DNS failures, CORS, stale Docker images, CrashLoopBackOff, and AWS Security Group configuration.
- Understand the responsibilities of Kubernetes versus the responsibilities of the database.

The most important lesson is this:

**Kubernetes provides the infrastructure (Pods, networking, storage, identity, recovery), while the application (such as MongoDB) is responsible for its own business logic (replication, leader election, transactions, consistency).**

Understanding this separation of responsibilities is the key to mastering StatefulSets and stateful workloads in Kubernetes.
