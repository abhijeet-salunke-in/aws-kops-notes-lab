## Kubernetes Jobs and CronJobs

## Introduction

Until now we have learned:
* Pod
* ReplicaSet
* Deployment
* DaemonSet
* Namespace
* Volumes
* Persistent Volume (PV)
* Persistent Volume Claim (PVC)
* ConfigMap
* Secret

Most of these Kubernetes objects are designed to keep applications running continuously.

**Examples:**
* Nginx
* Apache
* Jenkins
* Grafana
* Prometheus
* Banking Application
* E-Commerce Application

These applications should remain available all the time. Kubernetes uses **Deployments** to ensure that these applications continue running even if Pods fail.

However, not every workload should run forever. Some workloads only need to execute a task once and then stop.

**Examples:**
* Database Backup
* Database Migration
* Sending Emails
* Report Generation
* CSV Data Import
* Data Processing

For such workloads, Kubernetes provides:
1. **Job**
2. **CronJob**

---

## Why Jobs Are Needed

Imagine a company wants to take a database backup.

**Process workflow:**
`Database Backup` $\rightarrow$ `Store Backup` $\rightarrow$ `Exit`

Once the backup is complete, the task should stop. There is no need to keep the container running.

Another example:
`Generate Monthly Report` $\rightarrow$ `Create PDF` $\rightarrow$ `Send Email` $\rightarrow$ `Exit`

Again, once completed, the task should stop.

---

## Why Deployment Is Not Suitable

Deployment is designed for long-running applications (e.g., `kind: Deployment`).

**Deployment behavior:**
`Pod Running` $\rightarrow$ `Pod Crashes` $\rightarrow$ `Deployment Creates New Pod`

A Deployment always tries to maintain its desired number of replicas. Suppose we create a Deployment for a backup task:

`Backup Starts` $\rightarrow$ `Backup Completes` $\rightarrow$ `Container Exits` $\rightarrow$ `Deployment Creates New Pod` $\rightarrow$ `Backup Runs Again...`

This loop continues forever, which is not what we want.

---

## What is a Job?

> **Definition:** A Job is a Kubernetes object that runs a task until it completes successfully.

A Job ensures that:
* The task executes.
* The Pod is created.
* Failures are retried.
* Success is tracked.

Once successful:
`Job` $\rightarrow$ `Completed` (No new Pod is created).

---

## Job Architecture

```text
Job
└── Creates Pod
    └── Executes Task
        └── Completes

```
**Relationship:**
Job \rightarrow Pod \rightarrow Container
## Job Lifecycle
 * **Step 1:** Create Job (kubectl create -f job.yml)
 * **Step 2:** Kubernetes creates Pod (Job \rightarrow Pod Created)
 * **Step 3:** Container executes task (Task Running)
 * **Step 4:** Task finishes (Completed)
 * **Step 5:** Job marked successful (1/1 Completions)
## Job Practical Lab
### Objective
Create a Job that prints a message, waits 30 seconds, prints another message, and exits successfully.
### Step 1: Create Directory
```bash
mkdir jobs
cd jobs

```
### Step 2: Create YAML File
```bash
vim job1.yml

```
### Step 3: Paste YAML
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: first-job
spec:
  template:
    spec:
      containers:
      - name: cont1
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Job Started"
          sleep 30
          echo "Hello Kubernetes"
      restartPolicy: Never

```
### Step 4: Create Job
```bash
kubectl create -f job1.yml

```
**Expected Output:**
```text
job.batch/first-job created

```
### Step 5: Verify Job
```bash
kubectl get jobs

```
**Output:**
```text
NAME        COMPLETIONS   DURATION   AGE
first-job   0/1           4s         4s

```
### Step 6: Verify Pod
```bash
kubectl get pods

```
**Output:**
```text
NAME              READY   STATUS    RESTARTS   AGE
first-job-xxxxx   1/1     Running   0          6s

```
### Step 7: Wait 30 Seconds and Check Again
```bash
kubectl get jobs

```
**Output:**
```text
NAME        COMPLETIONS   DURATION   AGE
first-job   1/1           32s        35s

```
### Step 8: Check Pod Status
```bash
kubectl get pods

```
**Output:**
```text
NAME              READY   STATUS      RESTARTS   AGE
first-job-xxxxx   0/1     Completed   0          40s

```
### Step 9: Check Logs
```bash
kubectl logs <pod-name>
# Example:
kubectl logs first-job-abcde

```
**Output:**
```text
Job Started
Hello Kubernetes

```
### Step 10: Describe Job
```bash
kubectl describe job first-job

```
Observe fields like:
```text
Succeeded: 1
Failed: 0

```
### Step 11: Delete Job
```bash
kubectl delete job first-job

```
## Understanding Every Line of Job YAML
 * **apiVersion: batch/v1**: Jobs belong to the Batch API group.
 * **kind: Job**: Specifies that this resource is a Job.
 * **metadata / name: first-job**: The unique name of the Job.
 * **template:**: Defines the Pod specification template that the Job controller will use to create Pods.
 * **image: busybox**: The lightweight container image used to execute the work.
 * **command:**: The exact script or command executed inside the container.
 * **restartPolicy: Never**: Instructs Kubernetes not to restart the container after it successfully exits.
## Important Job Parameters
### completions
Defines how many successful Pod completions are required before the Job is considered done.
```yaml
spec:
  completions: 5

```
*Means:* The task must complete successfully 5 times.
### parallelism
Defines how many Pods can run concurrently at any given time.
```yaml
spec:
  parallelism: 2

```
*Means:* Up to 2 Pods will run together in parallel.
### backoffLimit
Defines the retry count limit before marking this Job as completely failed.
```yaml
spec:
  backoffLimit: 4

```
*Means:* Kubernetes will attempt to retry a failing Pod 4 times before giving up.
## What is a CronJob?
> **Definition:** A CronJob is a controller that manages time-based Jobs. It automatically runs Jobs according to a specified cron schedule.
> 
```text
CronJob
└── Creates Job
    └── Creates Pod
        └── Runs Task
            └── Completes

```
## Why CronJob Is Needed
Suppose a database backup task needs to run **every day at 2 AM**. Creating a manual Job definition file and deploying it every single night is highly inefficient.
CronJobs automate this scheduling logic entirely.
## CronJob Architecture
```text
CronJob
└── Creates Job
    └── Creates Pod

```
**Relationship:**
CronJob \rightarrow Job \rightarrow Pod
## Cron Schedule Explained
```text
* * * * *
│  │  │  │  │
│  │  │  │  └─── Day of Week (0 - 6) (Sunday to Saturday)
│  │  │  └────── Month (1 - 12)
│  │  └───────── Day of Month (1 - 31)
│  └──────────── Hour (0 - 23)
└─────────────── Minute (0 - 55)

```
| Schedule Expression | Meaning |
|---|---|
| * * * * * | Every single minute |
| 0 2 * * * | Every day at 2:00 AM |
| 0 18 * * * | Every day at 6:00 PM |
| 0 0 * * 0 | Every Sunday at midnight |
| */5 * * * * | Every 5 minutes |
## CronJob Practical Lab
### Objective
Run a basic container task on a schedule of every single minute.
### Step 1: Create File
```bash
vim cronjob.yml

```
### Step 2: Paste YAML
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: first-cronjob
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cont1
            image: busybox
            command:
            - sh
            - -c
            - |
              echo "CronJob Started"
              sleep 20
              echo "CronJob Completed"
          restartPolicy: Never

```
### Step 3: Create CronJob
```bash
kubectl create -f cronjob.yml

```
**Output:**
```text
cronjob.batch/first-cronjob created

```
### Step 4: Verify CronJob
```bash
kubectl get cj

```
**Output:**
```text
NAME            SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
first-cronjob   * * * * * False     0        <none>          12s

```
### Step 5: Wait One Minute
The CronJob controller automatically instantiates a new Job when the schedule matches.
```bash
kubectl get jobs

```
**Output:**
```text
NAME                     COMPLETIONS   DURATION   AGE
first-cronjob-29703119   0/1           5s         5s

```
### Step 6: Verify Pod
```bash
kubectl get pods

```
**Output:**
```text
NAME                           READY   STATUS    RESTARTS   AGE
first-cronjob-29703119-xxxxx   1/1     Running   0          10s

```
### Step 7: Check Logs
```bash
kubectl logs <pod-name>

```
**Output:**
```text
CronJob Started
CronJob Completed

```
### Step 8: Observe Again
Wait another minute and check your resources.
```bash
kubectl get jobs

```
You will notice a new Job item listed because the schedule triggered again.
### Step 9: Delete CronJob
```bash
kubectl delete cronjob first-cronjob
# OR
kubectl delete cj first-cronjob

```
## Troubleshooting
### Job Not Starting
 * Inspect the resource: kubectl describe job job-name
 * Look at the cluster Pods: kubectl get pods
 * Stream historical occurrences: kubectl get events --sort-by='.metadata.creationTimestamp'
### Job Failed
 * Tail container stdout logs: kubectl logs pod-name
 * Analyze execution limits or conditions: kubectl describe job job-name
### CronJob Not Creating Jobs
 * Ensure validation patterns match: kubectl describe cj cronjob-name
 * Verify whether SUSPEND is set to true.
 * Review controller orchestration logs: kubectl get events
### Pod Stuck in Error
 * Check configuration issues, OOM kills, or faulty entrypoints: kubectl describe pod pod-name
## Production Use Cases
 * **Database Backup:** Every night at 2 AM \rightarrow Take Backup \rightarrow Upload to AWS S3 \rightarrow Exit. *(CronJob)*
 * **Database Migration:** Apply old schema alterations \rightarrow Convert raw table structure \rightarrow New Schema. *(Job)*
 * **CSV Import:** Safely pull a specific file data payload \rightarrow Stream records into database tables \rightarrow Done. *(Job)*
 * **Log Cleanup:** Every midnight \rightarrow Purge local state indices or old temporary files. *(CronJob)*
 * **Generate Reports:** Every morning \rightarrow Assemble business graphs \rightarrow Dispatch emails. *(CronJob)*
 * **Bulk Email Processing:** Push a queue payload of 100,000 pending marketing message hooks \rightarrow Exit. *(Job)*
## Best Practices
 * Use **Jobs** for one-time, run-to-completion tasks.
 * Use **CronJobs** for recurring, period-based tasks.
 * Use lightweight target container base layers such as busybox or alpine to maintain low startup overhead.
 * Monitor resource logs regularly.
 * Avoid overloading scheduling frequencies (e.g., executing CronJobs every couple of seconds).
 * Clean up historical completed Jobs using .spec.ttlSecondsAfterFinished configuration property templates.
 * Use explicit, descriptive naming conventions.
 * Provide an engineered threshold for failure retries utilizing the backoffLimit.
 * Never hardcode database credentials; load them directly from **Secrets** or **ConfigMaps**.
## Interview Questions
### 1. What is a Job?
A Kubernetes Job is a workload abstraction entity that runs a task until a specific target count of successful completions is achieved.
### 2. What is a CronJob?
A CronJob is an object that schedules and runs Kubernetes Job tasks repeatedly over time based on an input crontab format.
### 3. Difference Between Job and CronJob?
 * **Job:** Runs once until successful completion, then stops.
 * **CronJob:** Automatically runs tasks repeatedly at regular, specific scheduled intervals.
### 4. Can CronJob Create Pods Directly?
No. Architecturally, a CronJob generates a **Job Object**, which then goes on to spin up standard **Pods** to perform execution.

### 5. Why Not Use Deployment for Backup Tasks?
Deployments aim to preserve an unceasingly live container state pattern. If a backup task finishes and shuts down cleanly, a Deployment treats it as an outage and spins up another redundant duplicate instance, creating an endless loop.
### 6. What is backoffLimit?
The maximum amount of failure retry iterations allowed before marking the complete Job deployment context as explicitly failed.
### 7. What is parallelism?
The deliberate ceiling configurations dictating how many container execution Pod contexts run at the exact same time.
### 8. What is completions?
The configured total number of successful termination iterations needed for a Job to transition into a finished state.
### 9. Most Common Production Use Case of CronJob?
Automating recurring database management backups and structural cleanup storage processes.
## Quick Revision
### Deployment \rightarrow Pod
 * **Pattern:** Long-Running Applications
 * **Examples:** Nginx, Jenkins, Grafana, Microservices
### Job \rightarrow Pod
 * **Pattern:** Run Once and Finish
 * **Examples:** Data Migrations, Single Database Backups, One-off Data Processing
### CronJob \rightarrow Job \rightarrow Pod
 * **Pattern:** Run Automatically on a Schedule
 * **Examples:** Daily Database Backups, Nightly Log Cleanups, Scheduled System Reports
> ### 💡 Golden Rule
>  * **Deployment** = Run Forever
>  * **Job** = Run Once
>  * **CronJob** = Run On Schedule
> 
```

```
