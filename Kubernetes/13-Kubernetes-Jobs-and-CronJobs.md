Kubernetes Jobs and CronJobs

Introduction

Until now we have learned:

- Pod
- ReplicaSet
- Deployment
- DaemonSet
- Namespace
- Volumes
- Persistent Volume (PV)
- Persistent Volume Claim (PVC)
- ConfigMap
- Secret

Most of these Kubernetes objects are designed to keep applications running continuously.

Examples:

- Nginx
- Apache
- Jenkins
- Grafana
- Prometheus
- Banking Application
- E-Commerce Application

These applications should remain available all the time.

Kubernetes uses Deployments to ensure that these applications continue running even if Pods fail.

However, not every workload should run forever.

Some workloads only need to execute a task once and then stop.

Examples:

- Database Backup
- Database Migration
- Sending Emails
- Report Generation
- CSV Data Import
- Data Processing

For such workloads Kubernetes provides:

- Job
- CronJob

---

Why Jobs Are Needed

Imagine a company wants to take a database backup.

Process:

Database Backup
↓
Store Backup
↓
Exit

Once the backup is complete, the task should stop.

There is no need to keep the container running.

Another example:

Generate Monthly Report
↓
Create PDF
↓
Send Email
↓
Exit

Again, once completed the task should stop.

---

Why Deployment Is Not Suitable

Deployment is designed for long-running applications.

Example:

kind: Deployment

Deployment behavior:

Pod Running
↓
Pod Crashes
↓
Deployment Creates New Pod

Deployment always tries to maintain desired replicas.

Suppose we create a Deployment for backup:

Backup Starts
↓
Backup Completes
↓
Container Exits
↓
Deployment Creates New Pod
↓
Backup Runs Again

This continues forever.

This is not desired.

---

What is a Job?

Definition:

A Job is a Kubernetes object that runs a task until it completes successfully.

Job ensures:

- Task executes
- Pod is created
- Failures are retried
- Success is tracked

Once successful:

Job
↓
Completed

No new Pod is created.

---

Job Architecture

Job
 │
 └── Creates Pod
         │
         └── Executes Task
                 │
                 └── Completes

Relationship:

Job
 ↓
Pod
 ↓
Container

---

Job Lifecycle

Step 1

Create Job

kubectl create -f job.yml

Step 2

Kubernetes creates Pod

Job
↓
Pod Created

Step 3

Container executes task

Task Running

Step 4

Task finishes

Completed

Step 5

Job marked successful

1/1 Completions

---

Job Practical Lab

Objective

Create a Job that:

- Prints a message
- Waits 30 seconds
- Prints another message
- Exits successfully

---

Step 1 Create Directory

mkdir jobs
cd jobs

---

Step 2 Create YAML File

vim job1.yml

---

Step 3 Paste YAML

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

---

Step 4 Create Job

kubectl create -f job1.yml

Expected Output:

job.batch/first-job created

---

Step 5 Verify Job

kubectl get jobs

Output:

NAME        COMPLETIONS
first-job   0/1

---

Step 6 Verify Pod

kubectl get pods

Output:

first-job-xxxxx Running

---

Step 7 Wait 30 Seconds

Check again:

kubectl get jobs

Output:

NAME        COMPLETIONS
first-job   1/1

---

Step 8 Check Pod Status

kubectl get pods

Output:

first-job-xxxxx Completed

---

Step 9 Check Logs

kubectl logs <pod-name>

Example:

kubectl logs first-job-abcde

Output:

Job Started
Hello Kubernetes

---

Step 10 Describe Job

kubectl describe job first-job

Observe:

Succeeded: 1
Failed: 0

---

Step 11 Delete Job

kubectl delete job first-job

---

Understanding Every Line of Job YAML

apiVersion

apiVersion: batch/v1

Jobs belong to Batch API.

---

kind

kind: Job

Creates Job resource.

---

metadata

metadata:
  name: first-job

Name of Job.

---

template

template:

Defines Pod specification.

---

image

image: busybox

Container image.

---

command

command:

Commands executed inside container.

---

restartPolicy

restartPolicy: Never

Do not restart container after completion.

---

Important Job Parameters

completions

Defines how many successful completions are required.

Example:

completions: 5

Means:

Task must complete successfully 5 times.

---

parallelism

Defines how many Pods run simultaneously.

Example:

parallelism: 2

Means:

2 Pods run together.

---

backoffLimit

Defines retry count if Job fails.

Example:

backoffLimit: 4

Means:

Retry 4 times before marking Job failed.

---

What is a CronJob?

CronJob is a scheduler for Jobs.

Definition:

A CronJob automatically creates Jobs according to a schedule.

Think:

CronJob
   ↓
Creates Job
   ↓
Creates Pod
   ↓
Runs Task
   ↓
Completes

---

Why CronJob Is Needed

Suppose backup should run:

Every Day at 2 AM

Will you create Job manually every day?

No.

Need automation.

CronJob provides scheduling.

---

CronJob Architecture

CronJob
   │
   └── Creates Job
            │
            └── Creates Pod
                     │
                     └── Executes Task

Relationship:

CronJob
 ↓
Job
 ↓
Pod

---

Cron Schedule Explained

Format:

* * * * *
│ │ │ │ │
│ │ │ │ └── Day Of Week
│ │ │ └──── Month
│ │ └────── Day
│ └──────── Hour
└────────── Minute

---

Every Minute

"* * * * *"

---

Every Day At 2 AM

"0 2 * * *"

---

Every Day At 6 PM

"0 18 * * *"

---

Every Sunday

"0 0 * * 0"

---

Every 5 Minutes

"*/5 * * * *"

---

CronJob Practical Lab

Objective

Run a task every minute.

---

Step 1 Create File

vim cronjob.yml

---

Step 2 Paste YAML

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

---

Step 3 Create CronJob

kubectl create -f cronjob.yml

Output:

cronjob.batch/first-cronjob created

---

Step 4 Verify CronJob

kubectl get cj

Output:

NAME
first-cronjob

---

Step 5 Wait One Minute

CronJob automatically creates Job.

Check:

kubectl get jobs

Output:

first-cronjob-29703119

---

Step 6 Verify Pod

kubectl get pods

Output:

first-cronjob-29703119-xxxxx

---

Step 7 Check Logs

kubectl logs <pod-name>

Output:

CronJob Started
CronJob Completed

---

Step 8 Observe Again

Wait another minute.

Run:

kubectl get jobs

You will see a new Job.

Reason:

CronJob creates a new Job every minute.

---

Step 9 Delete CronJob

kubectl delete cronjob first-cronjob

or

kubectl delete cj first-cronjob

---

Troubleshooting

Job Not Starting

Check:

kubectl describe job job-name

---

Check Pods:

kubectl get pods

---

Check Events:

kubectl get events

---

Job Failed

Check Logs:

kubectl logs pod-name

---

Describe Job:

kubectl describe job job-name

---

CronJob Not Creating Jobs

Verify Schedule:

kubectl describe cj cronjob-name

---

Verify Jobs:

kubectl get jobs

---

Verify Events:

kubectl get events

---

Pod Stuck in Error

Check:

kubectl describe pod pod-name

---

Production Use Cases

Database Backup

Every night:

2 AM
↓
Take Backup
↓
Upload to S3
↓
Exit

CronJob

---

Database Migration

Old Schema
↓
Convert Data
↓
New Schema

Job

---

CSV Import

Read CSV
↓
Insert Data
↓
Exit

Job

---

Log Cleanup

Every midnight:

Delete Old Logs

CronJob

---

Generate Reports

Every morning:

Generate Reports
↓
Send Email

CronJob

---

Bulk Email Processing

Send 1 Lakh Emails
↓
Exit

Job

---

Best Practices

1. Use Job for one-time tasks.

2. Use CronJob for recurring tasks.

3. Use lightweight images like BusyBox or Alpine.

4. Monitor logs regularly.

5. Avoid running CronJobs every few seconds.

6. Delete old completed Jobs.

7. Use meaningful names.

8. Configure retries using backoffLimit.

9. Store credentials in Secrets.

10. Store configurations in ConfigMaps.

---

Interview Questions

What is a Job?

A Job runs a task until successful completion.

---

What is a CronJob?

A CronJob creates Jobs on a schedule.

---

Difference Between Job and CronJob?

Job:

Runs once.

CronJob:

Runs repeatedly according to schedule.

---

Can CronJob Create Pods Directly?

No.

Architecture:

CronJob
 ↓
Job
 ↓
Pod

---

Why Not Use Deployment for Backup Tasks?

Deployment continuously recreates Pods.

Backup tasks should run once and stop.

---

What is backoffLimit?

Maximum retry count for failed Jobs.

---

What is parallelism?

Number of Pods running simultaneously.

---

What is completions?

Number of successful task executions required.

---

Most Common Production Use Case of CronJob?

Database Backups.

---

Quick Revision

Deployment
 ↓
Pod

Long Running Applications

Examples:

- Nginx
- Jenkins
- Grafana

---

Job
 ↓
Pod

Run Once

Examples:

- Data Migration
- Backup Once
- CSV Import

---

CronJob
 ↓
Job
 ↓
Pod

Run On Schedule

Examples:

- Daily Backup
- Log Cleanup
- Report Generation

---

Golden Rule

Deployment = Run Forever

Job = Run Once

CronJob = Run On Schedule
