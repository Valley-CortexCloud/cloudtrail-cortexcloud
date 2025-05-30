# AWS CloudTrail Log Ingestion for Cortex Cloud + XSIAM 🛡️

This guide outlines the steps to enable AWS CloudTrail log ingestion into Cortex XSIAM using the provided AWS CloudFormation template. This setup will configure an S3 bucket (either new or existing) to send CloudTrail log notifications to an SQS queue, which Cortex XSIAM will then process.

**Important Prerequisites:**
* You must have an active AWS CloudTrail trail configured in your AWS account.
* If you choose to create a **new S3 bucket** (Option 1), you will need to update your CloudTrail trail's configuration to deliver logs to this newly created bucket *after* the CloudFormation stack deployment is complete.
* If you choose to use an **existing S3 bucket** (Option 2), ensure your CloudTrail trail is already delivering logs to it.

---

## 🚀 Option 1: New S3 Bucket for CloudTrail Logs

Follow these steps if you want the CloudFormation template to create a new S3 bucket to store your CloudTrail logs. The template will also automatically configure this new S3 bucket to send event notifications directly to the SQS queue it creates.

### Running the CloudFormation Template

1.  Navigate to the AWS CloudFormation console.
2.  Click on **Create stack** (with new resources).
3.  Upload or specify the CloudFormation template file.
4.  Provide a **Stack Name**, for example:
    * `CortexCloudTrailLogIngestion`
5.  Configure the **Parameters** as follows:
    * **Create New S3 Bucket?**: `Yes`
    * **ARN of Existing S3 Bucket (if not creating new)**: Leave this field **BLANK**.
    * **New S3 Bucket Name (if creating new)**: Provide a globally unique S3 bucket name (e.g., `my-cortex-cloudtrail-logs-unique`). The template will append the AWS Account ID to ensure uniqueness.
    * **SQS Queue Name**: Provide a base name for the SQS queue (e.g., `cortex-cloudtrail-sqs`). The template will append the AWS Region for uniqueness.
    * **Cortex XSIAM AWS Account ID**: Enter the AWS Account ID provided by Cortex XSIAM for cross-account access (e.g., `006742885340`).
    * **External ID from Cortex XSIAM**: Generate a random UUID v4 (e.g., using [https://www.uuidgenerator.net/version4](https://www.uuidgenerator.net/version4)) and paste it here. This is used for the IAM role assumption.
    * **DLQ Max Receive Count**: Leave default or adjust as needed (e.g., `3`).
6.  Acknowledge IAM resource creation if prompted, and proceed through the remaining CloudFormation stack creation steps to create the stack.
7.  **Critical Post-Deployment Step:** Once the CloudFormation stack has been successfully created, you **must** configure your AWS CloudTrail trail to deliver logs to the **newly created S3 bucket**. You can find the bucket name in the **Outputs** tab of the CloudFormation stack (`CloudTrailLogsBucketName`).

---

## 📂 Option 2: Existing S3 Bucket for CloudTrail Logs

Follow these steps if you already have an S3 bucket where your CloudTrail logs are being delivered. The CloudFormation template will create the SQS queue and IAM roles, but you will need to **manually configure** your existing S3 bucket to send event notifications to the SQS queue created by this stack.

### Running the CloudFormation Template

1.  Navigate to the AWS CloudFormation console.
2.  Click on **Create stack** (with new resources).
3.  Upload or specify the CloudFormation template file.
4.  Provide a **Stack Name**, for example:
    * `CortexCloudTrailLogIngestion`
5.  Configure the **Parameters** as follows:
    * **Create New S3 Bucket?**: `No`
    * **ARN of Existing S3 Bucket (if not creating new)**: Enter the ARN of your existing S3 bucket that houses the CloudTrail logs (e.g., `arn:aws:s3:::your-existing-cloudtrail-log-bucket`). **Ensure you have the correct format.**
    * **New S3 Bucket Name (if creating new)**: Leave this field **BLANK** or with its default empty value.
    * **SQS Queue Name**: Provide a base name for the SQS queue (e.g., `cortex-cloudtrail-sqs`). The template will append the AWS Region for uniqueness.
    * **Cortex XSIAM AWS Account ID**: Enter the AWS Account ID provided by Cortex XSIAM for cross-account access (e.g., `006742885340`).
    * **External ID from Cortex XSIAM**: Generate a random UUID v4 (e.g., using [https://www.uuidgenerator.net/version4](https://www.uuidgenerator.net/version4)) and paste it here.
    * **DLQ Max Receive Count**: Leave default or adjust as needed (e.g., `3`).
6.  Acknowledge IAM resource creation if prompted, and proceed through the remaining CloudFormation stack creation steps to create the stack.

---

## ⚠️ Important: Manual Configuration for Existing S3 Bucket

If you selected **No** for `Create New S3 Bucket?` and provided an `ARN of Existing S3 Bucket`, you **must manually configure** the S3 event notification on that existing bucket. This allows the bucket to send `s3:ObjectCreated:*` events directly to the SQS queue created by this CloudFormation stack.

### Steps to Manually Configure S3 Event Notification:

1.  **Navigate to S3**: Go to the AWS S3 console.
2.  **Select Bucket**: Navigate to your **existing S3 bucket** that stores the CloudTrail logs.
3.  **Properties Tab**: Go to the **Properties** tab of the selected bucket.
4.  **Event Notifications**: Scroll down to the **Event notifications** section.
5.  **Create Event Notification**: Click **Create event notification**.
    * **Event name**: Give it a meaningful name (e.g., `CortexXSIAMCloudTrailNotification`).
    * **Prefix (optional)**: If your CloudTrail logs are stored under a specific prefix (e.g., `AWSLogs/YOUR_ACCOUNT_ID/CloudTrail/`), specify it here. This helps ensure that only relevant object creation events trigger the notification.
    * **Suffix (optional)**: Usually `.json.gz` for CloudTrail logs. Specify this to further filter events.
    * **Event types**: Select **All object create events** (which corresponds to `s3:ObjectCreated:*`).
    * **Destination**:
        * Choose **SQS Queue**.
        * Select **Choose from your SQS queues**.
        * In the **SQS queue** dropdown, find and select the SQS queue created by your CloudFormation stack.
            * The queue name will be based on the `SQS Queue Name` parameter you provided, with the region appended (e.g., `cortex-cloudtrail-sqs-us-east-1`).
            * You can retrieve the exact SQS Queue ARN from the **Outputs** tab of your CloudFormation stack (look for `SQSQueueARN`).
6.  **Save Changes**: Click **Save changes**.

✅ **Completion**: Once this manual S3 event notification is configured (for an existing bucket) or your CloudTrail is delivering logs to the newly created bucket, S3 will start sending notifications to the SQS queue. Cortex XSIAM will then be able to ingest the CloudTrail logs.

---

Refer to the Cortex XSIAM documentation to confirm data ingestion and for any further steps required within the XSIAM platform.
