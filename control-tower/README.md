# Cortex XSIAM Integration: AWS Control Tower Centralized CloudTrail Logs

This guide walks you through securely integrating Cortex XSIAM with an existing AWS Control Tower environment. It is designed to respect Control Tower's managed resources while establishing a robust, least-privilege ingestion pipeline for your centralized CloudTrail logs.

## Architecture Overview
In a standard Control Tower setup, resources are split across specific accounts to maintain separation of duties:
* **Account A (Key Management):** Owns the central KMS Key used to encrypt logs.
* **Account B (Log Archive):** Owns the central S3 Bucket storing the CloudTrail logs.
* **Account C (Security/Tooling):** The consumer account where we will deploy the Cortex XSIAM resources (SQS Queue and IAM Role).

---

## ⚠️ CRITICAL DEPLOYMENT PREREQUISITE
You **MUST** deploy the CloudFormation template in the **exact same AWS Region** as your Control Tower centralized S3 bucket (Account B). AWS strictly requires the S3 bucket and the SQS destination queue to reside in the same geographic region for S3 Event Notifications to function.

---

## Phase 1: Deploy Cortex Resources (Automated)

Deploy the provided CloudFormation template into your designated Security/Tooling Account (Account C).

### 1. Gather Required Parameters
Before deploying, collect the following information:
* **CentralBucketName:** Name of the Log Archive S3 bucket (e.g., `aws-controltower-logs-...`).
* **S3BucketAccountId:** The AWS Account ID for Account B.
* **KmsKeyArn:** The ARN of the KMS key in Account A (leave blank if encryption is disabled).
* **CortexXSIAMAccountID:** The Palo Alto Networks account from your Cortex UI (typically `006742885340` or `685269782068` for FedRAMP).
* **ExternalId:** The unique ID from your Cortex XSIAM setup page (prevents the confused deputy problem).

### 2. Deploy the Stack
1. Log in to **Account C** (Ensure your AWS Console is set to the correct region!).
2. Navigate to **CloudFormation** > **Create stack** > **With new resources (standard)**.
3. Upload the provided template (`cortex-xsiam-control-tower.yaml`).
4. Fill in the parameters gathered above.
5. Check the acknowledgment box at the bottom: *"I acknowledge that AWS CloudFormation might create IAM resources."*
6. Click **Submit**.
7. Once deployment is complete, go to the **Outputs** tab and copy all three values (`SQSQueueURL`, `SQSQueueARN`, and `CortexRoleArn`). You will need these for the next steps.

---

## Phase 2: Grant KMS Decrypt Access (Account A - Manual)

*Note: Because AWS Control Tower natively manages the KMS key, this policy must be appended manually. Attempting to manage it via third-party CloudFormation will cause strict drift violations.*

1. Log in to **Account A** (Key Management).
2. Navigate to **KMS** > **Customer managed keys** and select your Control Tower key.
3. Under **Key policy**, click **Edit**.
4. Append the following least-privilege statement to the existing JSON `Statement` array. 
   * *Replace `<CORTEX_ROLE_ARN>` with the `CortexRoleArn` from Phase 1.*
   * *Replace `<REGION>` with your AWS Region.*
   * *Replace `<CENTRAL_BUCKET_NAME>` with your S3 bucket name.*

```json
{
    "Sid": "AllowCortexXSIAMToDecryptCloudTrailLogs",
    "Effect": "Allow",
    "Principal": {
        "AWS": "<CORTEX_ROLE_ARN>"
    },
    "Action": "kms:Decrypt",
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
