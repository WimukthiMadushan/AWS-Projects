# Event-Driven Architecture with AWS (S3, SNS, SQS, Lambda)

This project demonstrates a robust, serverless **Event-Driven Architecture** using AWS services. It leverages the **Fan-out** messaging pattern to decouple event generation from processing, ensuring high scalability, fault tolerance, and reliability.

## Architecture Overview

![Architecture Diagram](../Images/image.png)

### The Workflow

1.  **Event Source (Event Producer)**: A file is uploaded to an **Amazon S3** bucket.
2.  **Event Ingestion (Event Router)**: S3 detects the upload and publishes an event notification to an **Amazon SNS** (Simple Notification Service) topic.
3.  **Event Stream (Event Buffer)**: The SNS topic broadcasts the notification to its subscribers. An **Amazon SQS** (Simple Queue Service) queue is subscribed to this topic and receives the message.
4.  **Event Consumer**: An **AWS Lambda** function is triggered by the SQS queue. It processes the message allowing it to perform actions on the uploaded file (e.g., processing, transformation, or logging).

---

## Components

| Component    | AWS Service    | Role                                                                                        |
| :----------- | :------------- | :------------------------------------------------------------------------------------------ |
| **Producer** | **Amazon S3**  | Object storage service that triggers notifications upon object creation/upload.             |
| **Router**   | **Amazon SNS** | Pub/Sub messaging service that fans out messages to multiple subscribers (decoupling).      |
| **Buffer**   | **Amazon SQS** | Message queue that buffers events, preventing data loss and managing load for the consumer. |
| **Consumer** | **AWS Lambda** | Serverless compute service that runs code in response to events from the queue.             |

## Key Concepts

### Why use SNS + SQS?

Using SNS _and_ SQS together provides the **Fan-out** pattern with **persistence**.

- **SNS** allows multiple different services to react to the same S3 event (e.g., one queue for image resizing, another for logging, another for analytics) without changing the S3 configuration.
- **SQS** provides a buffer. If the Lambda function is overwhelmed or fails, the message sits safely in the queue to be retried later, rather than being lost.

### Benefits

- **Decoupling**: The upload process doesn't wait for the processing to finish.
- **Scalability**: All components (S3, SNS, SQS, Lambda) scale automatically with the volume of requests.
- **Reliability**: Retries and Dead Letter Queues (DLQ) ensure no events are lost during processing failures.

---

## Setup Instructions

### Step 1: Create an S3 Bucket (Event Producer)

1.  Navigate to the **S3 Console**.
2.  Create a new bucket (e.g., `event-driven-bucket`).

### Step 2: Create an SNS Topic (Event Router)

1.  Navigate to the **SNS Console**.
2.  Create a **Standard** topic (e.g., `s3-upload-notifications`).
3.  **Access Policy**: Use the provided `sns-access-policy.json` file.
    - Replace `<region>`, `<accountId>`, `<snsTopicName>`, and `<s3BucketName>` with your specific values.
    - This policy allows the S3 bucket created in Step 1 to publish messages to this topic.

### Step 3: Create an SQS Queue (Event Buffer)

1.  Navigate to the **SQS Console**.
2.  Create a **Standard** queue (e.g., `s3-event-process-queue`).
3.  **Access Policy**: Use the provided `sqs-access-policy.json` file.
    - Replace `<region>`, `<accountId>`, `<sqsQueueName>`, and `<lambdaName>` placeholders.
    - This specifically allows the SNS topic to send messages to the queue.
4.  **Subscribe to SNS**:
    - Go to the SNS topic created in Step 2.
    - Create a subscription.
    - Protocol: **Amazon SQS**.
    - Endpoint: Select this SQS queue.

### Step 4: Configure S3 Event Notifications

1.  Go back to your **S3 Bucket** > **Properties** > **Event notifications**.
2.  Create a new notification.
3.  **Event types**: Select `Put` (or `All object create events`).
4.  **Destination**: Select **SNS Topic**.
5.  Choose the SNS topic created in Step 2.

### Step 5: Lambda Function (Event Consumer)

1.  Navigate to the **Lambda Console**.
2.  Create a function (e.g., `ProcessS3Events`).
3.  **IAM Role**: Ensure the execution role has permissions to:
    - Read from SQS (`sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`).
    - Write to CloudWatch Logs.
4.  **Trigger**: Add the SQS queue from Step 3 as a trigger.
