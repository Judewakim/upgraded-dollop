# Security Stack — Single-Account Production Deployment

Files:
- `securitystack-singleaccount-production.yaml` — CloudFormation template (single-account).
- `README-securitystack.md` — this README.

## Purpose
Deploy a production-ready single-account AWS Config environment:
- Central S3 bucket for AWS Config and conformance pack artifacts.
- Uploads a snapshot YAML of "AWS Security Best Practices" conformance pack into S3.
- Enables AWS Config recorder (records all supported and global resources).
- DeliveryChannel writes to S3 and publishes notifications to an SNS topic.
- SNS topic sends an email subscription confirmation; after confirmation, a scheduled reporter Lambda will publish conformance summaries to the SNS topic.
- Lifecycle rules and hardened IAM policies are included.

## High-level flow
1. CloudFormation creates central S3, uploader Lambda (custom resource), SNS topic, ConfigurationRecorder, DeliveryChannel, ConformancePack (pointing to uploaded S3 object), reporter Lambda, and custom start-recorder Lambda.
2. Uploader Lambda (custom resource) writes the embedded conformance pack YAML to: `s3://<CentralBucketName>/<ConformancePackS3Key>`.
3. DeliveryChannel is created and points to the central bucket and SNS topic.
4. Start-recorder Lambda starts the recorder so Config begins recording.
5. SNS sends subscription confirmation email to the `EmailAddress` parameter; user must confirm subscription.
6. Reporter Lambda runs daily, checks that the email subscription is confirmed, and publishes the conformance pack summary to the SNS topic (which emails subscribers).

## Parameters
- `EmailAddress` (REQUIRED): email to subscribe to the SNS topic. Recipient must confirm the SNS subscription.
- `CentralBucketName` (optional): S3 bucket name; must be globally unique. Default uses account/region.
- `ConformancePackS3Key` (optional): S3 key path to upload conformance pack (default `conformance-packs/security-best-practices.yaml`).
- `ConformancePackName` (optional): Conformance pack name (default `SecurityBestPractices-Custom`).

## Key production decisions & safeguards
- **SSE-S3 (AWS-managed)** encryption used (no KMS key created).
- **Versioning** enabled on the bucket.
- **Public access blocked** by `PublicAccessBlockConfiguration`.
- **Bucket lifecycle rules**: transition to STANDARD_IA after 30 days, GLACIER after 365 days, expire objects after ~7 years.
- **Bucket DeletionPolicy: Retain** to prevent accidental deletion of data.
- **BucketPolicy** restricts Config PutObject to requests from this account (no cross-region/cross-account write permission).
- **IAM policies hardened**:
  - Uploader role is scoped to the exact S3 keys used.
  - Reporter role is scoped to the specific SNS topic and conformance-pack ARN where possible.
  - Start-recorder role is scoped to config resources in this account/region.
  - Note: some AWS Config API calls may not support narrow resource-level policies in all regions/APIs; we've scoped where supported and limited `"*"` usage.

## Manual steps required (Before / During / After deployment)
**Before deployment**
1. Choose the AWS region to deploy (Config is regional). The stack will create a recorder in that region.
2. If you will reuse the bucket name, ensure it is globally unique (or accept the default).
3. Ensure the deploying IAM principal has adequate permissions to create IAM roles, Lambdas, S3, SNS, CloudWatch Events, AWS Config resources, and CloudFormation stacks. Administrator-level privileges simplify deployment.
   - Minimum deployment permissions (recommended to use admin for initial deploy): IAM (create roles/policies), Lambda, S3, SNS, Events, Config, CloudFormation.
4. Confirm organization / StackSet not required — this is single-account.

**During deployment**
1. Deploy CloudFormation stack in the chosen region (Management console, AWS CLI, or IaC pipeline).
2. Watch the stack events for successful creation of:
   - `CentralConfigBucket`
   - `ConformancePackUploaderCustom` (uploader ran and uploaded the YAML)
   - `ConfigDeliveryChannel`
   - `SecurityBestPracticesConformancePack`
   - `StartRecorderCustom` (recorder started)
   - `ConfigNotificationsTopic`
3. **SNS confirmation email**: shortly after stack creation the supplied `EmailAddress` will receive an SNS confirmation email. The recipient must click the confirmation link. The subscription will remain in `PendingConfirmation` until confirmed — the reporter will NOT publish reports until confirmed.

**After deployment**
1. Confirm the SNS subscription:
   - Confirm the email subscription via the link in the SNS confirmation message.
   - In the AWS console, navigate to SNS -> Topics -> `<ConfigNotificationsTopic>` -> Subscriptions to verify the subscription state is `Confirmed`.
2. Verify Config is recording:
   - Console -> AWS Config -> Resource inventory.
   - Ensure `Configuration recorder` is `Started`.
3. Verify conformance pack evaluation:
   - Console -> AWS Config -> Conformance packs -> `SecurityBestPractices-Custom`
   - Evaluation may take some minutes to hours for initial results depending on resources.
4. Verify reporter runs:
   - Reporter is scheduled daily (rate(1 day)). You can manually trigger the Lambda from the console to test publishing once subscription is confirmed.
   - Verify the SNS email delivery receives the report.

## Troubleshooting & common issues
- **No SNS email received**:
  - Check spam folder. Confirm subscription.
  - Confirm `ConfigNotificationsTopic` exists and its subscription shows `Confirmed` in SNS console.
- **Conformance pack shows no results**:
  - Initial evaluation can take time. Check AWS Config console Conformance pack details.
  - Check CloudWatch Logs for `ConformancePackEvaluation` (if created).
- **Reporter doesn't publish**:
  - Ensure SNS subscription is `Confirmed`.
  - Inspect CloudWatch Logs for the reporter Lambda to see error messages.
  - Ensure reporter IAM role allowed `config:GetConformancePackComplianceSummary` and `sns:Publish` on the topic.
- **Uploader custom resource failed**:
  - Check CloudWatch Logs for `ConformancePackUploaderFunction`.
  - Confirm uploader role has PutObject permissions to the specified S3 key.
- **Recorder not started**:
  - The custom resource calls `config:start-configuration-recorder`. Review CloudWatch Logs of `StartRecorderFunction` for errors.

## Security considerations & next-hardening steps
- Consider moving from SSE-S3 to SSE-KMS with a customer-managed KMS key for stronger access controls and auditability.
- Narrow IAM role names & policies further to meet corporate least-privilege rules.
- Replace embedded conformance pack snapshot with the official AWS-managed conformance pack YAML (keep under source control and versioned).
- Add CloudTrail+CloudWatch alarms for suspicious activity (e.g., bucket policy changes).
- Add object lock / MFA delete if required by retention policy (note: S3 Object Lock adds additional constraints).

## Maintenance & lifecycle
- Lifecycle rules will move older data to cheaper storage classes and eventually expire objects.
- The bucket is set to `Retain` on stack deletion; to remove the bucket, manually empty and then delete or change the stack's deletion policy.

## If you want me to do any of the following now:
- Replace the **embedded conformance pack snapshot** with the official AWS managed "Security Best Practices" pack (I can fetch and embed it).
- Switch encryption from SSE-S3 to SSE-KMS and create a CMK with a secure key policy.
- Add automated verification of SNS confirmation by sending a test message and reporting confirmation status to CloudWatch events.
- Add CloudWatch alarms for failures in custom resources or lambdas.

