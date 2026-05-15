# Cortex XSIAM Integration: AWS Control Tower Centralized CloudTrail Logs

This guide provides the necessary steps to securely integrate Cortex XSIAM with an existing AWS Control Tower environment. It is designed to strictly adhere to Control Tower guardrails while establishing a highly robust, least-privilege ingestion pipeline for your organizational CloudTrail logs.

## Architecture Overview
In a standard AWS Control Tower landing zone, resources are decentralized across specific specialized accounts to enforce separation of duties. This integration operates across the following functional boundaries:
* **KMS Administrator Account:** The account managing the central AWS KMS Customer Managed Key (CMK) used to encrypt the CloudTrail logs at rest.
* **Log Archive Account:** The AWS Control Tower managed account hosting the centralized Amazon S3 bucket where organizational CloudTrail logs are stored.
* **Security Tooling Account:** The consumer account where the Cortex XSIAM integration resources (Amazon SQS Queue, Dead Letter Queue, and Cross-Account IAM Role) will be deployed.

---

## ⚠️ CRITICAL DEPLOYMENT PREREQUISITE
You **MUST** deploy the initial CloudFormation template in the **exact same AWS Region** as your Control Tower centralized S3 bucket (located in the Log Archive Account). AWS strictly requires the source S3 bucket and the destination SQS queue to reside in the same geographic region for S3 Event Notifications to function correctly.

---

## Phase 1: Deploy Cortex Integration Resources (Automated)

Deploy the provided CloudFormation template into your designated **Security Tooling Account**. This establishes the queuing infrastructure and the cross-account role required by Cortex XSIAM.

### 1. Gather Required Parameters
Before initiating the deployment, collect the following environmental variables:
* **CentralBucketName:** The exact name of the centralized Log Archive S3 bucket (e.g., `aws-controltower-logs-...`).
* **S3BucketAccountId:** The 12-digit AWS Account ID of your Log Archive Account.
* **KmsKeyArn:** The ARN of the KMS key used for log encryption (leave blank if your bucket uses default Amazon S3 managed keys).
* **CortexXSIAMAccountID:** The Palo Alto Networks AWS account ID provided in your Cortex UI (typically `006742885340` for standard or `685269782068` for FedRAMP environments).
* **ExternalId:** The unique External ID generated in your Cortex XSIAM setup portal (utilized to prevent the confused deputy problem). Generate a random UUID v4 (e.g., using https://www.uuidgenerator.net/version4) and paste it here. This is used for the IAM role assumption.

### 2. Execute the CloudFormation Deployment
1. Log in to the **Security Tooling Account** (Ensure your AWS Console region matches your Log Archive region).
2. Navigate to **CloudFormation** > **Create stack** > **With new resources (standard)**.
3. Upload the integration template (`cortex-xsiam-control-tower.yaml`).
4. Input the parameters gathered in the previous step.
5. Acknowledge the IAM resource creation warning at the bottom of the review page: *"I acknowledge that AWS CloudFormation might create IAM resources."*
6. Click **Submit**.
7. Upon successful deployment, navigate to the **Outputs** tab and record the following generated values for use in subsequent phases: `SQSQueueURL`, `SQSQueueARN`, and `CortexRoleArn`.

---

## Phase 2: Grant KMS Decrypt Permissions (Manual Policy Update)

*Note: Because AWS Control Tower natively manages the centralized KMS key, this key policy must be updated manually. Modifying this policy via third-party infrastructure-as-code will trigger strict Control Tower drift violations.*

1. Log in to the **KMS Administrator Account**.
2. Navigate to **KMS** > **Customer managed keys** and select your Control Tower CloudTrail key.
3. Under the **Key policy** tab, click **Edit**.
4. Append the following least-privilege statement to the existing JSON `Statement` array. 
   * *Replace `<CORTEX_ROLE_ARN>` with the `CortexRoleArn` from Phase 1 Outputs.*
   * *Replace `<REGION>` with your target AWS Region.*
   * *Replace `<CENTRAL_BUCKET_NAME>` with your Log Archive S3 bucket name.*

```json
{
    "Sid": "AllowCortexXSIAMToDecryptCloudTrailLogs",
    "Effect": "Allow",
    "Principal": {
        "AWS": "<CORTEX_ROLE_ARN>"
    },
    "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
    ],
    "Resource": "*",
    "Condition": {
        "StringEquals": {
            "kms:ViaService": "s3.<REGION>.amazonaws.com"
        },
        "StringLike": {
            "kms:EncryptionContext:aws:s3:arn": "arn:aws:s3:::<CENTRAL_BUCKET_NAME>/*"
        }
    }
}
```
5. Click **Save changes**.

---

## Phase 3: Configure Log Routing via S3 Event Notifications (Manual Configuration)

*Note: The central S3 bucket is strictly managed by Control Tower baselines. We manually configure the S3 Event Notification to securely route new log objects to our SQS queue without altering the underlying bucket configuration or triggering drift.*

1. Log in to the **Log Archive Account**.
2. Navigate to **S3** and select the centralized CloudTrail bucket.
3. Select the **Properties** tab.
4. Scroll to the **Event notifications** section and click **Create event notification**.
5. **Event name:** Enter `CortexXSIAM-SQS-Routing`.
6. **Event types:** Check the box for **All object create events** (`s3:ObjectCreated:*`).
7. **Destination:** Select **SQS queue**.
8. **Specify SQS queue:** Choose **Enter SQS queue ARN**.
9. Paste the `SQSQueueARN` recorded from the Phase 1 Outputs.
10. Click **Save changes**.

---

## Phase 4: Finalize the Cortex XSIAM Data Source Connection

1. Return to the Cortex XSIAM Console.
2. Navigate to **Settings** > **Data Sources** > **Add Data Source** > **Amazon S3**.
3. Configure the SQS connection parameters:
   * **Connection Type:** `SQS`
   * **SQS URL:** Paste the `SQSQueueURL` (from Phase 1 Outputs).
   * **Role ARN:** Paste the `CortexRoleArn` (from Phase 1 Outputs).
   * **External ID:** Paste your unique External ID.
4. Click **Test Connection**, then click **Enable**.
5. Allow approximately 10-15 minutes for logs to populate. You can verify ingestion success by executing the following XQL query within Cortex XSIAM:

```xql
dataset = cloud_audit_logs | filter (identity_name contains "CortexXSIAMCloudTrailRole")
```
