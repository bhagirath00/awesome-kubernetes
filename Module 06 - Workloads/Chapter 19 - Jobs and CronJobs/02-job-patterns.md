# Job Patterns

## Introduction

Jobs are not just for simple single-Pod tasks. Kubernetes supports four distinct Job patterns for different types of batch workloads — from simple one-shot migrations to massively parallel data processing pipelines. Choosing the right pattern directly impacts performance, cost, and reliability.

---

## Pattern 1: Single Completion (Default)

One Pod runs, completes, Job is done. This is what most people think of when they think of a Job:

```yaml
spec:
  completions: 1    # 1 successful Pod = Job done
  parallelism: 1    # run 1 Pod at a time
  backoffLimit: 3
```

**Use for:** Database migrations, one-time data transformations, report generation, any task that runs once.

---

## Pattern 2: Fixed Completion Count — Independent Work Items

Each Pod processes one work item independently. The Job runs N Pods (in parallel up to `parallelism`) until `completions` successful Pods have run:

```yaml
spec:
  completions: 10    # need 10 successful completions
  parallelism: 3     # run 3 Pods at a time
  backoffLimit: 5
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: data-processor:v1.0
        command: ["python", "process.py"]
        # Each Pod independently picks up work from a queue or uses its index
```

Timeline with `completions: 10, parallelism: 3`:
```
Time 0: Pods 1, 2, 3 start
Time 5: Pod 2 finishes → Pod 4 starts
Time 8: Pods 1, 3 finish → Pods 5, 6 start
...continues until 10 Pods have completed successfully
```

**Use for:** Processing N files, N database records, N S3 objects — where each Pod handles one item and items are self-contained.

---

## Pattern 3: Indexed Jobs — Each Pod Knows Its ID

Kubernetes 1.21+ supports indexed Jobs — each Pod gets a unique index (0 to completions-1) available as `JOB_COMPLETION_INDEX` environment variable. Each Pod processes the work for its specific index:

```yaml
spec:
  completions: 5
  parallelism: 5         # all 5 run in parallel
  completionMode: Indexed # each Pod gets a unique index
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: ml-trainer:v1.0
        command:
        - python
        - train.py
        - --shard-index=$(JOB_COMPLETION_INDEX)
        - --total-shards=5
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

Pod 0 trains on shard 0, Pod 1 on shard 1, etc. All 5 run simultaneously. When all complete successfully, the Job is done.

**Use for:** ML model training with data sharding, parallel map operations on partitioned datasets, distributed computation.

---

## Pattern 4: Work Queue — Pods Pull from a Queue

Each Pod pulls tasks from a queue (SQS, Redis, RabbitMQ). The Job runs until `parallelism` Pods all exit with code 0 (signaling the queue is empty):

```yaml
spec:
  parallelism: 5         # 5 workers processing from queue simultaneously
  completions: 1         # needs 1 successful completion to stop
  # Alternative: omit completions entirely → Job runs until all parallelism Pods exit 0
  backoffLimit: 10       # high backoff — queue workers are expected to fail occasionally
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: queue-worker:v1.0
        command:
        - python
        - worker.py
        - --queue-url=$(SQS_QUEUE_URL)
        env:
        - name: SQS_QUEUE_URL
          value: "https://sqs.us-east-1.amazonaws.com/123456789/orders-queue"
        - name: AWS_REGION
          value: "us-east-1"
```

The `worker.py` script:
```python
import boto3

sqs = boto3.client('sqs')
queue_url = os.environ['SQS_QUEUE_URL']

while True:
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=1,
        WaitTimeSeconds=20    # long poll — wait up to 20s for messages
    )
    
    messages = response.get('Messages', [])
    if not messages:
        print("Queue is empty — exiting with success")
        sys.exit(0)     # exit 0 → Job counts this Pod as a successful completion
    
    message = messages[0]
    try:
        process_message(message['Body'])
        sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=message['ReceiptHandle'])
    except Exception as e:
        print(f"Processing failed: {e}")
        sys.exit(1)     # exit 1 → Pod fails → Job retries up to backoffLimit
```

**Use for:** Order processing, image processing pipelines, report generation from SQS, async task processing.

---

## Indexed Job with S3 File Processing

A complete real-world example — process 100 S3 files, 10 at a time:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: process-reports
  namespace: production
spec:
  completions: 100       # 100 files to process
  parallelism: 10        # 10 workers at a time
  completionMode: Indexed
  backoffLimit: 3
  ttlSecondsAfterFinished: 3600
  activeDeadlineSeconds: 7200    # fail if not done in 2 hours
  
  template:
    spec:
      serviceAccountName: report-processor-sa   # IRSA for S3 access
      restartPolicy: Never
      containers:
      - name: processor
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/report-processor:v2.0
        command:
        - python
        - process_report.py
        env:
        - name: FILE_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
        - name: S3_BUCKET
          value: "my-company-reports"
        - name: AWS_REGION
          value: "us-east-1"
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

```python
# process_report.py
import boto3
import os

file_index = int(os.environ['FILE_INDEX'])
bucket = os.environ['S3_BUCKET']

s3 = boto3.client('s3')

# List all files and process the one at this index
response = s3.list_objects_v2(Bucket=bucket, Prefix='reports/')
files = sorted(response['Contents'], key=lambda x: x['Key'])
file_key = files[file_index]['Key']

print(f"Processing file {file_index}: {file_key}")
process_file(s3, bucket, file_key)
print(f"File {file_index} processed successfully")
```

---

## Monitoring Job Progress

```bash
# Overall Job status
kubectl get job process-reports -n production

# NAME              COMPLETIONS   DURATION   AGE
# process-reports   47/100        5m30s      5m30s    ← 47 of 100 done

# Count running/failed Pods
kubectl get pods -n production -l job-name=process-reports \
  -o jsonpath='{range .items[*]}{.status.phase}{"\n"}{end}' | sort | uniq -c
# 10 Running
# 47 Succeeded
# 3 Failed   ← 3 Pods failed (retried automatically)

# View logs for a failed Pod
kubectl get pods -n production -l job-name=process-reports \
  --field-selector='status.phase=Failed' \
  -o name | head -1 | xargs kubectl logs -n production

# Prometheus metric for Job progress:
# kube_job_status_succeeded{job_name="process-reports"} 47
# kube_job_status_failed{job_name="process-reports"} 3
```

---

## Summary

- **Single completion**: 1 Pod runs once — for migrations and one-time tasks
- **Fixed completion count**: N Pods run (up to `parallelism` at a time) until N successes — for independent batch items
- **Indexed Jobs** (`completionMode: Indexed`): each Pod gets a `JOB_COMPLETION_INDEX` — for sharded processing where each Pod handles a specific partition
- **Work queue**: `parallelism` workers pull from SQS/Redis — exit 0 when queue empty; good when total work count is unknown
- Set `ttlSecondsAfterFinished` to automatically clean up completed Jobs; set `activeDeadlineSeconds` as a circuit breaker for runaway Jobs

In the next file, `03-what-is-a-cronjob.md`, you will learn about CronJobs — the Kubernetes scheduler for running Jobs on a time-based schedule, equivalent to cron on Linux.
