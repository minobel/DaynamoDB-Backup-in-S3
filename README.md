```
# Serverless Data Pipeline: AWS Lambda, DynamoDB & Scheduled S3 Backups

An enterprise-grade serverless data pipeline built using Python (`boto3`, `urllib`) and AWS core services. This project ingests public REST API data, stores it in an indexed NoSQL database, and automates fault-tolerant, timestamped JSON backups to private object storage every 2 minutes via EventBridge Scheduler.

---

## 🏗️ Architecture & Detailed Data Flow

```text
[https://jsonplaceholder.typicode.com/users](https://jsonplaceholder.typicode.com/users)
                 │
                 ▼  (HTTP GET via urllib)
[Lambda #1: import-jsonplaceholder-users]
                 │
                 ▼  (boto3: dynamodb:PutItem)
   [DynamoDB Table: jsonplaceholder-users]
                 │
                 ▼  (boto3: dynamodb:Scan with Pagination)
  [Lambda #2: export-dynamodb-to-s3] <── [EventBridge Scheduler] (rate: 2 mins)
                 │
                 ▼  (boto3: s3:PutObject)
[Private S3 Bucket: backups/YYYY-MM-DD/users-*.json]
                 │
                 ▼
    [Amazon CloudWatch (Logs & Metrics)]

```

## 🛠️ Tech Stack & Service Configuration

**Service**

**Configuration / Specifications**

**Purpose**

**AWS Lambda**

Python 3.12, 128 MB RAM, 30s Timeout

Serverless compute engine for ETL operations

**Amazon DynamoDB**

On-Demand Capacity, Partition Key: `id` (Number)

Central NoSQL database store

**Amazon S3**

SSE-S3 Encryption, Block All Public Access

Private destination for timestamped JSON backups

**EventBridge Scheduler**

Rate Expression: `rate(2 minutes)`, Window: Off

Automated cron-like serverless trigger

**Amazon CloudWatch**

Custom Python logging & Lambda metrics

Observability, log aggregation, and debugging

**Boto3 SDK**

Standard Python runtime SDK

AWS API integration (DynamoDB & S3 operations)

## 📂 Repository File Structure

Plaintext

```
aws-serverless-dynamodb-backup-pipeline/
├── README.md                              <-- Complete project documentation
├── iam_policies/
│   ├── lambda-import-putitem-policy.json    <-- Least-privilege IAM policy for Lambda #1
│   └── lambda-backup-export-policy.json     <-- Least-privilege IAM policy for Lambda #2
├── lambda_src/
│   ├── import_users/
│   │   └── lambda_function.py            <-- Python source: API fetching & DynamoDB PutItem
│   └── backup_users/
│       └── lambda_function.py            <-- Python source: DynamoDB Scan & S3 PutObject
├── lambda_zips/
│   ├── lambda-import-users.zip            <-- Deployment ZIP for Lambda #1
│   └── lambda-backup-to-s3.zip            <-- Deployment ZIP for Lambda #2
└── screenshots/                           <-- Visual verification evidence
    ├── 01-dynamodb-table-active.png
    ├── 02-lambda-import-success.png
    ├── 03-s3-backup-files.png
    └── 04-eventbridge-schedule.png

```

## 🚀 Deep-Dive Step-by-Step Implementation

### PART A — Database & Object Storage Provisioning

1.  **AWS Region Standardisation:**
    
    -   Configured all resources within a single AWS Region (`us-west-2` / Oregon) to eliminate cross-region latency and billing complications.
        
2.  **DynamoDB Table Setup (`jsonplaceholder-users`):**
    
    -   Created table `jsonplaceholder-users` with Partition Key `id` set as **Number** (NOT String) to match API integer IDs.
        
    -   Configured read/write capacity mode to **On-Demand** to automatically handle spikes without manual scaling.
        
    -   🔗 _[View DynamoDB Table Setup Evidence](https://github.com/minobel/DaynamoDB-Backup-in-S3/blob/main/screenshots/01-dynamodb-table-active.png)_
        
3.  **Private S3 Destination Bucket:**
    
    -   Provisioned a globally unique bucket (`nobel-devops-backups-2026`).
        
    -   Explicitly enabled **Block all public access** and active **SSE-S3 (AES-256)** server-side encryption.
        
    -   S3 prefix `backups/` is managed dynamically during uploads without requiring physical folder creation.
        

### PART B — Lambda #1: REST API Ingestion to DynamoDB

1.  **Lambda #1 Creation (`import-jsonplaceholder-users`):**
    
    -   Runtime: Python 3.12 | Architecture:  `x86_64`.
        
    -   Execution Timeout: Adjusted from default 3s to **30 seconds** (128 MB RAM).
        
2.  **IAM Least-Privilege Execution Role:**
    
    -   Attached an inline policy (`AllowPutDemoUsers`) restricting permissions strictly to `dynamodb:PutItem` on `arn:aws:dynamodb:REGION:ACCOUNT_ID:table/jsonplaceholder-users`.
        
3.  **Environment Configuration & Deployment:**
    
    -   Set Environment Variable:  `TABLE_NAME = jsonplaceholder-users1`.
        
    -   Uploaded `lambda_zips/lambda-import-users.zip` containing `lambda_function.py`.
        
4.  **Invocation & Execution Flow:**
    
    -   Executed manual test with empty event `{}`.
        
    -   Lambda fetched 10 user objects from `https://jsonplaceholder.typicode.com/users` via Python `urllib.request`.
        
    -   Stored records in DynamoDB using `table.put_item(Item=user)`.
        
    -   🔗 _[View Execution Output Evidence](https://www.google.com/search?q=%2302-lambda-import-success)_
        

### PART C — Lambda #2: Data Extraction & S3 Backup

1.  **Lambda #2 Creation (`export-dynamodb-to-s3`):**
    
    -   Runtime: Python 3.12 | RAM: 128 MB | Timeout: 30 seconds.
        
2.  **IAM Security Policy:**
    
    -   Attached policy (`AllowDynamoDBJsonBackup`) scoped strictly to:
        
        -   `dynamodb:Scan` on table `jsonplaceholder-users`.
            
        -   `s3:PutObject` on resource `arn:aws:s3:::YOUR-BUCKET-NAME/backups/*`.
            
3.  **Environment Variables Configuration:**
    
    -   `TABLE_NAME` = `jsonplaceholder-users`
        
    -   `BUCKET_NAME` = `your-actual-s3-bucket-name`
        
    -   `BACKUP_PREFIX` = `backups`
        
4.  **Data Extraction & Transformation Logic:**
    
    -   Performs full-table scan using Boto3 with **pagination handling** (`LastEvaluatedKey`).
        
    -   Converts non-serializable DynamoDB `Decimal` types into standard JSON numbers.
        
    -   Writes array to S3 using timestamped UTC keys:  `backups/YYYY-MM-DD/users-HH-MM-SS-uuid.json`.
        

### PART D — Automated Trigger via EventBridge Scheduler

1.  **Cron Automation Setup:**
    
    -   Created a recurring schedule `dynamodb-backup-every-2min` in EventBridge Scheduler.
        
    -   Set Schedule Pattern: Rate-based -> `rate(2 minutes)` with Flexible Time Window set to **Off**.
        
2.  **Target & Delegation:**
    
    -   Targeted Lambda #2 (`export-dynamodb-to-s3`).
        
    -   Created a dedicated Scheduler Execution Role allowing EventBridge to invoke Lambda #2.
        
3.  **Verification:**
    
    -   Verified automated triggers creating distinct, timestamped JSON snapshots in S3 every 2 minutes.
        
    -   🔗 _[View EventBridge Schedule Evidence](https://www.google.com/search?q=%2304-eventbridge-schedule)_ | 🔗 _[View S3 Automated Files Evidence](https://www.google.com/search?q=%2303-s3-backup-files)_
        

## 🛠️ Common Errors, Root Cause & Troubleshooting Matrix

**Symptom / Error**

**Root Cause Analysis**

**Engineering Solution**

`KeyError: 'TABLE_NAME'`

Missing environment variable in Lambda configuration.

Added `TABLE_NAME=jsonplaceholder-users` in Lambda -> Configuration -> Environment variables.

`AccessDeniedException` on `PutItem`

IAM Role lacks permission to write to the specific DynamoDB table.

Updated IAM policy with explicit `dynamodb:PutItem` action matching the target table ARN.

`TypeError: Object of type Decimal is not JSON serializable`

Python standard `json.dumps()` cannot handle DynamoDB `Decimal` numeric responses natively.

Implemented custom Decimal encoder / cast values to standard Python `int`/`float` before serialization.

`ValidationException: Supplying values for numeric attribute...`

Partition key `id` was created as String instead of Number in DynamoDB.

Recreated DynamoDB table specifying `id` with type **Number**.

Lambda times out after 3.00s

Public API fetch taking longer than Lambda default timeout limit.

Extended Lambda Execution Timeout to 30 seconds under General Configuration.

## 📊 Verification & Screenshots

#### 1. DynamoDB Table Active & Populated

Shows 10 demo user items populated via API ingestion with partition key `id` (Number).

#### 2. Lambda #1 Ingestion Test

Successful manual invocation output (`saved_count: 10`) and CloudWatch execution log output.

#### 3. S3 Timestamped Backup Files

Multiple automated JSON backup exports generated every 2 minutes inside `backups/YYYY-MM-DD/`.

#### 4. EventBridge Scheduler Setup

Active EventBridge schedule configured at `rate(2 minutes)` targeting the export Lambda.

## 💡 Technical Takeaways & Production Considerations

1.  **Idempotency via PutItem:** Re-running Lambda #1 replaces existing items with matching partition keys (`id`) rather than creating duplicates.
    
2.  **Scan vs Point-in-Time Recovery (PITR):**
    
    -   _Lab Approach:_ Performs table scans export to S3. This is useful for data extraction, but can be expensive and inefficient for large tables.
        
    -   _Production Recommendation:_ Use **DynamoDB Point-in-Time Recovery (PITR)** and native AWS managed **Export to S3** via AWS Glue / DynamoDB native export to avoid consuming Read Capacity Units (RCUs).
        
3.  **IAM Role Separation:** Separate roles were used for Lambda compute execution vs EventBridge Scheduler execution, following the Principle of Least Privilege.
    

## 🧹 Infrastructure Cleanup

To prevent unnecessary AWS billable usage:

1.  Disabled and deleted schedule `dynamodb-backup-every-2min` in EventBridge.
    
2.  Deleted Lambda functions `import-jsonplaceholder-users` and `export-dynamodb-to-s3`.
    
3.  Emptied and deleted the destination S3 bucket.
    
4.  Deleted table `jsonplaceholder-users` and associated custom IAM roles.