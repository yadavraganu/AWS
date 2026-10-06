## Table of Contents

1. [What is AWS Lambda?](#1-what-is-aws-lambda)
2. [The Execution Environment](#2-the-execution-environment)
3. [Cold Start vs. Warm Start](#3-cold-start-vs-warm-start)
4. [Invocation Types](#4-invocation-types)
5. [Triggers & Event Sources](#5-triggers--event-sources)
6. [Event Source Mappings (Poll-Based Sources)](#6-event-source-mappings-poll-based-sources)
7. [Concurrency](#7-concurrency)
8. [Throttling](#8-throttling)
9. [Layers](#9-layers)
10. [Lambda with VPC](#10-lambda-with-vpc)
11. [Environment Variables & Configuration](#11-environment-variables--configuration)
12. [Versioning & Aliases](#12-versioning--aliases)
13. [IAM Execution Role](#13-iam-execution-role)
14. [Error Handling, Retries, and Destinations](#14-error-handling-retries-and-destinations)
15. [Packaging: Zip Archives vs Container Images](#15-packaging-zip-archives-vs-container-images)
16. [Ephemeral Storage (/tmp) & Idempotency](#16-ephemeral-storage-tmp--idempotency)
17. [Function URLs](#17-function-urls)
18. [Lambda Extensions](#18-lambda-extensions)
19. [Monitoring & Observability](#19-monitoring--observability)
20. [Lambda Limits & Quotas](#20-lambda-limits--quotas)
21. [Pricing](#21-pricing)
22. [Lambda@Edge vs CloudFront Functions](#22-lambdaedge-vs-cloudfront-functions)
23. [Lambda Best Practices](#23-lambda-best-practices)
24. [Common Interview Questions](#24-common-interview-questions)

---

## 1. What is AWS Lambda?

**AWS Lambda** is a serverless, event-driven compute service that runs your code in response to triggers, automatically managing the underlying compute infrastructure — no servers to provision or patch.

- You provide code (as a zip archive or container image) plus a **handler function**; Lambda runs it on demand and scales automatically from zero to thousands of concurrent invocations.
- You pay only for compute time **consumed while your code runs**, billed per millisecond (rounded in some older pricing models, now effectively per-ms), plus the number of requests.
- Supports many runtimes natively (Node.js, Python, Java, .NET, Go, Ruby) or any language via a **custom runtime** / container image.

---

## 2. The Execution Environment

An AWS Lambda execution environment is a secure, isolated micro-virtual machine powered by **AWS Firecracker** that manages the resources required to run your function.

The lifecycle consists of three distinct phases managed completely by AWS:

1. **INIT**: Lambda downloads the code, forks a microVM, boots the runtime, and runs static setup code.
2. **INVOKE**: Lambda executes your function's handler method to process the incoming event data.
3. **SHUTDOWN**: If no new traffic arrives, the runtime and any attached extensions are safely terminated.

---

## 3. Cold Start vs. Warm Start

The concept of cold and warm starts dictating serverless performance is directly tied to the execution environment's lifecycle.

| Metric / Step | Cold Start | Warm Start |
|---|---|---|
| Trigger Mechanism | First invocation or a sudden spike in scale. | High frequency, sequential back-to-back requests. |
| Download Code | Yes (S3 or ECR container image). | No (already in memory). |
| Boot Runtime | Yes (spins up Python, Node.js, JVM, etc.). | No (already running). |
| Static / Init Code | Yes (runs code outside the main handler). | No (reuses the initialized memory state). |
| Handler Execution | Yes (runs code inside the handler). | Yes (runs code inside the handler). |
| Latency Impact | High (+100 ms to >1 second). | Minimal (low single-digit milliseconds). |

### What is a Cold Start?
A cold start happens when a function is invoked but there is no idle execution environment available. AWS must spin up a container completely from scratch. This typically occurs on the very first request, during function updates, or when scaling out horizontally to support simultaneous concurrent requests.

### What is a Warm Start?
After processing an invocation, AWS freezes the execution environment and retains it for a non-deterministic period (typically 5 to 45 minutes). If a subsequent request arrives while the container is frozen, Lambda thaws and reuses it. This completely skips the code download, runtime booting, and static initialization phases.

### Key Execution Behavior to Remember
- **One Container, One Request**: a single execution environment will never process multiple concurrent requests at the exact same millisecond (outside of explicit response streaming/async worker patterns within your own code). Concurrency scales by adding completely new containers, which causes simultaneous cold starts.
- **State Preservation**: variables declared globally outside your handler function persist during a warm start. The temporary `/tmp` local directory also retains files between warm invocations (see [Section 16](#16-ephemeral-storage-tmp--idempotency)).
- **Billing Adjustments**: AWS standardizes billing structures to charge for the time spent inside the INIT phase. Keeping initializations highly optimized directly limits your runtime costs.

### Best Practices to Mitigate Cold Starts
- **Initialize Globally**: establish heavy elements like SDK clients, configurations, and database connections outside of the handler function. This isolates the performance penalty to the cold start phase only.
- **Trim Package Sizes**: only bundle the raw dependencies required for execution. Smaller zip or container payloads drastically reduce the time AWS spends downloading data from Amazon S3/ECR.
- **Utilize Provisioned Concurrency**: if your workload is highly latency-sensitive, you can pay AWS to keep a dedicated pool of environments permanently pre-warmed and ready to respond instantly.
- **Enable Lambda SnapStart**: for runtimes like Java, Python, or .NET, turn on AWS Lambda SnapStart. It caches a snapshot of the initialized microVM tier, improving startup performance up to 10x for free.

---

## 4. Invocation Types

Lambda supports three invocation models, and which one applies depends entirely on the **trigger**, not something you freely choose per call.

| Type | How It Works | Example Triggers | Error Handling |
|---|---|---|---|
| **Synchronous** | Caller waits for the function to run and return a response. | API Gateway, Application Load Balancer, Function URLs, direct SDK `Invoke` call | Errors are returned directly to the caller; caller is responsible for retries. |
| **Asynchronous** | Lambda queues the event internally and returns immediately; the function runs separately. | S3 events, SNS, EventBridge, CloudWatch Logs | Lambda automatically retries **twice** by default on failure, then can route to a **Dead Letter Queue (DLQ)** or **on-failure destination**. |
| **Poll-based (Event Source Mapping)** | Lambda's own internal poller reads from a stream/queue and invokes the function with a batch of records. | SQS, Kinesis Data Streams, DynamoDB Streams, Amazon MQ, Kafka | Managed by the event source mapping; retry behavior, batch size, and failure handling are configured on the mapping itself (see [Section 6](#6-event-source-mappings-poll-based-sources)). |

> **Interview tip:** The invocation type is a property of the **trigger**, not a setting on the function. "What happens if my function errors?" is really asking "what invocation type is this?" — synchronous callers must handle retries themselves; asynchronous and poll-based invocations have Lambda-managed retry behavior.

---

## 5. Triggers & Event Sources

A **trigger** is the event source that invokes a Lambda function. Common triggers and their invocation type:

| Service | Typical Use | Invocation Type |
|---|---|---|
| **API Gateway** | REST/HTTP APIs calling Lambda as a backend | Synchronous |
| **Application Load Balancer** | Lambda as a target behind an ALB | Synchronous |
| **Function URLs** | Direct HTTPS endpoint for a function, no API Gateway needed | Synchronous |
| **S3 Events** | Object created/removed triggers processing (e.g., thumbnailing) | Asynchronous |
| **SNS** | Pub/sub fan-out to one or more Lambda subscribers | Asynchronous |
| **EventBridge** | Scheduled (cron) jobs, or rule-based event routing | Asynchronous |
| **SQS** | Queue-based processing, batched | Poll-based (event source mapping) |
| **Kinesis Data Streams / DynamoDB Streams** | Stream processing, ordered within a shard/partition | Poll-based (event source mapping) |
| **Step Functions** | Orchestrated, multi-step workflows calling Lambda as a task | Synchronous (per task, within the state machine) |

---

## 6. Event Source Mappings (Poll-Based Sources)

For stream- and queue-based sources (SQS, Kinesis, DynamoDB Streams, Amazon MQ, self-managed Kafka), Lambda does not react to a push notification — instead, the Lambda service itself **polls** the source continuously using an **Event Source Mapping (ESM)**, then invokes your function synchronously with a batch of records on your behalf.

### Key Configuration
- **Batch size**: the max number of records delivered per invocation.
- **Batch window**: max time to wait for a batch to fill before invoking anyway.
- **Parallelization factor** (Kinesis/DynamoDB Streams): process multiple batches from the same shard concurrently (up to 10), useful when per-record processing is slow.
- **Maximum retry attempts / record age**: controls how long and how many times a failed record batch is retried before being discarded or sent to a failure destination.
- **Bisect on function error** (streams): splits a failing batch in half and retries each half separately, to isolate the specific bad record instead of blocking the whole batch.

### SQS vs Streams (Kinesis/DynamoDB) Behavior

| Aspect | SQS | Kinesis / DynamoDB Streams |
|---|---|---|
| Ordering | Best-effort (FIFO queues guarantee order within a message group) | Strict order **within a shard/partition key** |
| Failed record handling | Failed messages become visible again after the visibility timeout; can route to a queue-level DLQ | A persistently failing record **blocks the shard** until it succeeds, expires, or is skipped — this is the classic "poison pill" problem |
| Scaling unit | Lambda scales pollers based on queue depth, up to function concurrency limits | Concurrency is capped by the **number of shards/partitions** — one invocation processes one shard's batch at a time by default |
| IteratorAge metric | N/A | Tracks how far behind real-time your consumer is — a rising `IteratorAge` signals your function can't keep up |

> **Interview tip:** The classic Kinesis/DynamoDB Streams "poison pill" question: a malformed record that always throws an error will **block the entire shard** from progressing until you configure a **maximum retry count**, **bisect-on-error**, or a **failure destination** to skip past it.

---

## 7. Concurrency

In AWS Lambda, **concurrency** is the number of in-flight requests that your function is executing at any given single moment. Every time your function is invoked, Lambda spins up a separate instance (execution environment) to handle that request.

AWS provides two types of concurrency configurations to manage scaling, prevent system crashes, and optimize latency: **Reserved Concurrency** and **Provisioned Concurrency**.

| Feature | Standard Concurrency | Reserved Concurrency | Provisioned Concurrency |
|---|---|---|---|
| Primary Purpose | Default auto-scaling | Cap scaling & guarantee capacity | Eliminate cold starts |
| Max Cap Limit | Account limit (default: 1,000) | Custom maximum cap | Up to the reserved amount |
| Guaranteed Pool | None (shared pool) | Yes (exclusive to that function) | Yes (warmed execution environments) |
| Cold Starts | Yes (on scale-up) | Yes (on scale-up) | No (pre-warmed up to the limit) |
| Additional Cost | Free | Free | Paid (billed per second allocated) |

### 1. Standard (Unreserved) Concurrency
By default, all Lambda functions in a single AWS Region share a single regional pool of 1,000 concurrent executions.

- **How it works**: if Function A suddenly handles 950 concurrent requests, all other functions in your account are left with only 50 available slots.
- **The Risk**: a single traffic spike or a rogue recursive loop in one minor function can exhaust your entire account quota, causing a denial of service (throttling) across your critical applications.

### 2. Reserved Concurrency
Reserved Concurrency allows you to dedicate a specific portion of your regional pool exclusively to a single function. It acts as both a **floor** (guaranteed minimum) and a **ceiling** (strict limit).

- **Guarantees Capacity**: it saves a slice of your account pool for that function. No other function can steal this capacity, even if your account pool is exhausted.
- **Limits Scale**: the function cannot scale past this number. If you reserve 50 units, any incoming 51st simultaneous request will be throttled.
- **Use Cases**:
  - Protecting downstream systems like databases (e.g., stopping Lambda from overwhelming a MySQL DB that only allows 20 connections).
  - Ensuring a highly critical API function always has available slots.

### 3. Provisioned Concurrency
Provisioned Concurrency initializes a requested number of execution environments ahead of time so they are always warm and ready to respond instantly.

- **Eliminates Cold Starts**: standard Lambda functions experience a lag ("cold start") when spinning up new environments. Provisioned concurrency completely removes this initialization latency.
- **Scope**: you must apply this to a specific **published Function Version or Alias** (it cannot be applied to `$LATEST`).
- **Handling Spikes**: if your provisioned concurrency is set to 100, the first 100 concurrent requests have zero cold start lag. If you hit 101, Lambda will still process the extra request using normal, standard scaling (which might experience a cold start).
- **Use Cases**: highly interactive, low-latency applications like web checkout APIs, mobile app entry points, or synchronous microservices where double-digit millisecond response times are mandatory.
- **Application Auto Scaling**: provisioned concurrency can itself be scaled on a schedule or target-tracking metric (e.g., scale up before a known daily traffic spike).

---

## 8. Throttling

When AWS Lambda receives more concurrent or rapid-fire invocation requests than your account or function-level limits allow, it rejects the excess requests with a **429 TooManyRequestsException**. This rejection behavior is called **throttling**.

### What Is Throttling?
Throttling is AWS Lambda's built-in mechanism to protect its infrastructure and ensure fair resource usage across all customers. When your function's invocation rate or concurrent executions exceed specified quotas, Lambda starts returning 429 errors instead of processing new invocations.

### When Does Throttling Happen?
- Account concurrency limit is exceeded (default 1,000 concurrent executions per Region).
- Reserved concurrency for a specific function is reached if you've set a cap.
- Burst concurrency limit is hit during a sudden spike; Lambda can only launch a finite number of new execution environments in a short period.
- Downstream API calls inside your function are themselves throttled, causing your function to fail or retry.

### Why Does Throttling Happen?
- Shared infrastructure must be protected from "noisy neighbor" effects.
- Prevents runaway costs and resource exhaustion at hyper scale.
- Encourages predictable performance by enforcing throughput boundaries.
- Ensures long-running or heavily loaded functions can't starve others of capacity.

### How to Mitigate Throttling

1. **Monitor and Analyze**
   - Use CloudWatch metrics: `ConcurrentExecutions`, `Throttles`, and `IteratorAge` (for stream-based triggers).
   - Set alarms to detect rising throttle rates early.

2. **Reserve or Provision Concurrency**
   - Reserved Concurrency guarantees minimum capacity or caps max concurrency for critical functions.
   - Provisioned Concurrency warms execution environments ahead of time for consistent performance and higher burst headroom.

3. **Implement Retry and Backoff Logic**
   - For synchronous calls: catch 429 errors and retry with exponential backoff and jitter.
   - For asynchronous or event-source mappings: tune the retry policy, maximum retry attempts, and dead-letter queue.

4. **Smooth Invocation Patterns**
   - Buffer spikes using SQS or Kinesis between your clients and Lambda.
   - Use Step Functions to orchestrate high-volume workflows with built-in error handling.

5. **Optimize Function Runtime**
   - Reduce execution time by right-sizing memory. Shorter functions free up concurrency faster.
   - Break monolithic flows into smaller, independent Lambdas to spread the load.

---

## 9. Layers

AWS Lambda Layers are a way to package libraries, custom runtimes, and other dependencies to be used by multiple Lambda functions. They help manage dependencies, reduce the size of your deployment package, and promote code sharing across your functions.

### How to Set Up Layers for a Python Application

1. **Prepare your dependencies**: create a directory named `python` and place all your necessary libraries (e.g., via `pip install -t python <library_name>`) inside it. This specific directory structure is crucial, as AWS Lambda looks for Python modules in a folder named `python`.
2. **Create a .zip file**: zip the `python` directory. The structure should be `python/your_libraries`.
3. **Upload to AWS Lambda**:
   - Navigate to the AWS Lambda console.
   - Click on **Layers** in the left navigation pane.
   - Click **Create layer**.
   - Provide a name and description for your layer.
   - Upload the .zip file you created.
   - Select the compatible runtime (e.g., Python 3.9).
4. **Attach the layer to your function**:
   - Go to your Lambda function's configuration page.
   - In the **Designer** section, click on **Layers**.
   - Click **Add a layer**.
   - Select the layer you just created from the list and choose a version.
   - Click **Add**.

Your Lambda function can now access the libraries from the layer as if they were in the function's own deployment package.

### Where Are Layers Stored?
Lambda Layers are stored in a centralized location managed by AWS. They are not stored within your function's deployment package. Each layer is versioned, and a specific version of a layer is immutable. When you attach a layer to a function, Lambda mounts the layer's content to the `/opt` directory in the function's execution environment. This is why you need to ensure your libraries are in the `python` folder within the .zip file, as this becomes `/opt/python` in the Lambda environment.

### When to Use Layers
- **Sharing code between functions**: if you have multiple functions that use the same set of libraries (e.g., a database driver, an SDK, or a common utility library), a layer is an efficient way to manage and share them.
- **Reducing deployment package size**: a Lambda function's deployment package has a size limit. By moving bulky libraries to a layer, you can keep your function code small, which speeds up deployments and makes them easier to manage.
- **Managing dependencies**: layers simplify dependency management. Instead of bundling the same large libraries with every function, you can update the library in one place (the layer) and then just update the layer version for all dependent functions.
- **Custom runtimes**: layers can be used to package and deploy custom runtimes, allowing you to write Lambda functions in languages not natively supported by AWS.

### Benefits of Using Layers
- **Smaller deployment packages**: leads to faster deployments and easier management of function code.
- **Reduced build time**: by separating dependencies, you only need to update and deploy the layer when dependencies change, not every single function.
- **Code sharing and reusability**: promotes a more modular and organized architecture by centralizing common components.
- **Improved development workflow**: teams can independently manage function code and shared libraries, leading to a more efficient development process.
- **Simplified management**: it's easier to maintain and update a single layer than to manage dependencies across dozens or hundreds of functions.

### CLI Setup
Setting up AWS Lambda layers for a Python application using the AWS CLI is a three-step process: package your dependencies, publish the layer, and then attach it to your function.

#### Step 1: Package Your Dependencies

For a Python layer, all your packages must be in a folder named `python`.

1. Create the `python` directory:
   ```bash
   mkdir python
   ```
2. Install your required libraries into this directory using `pip`:
   ```bash
   pip install <package-name> -t python/
   ```
   Repeat this for all the packages you need.
3. Zip the `python` folder. This is the file you'll upload to AWS.
   ```bash
   zip -r my-python-layer.zip python/
   ```
   The zipped file will have a structure where the `python` directory is at the root.

#### Step 2: Publish the Layer

Once you have the `.zip` file, you can publish it as a new Lambda layer version using the `aws lambda publish-layer-version` command.

```bash
aws lambda publish-layer-version \
    --layer-name my-python-layer \
    --description "My custom Python dependencies" \
    --zip-file fileb://my-python-layer.zip \
    --compatible-runtimes python3.9 python3.10 python3.11 \
    --region us-east-1
```

- `--layer-name`: a unique name for your layer.
- `--description`: a brief description of what the layer contains.
- `--zip-file fileb://...`: specifies the path to your zipped file. The `fileb://` prefix indicates that the content is a binary file.
- `--compatible-runtimes`: a space-separated list of Python runtimes that can use this layer.
- `--region`: the AWS Region where you want to create the layer.

This command will output a JSON object containing details about the new layer version, including its **LayerVersionArn**. You'll need this ARN to attach the layer to a function.

### Layer Limits
- A function can have up to **5 layers** attached.
- The combined unzipped size of the function and all its layers cannot exceed the **250 MB** deployment package limit.

---

## 10. Lambda with VPC

By default, AWS Lambda functions run in a VPC managed by AWS, not your own. You need to configure a function to run inside your own Virtual Private Cloud (VPC) to access resources that are not publicly available, like an Amazon Relational Database Service (RDS) database or an Amazon ElastiCache cluster.

### VPC Configuration Steps

1. **Create an IAM Role with VPC Permissions**: your function's execution role must have permissions to manage network interfaces. The managed policy `AWSLambdaVPCAccessExecutionRole` grants the necessary permissions: `ec2:CreateNetworkInterface`, `ec2:DescribeNetworkInterfaces`, and `ec2:DeleteNetworkInterface`.
2. **Attach the Function to a VPC**: you configure the Lambda function by specifying a VPC, two or more subnets across different Availability Zones for high availability, and one or more security groups. When the function is invoked, Lambda creates an Elastic Network Interface (**ENI**) in one of your chosen subnets, which allows the function to communicate with other resources inside that VPC.

### Important Considerations
- **No Internet Access by Default**: when a Lambda function is connected to your VPC, it loses its default internet access. If the function needs to connect to the public internet (e.g., to fetch data from a public API), you must configure a **NAT Gateway** in a public subnet and set up routing from the private subnet where the function's ENI resides.
- **Security Groups**: the security groups you assign control the inbound and outbound traffic for the function. You must configure them to allow communication with the specific resources it needs to access within the VPC.
- **Performance**: attaching a Lambda function to a VPC can increase cold start times because of the time it takes for AWS to create and attach the ENI.
- **IP Address Exhaustion**: each ENI consumes an IP address from the subnet. If your function scales up rapidly and the subnets are too small, you could run out of available IP addresses, which would prevent your function from scaling further.
- **Hyperplane ENIs**: to improve performance and reusability, AWS uses a technology called Hyperplane to manage ENIs more efficiently. This allows multiple function invocations to share ENIs.

### Accessing S3/DynamoDB from a VPC-Attached Function
Even from inside a VPC, you don't need a NAT Gateway just to reach S3 or DynamoDB — attach a **Gateway VPC Endpoint** for those services to the function's subnets' route tables, avoiding the NAT Gateway cost/latency entirely for that traffic.

---

## 11. Environment Variables & Configuration

### Setting Environment Variables

#### Using AWS Console
1. Go to the **Lambda function** in the AWS Management Console.
2. Navigate to **Configuration → Environment variables**.
3. Click **Edit**.
4. Add key-value pairs (e.g., `DB_HOST = mydb.example.com`).
5. Click **Save**.

#### Using AWS CLI
```bash
aws lambda update-function-configuration \
  --function-name my-function-name \
  --environment "Variables={DB_HOST=mydb.example.com,API_KEY=xyz123}"
```

#### Accessing in Python
```python
import os

db_host = os.environ['DB_HOST']
api_key = os.environ.get('API_KEY', 'default_value')
```

### Security Best Practices
- **Avoid storing secrets directly** in environment variables.
- Use **AWS Secrets Manager** for sensitive data.
- Enable **encryption at rest** using AWS KMS.
- Use **IAM roles** to restrict access to environment variables and secrets.

### Setting Memory, Timeout, and Concurrency

#### Memory
- Range: **128 MB to 10,240 MB**.
- More memory = more CPU (proportional scaling) — CPU allocation scales linearly with memory, so a CPU-bound function can sometimes run **faster and cheaper overall** at a higher memory setting despite the higher per-ms rate, because total duration drops enough to offset it.

#### Timeout
- Max: **15 minutes**.
- Set based on expected execution time — a too-generous timeout can let a hung function burn cost silently; a too-tight one causes false failures.

#### Concurrency
- **Reserved concurrency**: guarantees a set number of concurrent executions.
- **Provisioned concurrency**: pre-warms Lambda instances to reduce cold starts.

### Set via Console or CLI
```bash
aws lambda update-function-configuration \
  --function-name my-function-name \
  --memory-size 1024 \
  --timeout 300 \
  --reserved-concurrent-executions 10
```

---

## 12. Versioning & Aliases

Managing **Lambda versioning and aliases** is essential for deploying and maintaining serverless applications across environments like **dev**, **test**, and **prod**.

### What is a Version?
- A **version** is a snapshot of your Lambda function code and configuration.
- Once published, it's **immutable** — you can't change the code or settings.
- Useful for rollback, testing, and stable deployments.
- `$LATEST` is the mutable, unpublished version that always reflects your most recent code edits.

### How to Create a Version

**Console:**
1. Go to your Lambda function.
2. Click **Actions → Publish new version**.
3. Add a description and publish.

**CLI:**
```bash
aws lambda publish-version --function-name my-function
```

### What is an Alias?
- An **alias** is a pointer to a specific version of your Lambda function.
- You can name aliases like `dev`, `test`, `prod`.
- Aliases can be updated to point to new versions without changing the function name.

### How to Create and Use Aliases

**Console:**
1. Go to your Lambda function → **Aliases** tab.
2. Click **Create alias**.
3. Name it (e.g., `prod`) and choose a version.

**CLI:**
```bash
aws lambda create-alias --function-name my-function --name prod --function-version 5
```

**Updating an Alias**
```bash
aws lambda update-alias --function-name my-function --name prod --function-version 6
```

### Weighted Aliases (Canary / Blue-Green Deployments)
An alias can split traffic between **two versions** by weight, enabling gradual rollout:
```bash
aws lambda update-alias \
  --function-name my-function \
  --name prod \
  --function-version 6 \
  --routing-config AdditionalVersionWeights={"5"=0.1}
```
This sends 10% of traffic to version 5 and 90% to version 6 — the basis for canary deployments, often automated further with **AWS CodeDeploy** (linear/canary deployment presets with automatic rollback on CloudWatch alarm).

### Benefits of Using Versions & Aliases
- **Safe deployments**: test new versions before updating `prod`.
- **Blue/Green deployments**: shift traffic gradually using alias weights.
- **Rollback**: quickly revert to a previous version.
- **Environment isolation**: separate dev/test/prod without duplicating functions.

---

## 13. IAM Execution Role

Every Lambda function requires an **execution role** — an IAM role that the Lambda service assumes to run your function, granting it permission to interact with other AWS services and to write logs.

### Key Points
- The execution role's **trust policy** must allow the `lambda.amazonaws.com` service principal to assume it.
- At minimum, the role needs `AWSLambdaBasicExecutionRole` (permissions to create CloudWatch Log Groups/Streams and put log events) — without it, your function will run but you'll have no logs.
- Additional managed policies are attached based on what the function needs to do:
  - `AWSLambdaVPCAccessExecutionRole` — for VPC-attached functions ([Section 10](#10-lambda-with-vpc)).
  - Custom policies scoped to specific resources (e.g., `dynamodb:PutItem` on one specific table) — always follow least privilege rather than attaching broad managed policies like `AmazonDynamoDBFullAccess`.
- **Resource-based policies** on the function itself (separate from the execution role) control **who can invoke** the function — e.g., granting `lambda:InvokeFunction` to API Gateway or an S3 bucket's event notification system. This is the inverse of the execution role: the execution role is what Lambda can do *outward*; the resource policy is who can call *inward*.

---

## 14. Error Handling, Retries, and Destinations

### Synchronous Invocations
- Errors are returned directly to the caller (e.g., an API Gateway 502/500) — the caller is responsible for implementing its own retry logic.

### Asynchronous Invocations
- Lambda automatically retries a failed invocation **up to 2 times** by default, with a delay between attempts.
- After retries are exhausted, the event can be routed to:
  - **Dead Letter Queue (DLQ)** — a legacy mechanism sending the failed event to an SQS queue or SNS topic (captures only the original event payload).
  - **On-failure Destination** (newer, more capable) — can route to SQS, SNS, Lambda, or **EventBridge**, and includes richer context (invocation metadata, error details) beyond just the raw event.
- An **on-success destination** can also be configured, to chain asynchronous workflows without writing custom glue code.

### Poll-Based (Event Source Mapping) Invocations
- Retry behavior is governed by the event source mapping's own settings: `MaximumRetryAttempts`, `MaximumRecordAgeInSeconds`, and `BisectBatchOnFunctionError` (streams only).
- An ESM-level **on-failure destination** can capture batches that ultimately fail, so they aren't silently dropped (critical for Kinesis/DynamoDB Streams where a stuck record otherwise blocks the whole shard — see [Section 6](#6-event-source-mappings-poll-based-sources)).

### DLQ vs Destinations

| Aspect | Dead Letter Queue (DLQ) | On-Failure/On-Success Destination |
|---|---|---|
| Targets | SQS, SNS only | SQS, SNS, Lambda, EventBridge |
| Captures success path | ❌ No | ✅ Yes (on-success) |
| Payload richness | Just the original event | Event + invocation/error metadata |
| Recommended today | Legacy, still supported | ✅ Preferred for new functions |

---

## 15. Packaging: Zip Archives vs Container Images

Lambda functions can be packaged two ways:

| Aspect | Zip Archive | Container Image |
|---|---|---|
| Max size | 50 MB (zipped, direct upload) / 250 MB (unzipped, via S3) | Up to **10 GB** |
| Storage | S3 | Amazon ECR |
| Base image | Lambda-managed runtime | Any OCI-compliant image, including custom base images, as long as it implements the Lambda Runtime API |
| Best for | Smaller functions, simpler dependency trees | Large dependencies (e.g., ML libraries, native binaries), teams already using container-based CI/CD |
| Local testing | Harder to fully replicate the Lambda runtime locally | Easier — the same container image can be tested with the **Lambda Runtime Interface Emulator (RIE)** |
| Cold start | Generally comparable since Lambda lazily loads container layers, though very large images can still add latency | Can be comparable to zip if layers are optimized |

---

## 16. Ephemeral Storage (/tmp) & Idempotency

### Ephemeral Storage
- Every Lambda execution environment provides a `/tmp` directory for temporary file storage.
- Configurable from **512 MB (default) up to 10,240 MB (10 GB)**, billed per MB-second provisioned, separate from function memory.
- **Persists across warm invocations** of the *same* execution environment (not guaranteed across different environments) — useful for caching downloaded assets, but must never be relied upon as durable storage, since the environment can be recycled at any time.

### Idempotency
Because Lambda can invoke your function **more than once for the same event** — a redelivered SQS message, an asynchronous retry, or a duplicate stream record — functions should be written to be **idempotent**: processing the same event twice produces the same result as processing it once.

- Common pattern: use a **conditional write** (e.g., DynamoDB `ConditionExpression` on an idempotency key) to detect and skip already-processed events.
- AWS provides the **Lambda Powertools idempotency utility** (available for several runtimes) to handle this pattern with minimal boilerplate, typically backed by a DynamoDB table.

---

## 17. Function URLs

**Lambda Function URLs** provide a dedicated, built-in HTTPS endpoint for a function, without needing to set up API Gateway.

### Key Points
- Enabled per function (or per alias), generating a unique URL like `https://<url-id>.lambda-url.<region>.on.aws/`.
- Supports two **auth types**: `AWS_IAM` (SigV4-signed requests only) or `NONE` (public, open endpoint — access control must then be handled entirely inside your function code if needed).
- Supports **CORS configuration** directly on the function URL.
- Simpler and cheaper than API Gateway for basic single-function HTTP use cases, but lacks API Gateway's richer features: request validation, usage plans/API keys, custom domain mapping (though Function URLs can be fronted by CloudFront for a custom domain), request throttling per client, and multi-route/multi-method API definitions.

### Function URL vs API Gateway

| Aspect | Function URL | API Gateway |
|---|---|---|
| Setup complexity | Minimal — a few clicks/one API call | More involved — routes, stages, integrations |
| Cost | Free (pay only for Lambda invocation) | Additional per-request + data transfer cost |
| Features | Basic HTTP(S) endpoint, CORS, IAM/no auth | Full REST/HTTP API feature set: throttling, API keys, request validation, custom authorizers, multiple routes |
| Best for | Simple webhooks, single-purpose APIs, prototypes | Production multi-route APIs needing fine-grained control |

---

## 18. Lambda Extensions

**Lambda Extensions** let you augment a function's execution environment with additional processes that run alongside your function code — e.g., for monitoring, security, or configuration management — without modifying your function's own code.

### Types
- **Internal extensions**: run in-process as part of the runtime (e.g., a wrapper script).
- **External extensions**: run as a **separate process** within the same execution environment, able to hook into the INIT, INVOKE, and SHUTDOWN lifecycle phases via the **Extensions API**.

### Common Use Cases
- Streaming logs/metrics/traces to a third-party observability platform (Datadog, New Relic, etc.) without instrumenting your own code.
- Fetching and caching secrets or configuration at INIT time (e.g., the **AWS Parameter and Secrets Lambda Extension**, which caches Secrets Manager/SSM Parameter Store values locally to reduce latency and API call volume on every invocation).
- Security/compliance agents that need visibility into the function's runtime behavior.

### Key Point
- Extensions add to the function's **INIT** duration (and therefore billed cold-start time) and consume part of the function's configured memory — they're not "free" just because they run outside your own code.

---

## 19. Monitoring & Observability

### CloudWatch Logs
- Every invocation's `print`/logging output is automatically sent to a **CloudWatch Log Group** named `/aws/lambda/<function-name>`, as long as the execution role has `AWSLambdaBasicExecutionRole` ([Section 13](#13-iam-execution-role)).
- Each invocation's log stream includes a `START`, `END`, and `REPORT` line — the `REPORT` line shows billed duration, memory used/configured, and init duration (useful for spotting cold starts and right-sizing memory).

### CloudWatch Metrics
Key built-in metrics per function: `Invocations`, `Errors`, `Duration`, `Throttles`, `ConcurrentExecutions`, `IteratorAge` (stream sources), `DeadLetterErrors`, `DestinationDeliveryFailures`.

### AWS X-Ray
- Enabling **Active Tracing** on a function instruments it to send trace segments to X-Ray, letting you visualize the full request path across services (API Gateway → Lambda → DynamoDB, etc.) and pinpoint where latency is coming from.
- Requires the `AWSXRayDaemonWriteAccess` policy (or equivalent permissions) on the execution role.

### Lambda Insights
- An optional CloudWatch Logs/Lambda extension that collects enhanced system-level metrics (CPU, memory, network, disk I/O within the execution environment) beyond the standard function metrics — useful for diagnosing resource-bound performance issues.

---

## 20. Lambda Limits & Quotas

A quick-reference table of commonly tested limits (always worth verifying against current AWS docs, as these can change):

| Limit | Value |
|---|---|
| Memory | 128 MB – 10,240 MB |
| Timeout (max) | 15 minutes |
| Ephemeral storage (`/tmp`) | 512 MB (default) – 10,240 MB |
| Deployment package (zip, direct upload) | 50 MB compressed |
| Deployment package (zip, via S3) | 250 MB uncompressed |
| Container image size | 10 GB |
| Environment variables (total size) | 4 KB |
| Layers per function | 5 |
| Concurrent executions (default account limit) | 1,000 per Region (soft limit, increasable) |
| Synchronous invocation payload | 6 MB |
| Asynchronous invocation payload | 256 KB |
| Function URL / RIE payload | 6 MB (request), 20 MB (response, streaming) |

---

## 21. Pricing

Lambda pricing has two core components:

1. **Requests**: charged per invocation (first ~1M requests/month free under the perpetual free tier).
2. **Duration**: charged per GB-second — (memory allocated in GB) × (execution duration, billed in 1-ms increments) — so both memory size and runtime directly drive cost.

### Key Cost Levers
- **Right-sizing memory**: since CPU scales with memory, increasing memory can sometimes *reduce* total cost by shortening duration enough to offset the higher per-ms rate — worth profiling rather than assuming "less memory = cheaper."
- **Provisioned Concurrency** adds its own separate hourly charge, billed whether or not it's actively invoked — a deliberate latency-vs-cost trade-off.
- **ARM/Graviton2 (`arm64`) architecture** is typically ~20% cheaper per GB-second than `x86_64` for comparable performance, when your code/dependencies support it.

---

## 22. Lambda@Edge vs CloudFront Functions

Both let you run code at AWS edge locations in response to CloudFront events, but they target very different workloads.

| Aspect | Lambda@Edge | CloudFront Functions |
|---|---|---|
| Runtime | Node.js or Python (full Lambda runtime) | Lightweight JavaScript (ECMAScript subset) |
| Max execution time | Up to 5s (viewer) / 30s (origin) depending on event type | Sub-millisecond, hard-capped at ~1ms |
| CloudFront trigger points | Viewer request/response, Origin request/response (all 4) | Viewer request/response only |
| Use cases | Complex logic: A/B testing, auth with external calls, image manipulation, SSR | High-volume, simple logic: header manipulation, URL rewrites/redirects, basic auth checks |
| Pricing | Charged like a normal Lambda invocation, replicated to edge locations | Much cheaper per-invocation, designed for very high request volumes |
| Access to network/other AWS services | ✅ Yes | ❌ No (pure, isolated JS execution) |

> **Interview tip:** Default to **CloudFront Functions** for simple, extremely high-volume edge logic (it's cheaper and faster); reach for **Lambda@Edge** only when you need full Node/Python capability (network calls, npm packages) or need to hook the origin request/response events.

---

## 23. Lambda Best Practices

1. **Initialize outside the handler** to amortize cold-start cost across warm invocations ([Section 3](#3-cold-start-vs-warm-start)).
2. **Keep functions small and single-purpose** — easier to test, deploy, and right-size memory/timeout for.
3. **Use Layers** for shared dependencies instead of duplicating them in every function's package ([Section 9](#9-layers)).
4. **Set Reserved Concurrency on critical functions** that protect fragile downstream systems (e.g., a database with a low connection limit) ([Section 7](#7-concurrency)).
5. **Design for idempotency** — assume at-least-once delivery for async and poll-based invocations ([Section 16](#16-ephemeral-storage-tmp--idempotency)).
6. **Use destinations over DLQs** for new functions — richer failure context and more target options ([Section 14](#14-error-handling-retries-and-destinations)).
7. **Avoid unnecessary VPC attachment** — only attach a function to a VPC if it truly needs to reach private resources (RDS, ElastiCache); it adds cold-start overhead otherwise ([Section 10](#10-lambda-with-vpc)).
8. **Use Provisioned Concurrency or SnapStart** for latency-sensitive, user-facing synchronous APIs ([Section 3](#3-cold-start-vs-warm-start)).
9. **Follow least privilege on the execution role** — scope IAM permissions to exactly the resources/actions the function needs ([Section 13](#13-iam-execution-role)).
10. **Use versions and aliases** for safe, gradual (canary) deployments instead of overwriting `$LATEST` directly in production ([Section 12](#12-versioning--aliases)).
11. **Monitor `IteratorAge`, `Throttles`, and `Errors`** proactively with CloudWatch alarms rather than discovering problems from user complaints ([Sections 8, 19](#8-throttling)).

---

## 24. Common Interview Questions

1. **What's the difference between a cold start and a warm start?** → [Section 3](#3-cold-start-vs-warm-start).
2. **How do you eliminate cold starts for a latency-sensitive API?** → Provisioned Concurrency, or SnapStart for supported runtimes ([Section 3](#3-cold-start-vs-warm-start)).
3. **What's the difference between Reserved and Provisioned Concurrency?** → [Section 7](#7-concurrency) — one caps/guarantees a concurrency slice (free), the other pre-warms environments to kill cold starts (paid).
4. **Why would a Kinesis/DynamoDB Streams consumer "get stuck"?** → A single poison-pill record blocks the shard until retries are exhausted, bisect-on-error isolates it, or a failure destination is configured ([Section 6](#6-event-source-mappings-poll-based-sources)).
5. **How does error handling differ for sync vs async vs poll-based invocations?** → [Sections 4, 14](#4-invocation-types) — caller-handled, Lambda auto-retry + DLQ/destination, and ESM-configured retry respectively.
6. **How do you let a Lambda function reach an RDS database in a private subnet?** → Attach the function to the VPC with the right subnets/security groups ([Section 10](#10-lambda-with-vpc)).
7. **How do you avoid NAT Gateway costs for a VPC-attached function calling S3?** → A Gateway VPC Endpoint for S3 ([Section 10](#10-lambda-with-vpc)).
8. **How do you safely roll out a new function version?** → Weighted alias canary deployment, optionally automated with CodeDeploy ([Section 12](#12-versioning--aliases)).
9. **How do you share a database connection/SDK client across invocations?** → Initialize it in global scope, outside the handler ([Section 3](#3-cold-start-vs-warm-start)).
10. **When would you use a container image instead of a zip package?** → Large dependencies (>250 MB unzipped), existing container CI/CD tooling, or needing up to 10 GB package size ([Section 15](#15-packaging-zip-archives-vs-container-images)).
11. **Function URL vs API Gateway — when would you pick each?** → [Section 17](#17-function-urls) — simple single-function HTTP needs vs. full-featured multi-route APIs.
12. **How do you make sure a function doesn't process the same event twice incorrectly?** → Design for idempotency, typically via a conditional write keyed on an idempotency token ([Section 16](#16-ephemeral-storage-tmp--idempotency)).
13. **CloudFront Functions vs Lambda@Edge — which would you use for a simple header rewrite at massive scale?** → CloudFront Functions, for cost and latency ([Section 22](#22-lambdaedge-vs-cloudfront-functions)).
