# Kubernetes Jobs and CronJobs

# 1. Introduction

Most Kubernetes workloads run continuously.

Examples:

- Nginx
- Jenkins
- Grafana
- Prometheus
- Applications
- APIs

These applications should remain available all the time.

For such workloads Kubernetes provides:

- Deployment
- ReplicaSet
- StatefulSet
- DaemonSet

However, not every task needs to run forever.

Some tasks only need to run once and then stop.

Examples:

- Database Backup
- Data Migration
- Report Generation
- Batch Processing
- Sending Emails
- Data Import

For such workloads Kubernetes provides:

- Job
- CronJob

---

# 2. Why Jobs Are Needed

Suppose we need to take a database backup.

Process:

Database Backup
↓
Store Backup
↓
Exit

The task is completed.

There is no reason to keep the container running.

If Kubernetes keeps restarting it, the backup will run again and again.

This is where Jobs help.

A Job executes a task and stops when the task succeeds.

---

# 3. Problem with Deployments for One-Time Tasks

Deployment is designed for long-running applications.

Example:

```yaml
kind: Deployment
```

Deployment behavior:

```text
Pod Starts
↓
Application Runs
↓
Pod Crashes
↓
Deployment Creates New Pod
```

Deployment always tries to maintain desired replicas.

This is useful for:

- Web Servers
- APIs
- Monitoring Tools

But not useful for:

- Backup Tasks
- Data Migration
- Report Generation

---

# 4. What is a Job?

Definition:

A Job is a Kubernetes resource that runs a task until it completes successfully.

Job ensures:

- Task runs successfully
- Pod is created
- Failures are retried
- Completion status is tracked

Once successful:

```text
Job
↓
Completed
```

No new Pod is created.

---

# 5. Job Architecture

```text
Job
 │
 └── Creates Pod
         │
         └── Executes Task
                 │
                 └── Completes
```

Relationship:

```text
Job
 ↓
Pod
 ↓
Container
```

---

# 6. Job Lifecycle

Step 1

Create Job

```bash
kubectl create -f job.yml
```

Step 2

Kubernetes creates Pod

```text
Job
↓
Pod Created
```

Step 3

Container executes task

```text
Task Running
```

Step 4

Task finishes

```text
Completed
```

Step 5

Job marked successful

```text
1/1 Completions
```

---

# 7. Job Practical Example

Create file:

```bash
vim job1.yml
```

Content:

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

Apply:

```bash
kubectl create -f job1.yml
```

---

# Verify Job

Check Job

```bash
kubectl get jobs
```

Example:

```text
NAME        COMPLETIONS
first-job   0/1
```

After 30 seconds:

```text
NAME        COMPLETIONS
first-job   1/1
```

---

Check Pod

```bash
kubectl get pods
```

Output:

```text
first-job-xxxxx   Completed
```

---

Check Logs

```bash
kubectl logs pod-name
```

Output:

```text
Job Started
Hello Kubernetes
```

---

# 8. Job YAML Explanation

## apiVersion

```yaml
apiVersion: batch/v1
```

Jobs belong to Batch API.

---

## kind

```yaml
kind: Job
```

Creates Job resource.

---

## metadata

```yaml
metadata:
  name: first-job
```

Job name.

---

## template

```yaml
template:
```

Defines Pod specification.

---

## image

```yaml
image: busybox
```

Container image.

---

## command

```yaml
command:
```

Commands executed inside container.

---

## restartPolicy

```yaml
restartPolicy: Never
```

Container will not restart after completion.

---

# 9. Useful Job Commands

Create Job

```bash
kubectl create -f job.yml
```

List Jobs

```bash
kubectl get jobs
```

Describe Job

```bash
kubectl describe job first-job
```

Check Pods

```bash
kubectl get pods
```

View Logs

```bash
kubectl logs pod-name
```

Delete Job

```bash
kubectl delete job first-job
```

---

# 10. Real-World Use Cases of Jobs

## Database Migration

```text
Old Schema
↓
Convert Data
↓
New Schema
```

---

## Database Backup

```text
Take Backup
↓
Store Backup
↓
Exit
```

---

## Email Processing

```text
Send Emails
↓
Exit
```

---

## CSV Import

```text
Read CSV
↓
Insert Data
↓
Exit
```

---

# 11. What is a CronJob?

CronJob is a scheduler for Jobs.

Definition:

A CronJob automatically creates Jobs according to a schedule.

---

# 12. Why CronJob is Needed

Suppose backup should run daily at 2 AM.

Using Job:

Someone must manually create Job every day.

Not practical.

Need automation.

Solution:

CronJob

---

# 13. CronJob Architecture

```text
CronJob
   │
   └── Creates Job
            │
            └── Creates Pod
                     │
                     └── Executes Task
```

Relationship:

```text
CronJob
 ↓
Job
 ↓
Pod
```

---

# 14. Cron Schedule Format

Format:

```text
* * * * *
│ │ │ │ │
│ │ │ │ └── Day of Week
│ │ │ └──── Month
│ │ └────── Day
│ └──────── Hour
└────────── Minute
```

---

Examples

Every Minute

```yaml
"* * * * *"
```

---

Every Day 2 AM

```yaml
"0 2 * * *"
```

---

Every Day 6 PM

```yaml
"0 18 * * *"
```

---

Every Sunday

```yaml
"0 0 * * 0"
```

---

# 15. CronJob Practical Example

Create file:

```bash
vim cronjob.yml
```

Content:

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

Apply:

```bash
kubectl create -f cronjob.yml
```

---

# 16. Verify CronJob

List CronJobs

```bash
kubectl get cronjobs
```

or

```bash
kubectl get cj
```

---

Check Jobs

```bash
kubectl get jobs
```

You will see new Jobs being created automatically.

---

Check Pods

```bash
kubectl get pods
```

---

Check Logs

```bash
kubectl logs pod-name
```

---

# 17. Useful CronJob Commands

Create

```bash
kubectl create -f cronjob.yml
```

List

```bash
kubectl get cj
```

Describe

```bash
kubectl describe cj first-cronjob
```

Delete

```bash
kubectl delete cj first-cronjob
```

Suspend

```bash
kubectl patch cronjob first-cronjob \
-p '{"spec":{"suspend":true}}'
```

Resume

```bash
kubectl patch cronjob first-cronjob \
-p '{"spec":{"suspend":false}}'
```

---

# 18. Real-World Use Cases of CronJobs

## Nightly Database Backup

```text
2 AM Daily
```

---

## Cleanup Old Logs

```text
12 AM Daily
```

---

## Generate Reports

```text
Every Morning
```

---

## Send Reminder Emails

```text
Every Hour
```

---

# 19. Job vs CronJob

| Feature | Job | CronJob |
|----------|------|----------|
| Execution | Once | Repeated |
| Scheduler | No | Yes |
| Creates Pod | Yes | Yes |
| Creates Job | No | Yes |
| Use Case | Backup Once | Daily Backup |

---

# 20. Deployment vs Job vs CronJob

| Feature | Deployment | Job | CronJob |
|-----------|------------|------|----------|
| Long Running | Yes | No | No |
| Scheduled | No | No | Yes |
| Auto Restart | Yes | No | No |
| Use Case | Applications | One-Time Task | Recurring Task |

---

# 21. Troubleshooting

## Job Not Starting

Check:

```bash
kubectl describe job job-name
```

---

Check Pod

```bash
kubectl get pods
```

---

View Logs

```bash
kubectl logs pod-name
```

---

## CronJob Not Running

Check Schedule

```bash
kubectl describe cj cronjob-name
```

---

Check Created Jobs

```bash
kubectl get jobs
```

---

Check Controller Events

```bash
kubectl get events
```

---

# 22. Best Practices

1. Use Jobs for one-time tasks.
2. Use CronJobs for recurring tasks.
3. Use lightweight images.
4. Monitor logs regularly.
5. Clean up completed Jobs.
6. Avoid excessive schedules.
7. Test Cron expressions carefully.

---

# 23. Interview Questions

### What is a Job?

A Job runs a task until successful completion.

---

### What is a CronJob?

A CronJob creates Jobs on a schedule.

---

### Difference between Job and CronJob?

Job runs once.

CronJob runs repeatedly according to a schedule.

---

### Can a CronJob create Pods directly?

No.

CronJob → Job → Pod

---

### Why not use Deployment for backup tasks?

Deployment continuously recreates Pods.

Backup tasks should run once and stop.

---

# Quick Revision

```text
Deployment
 ↓
Pod

Job
 ↓
Pod

CronJob
 ↓
Job
 ↓
Pod
```

Remember:

Deployment = Run Forever

Job = Run Once

CronJob = Run On Schedule
