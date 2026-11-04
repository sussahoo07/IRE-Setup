# AWS Isolated Recovery Environment (IRE) — Test Plan

**Document Type:** Test Plan  
**Version:** 1.0  
**Date:** November 2026  
**Status:** Draft

---

## 1. Overview

This document defines the test strategy, test cases, and acceptance criteria for validating the AWS Isolated Recovery Environment (IRE). Tests are organized by domain: Identity, Network Isolation, Data Replication, Security Monitoring, SaaS Isolation, and Recovery Validation.

Tests should be executed:
- During initial deployment (acceptance testing)
- After any infrastructure change to IRE
- During scheduled semi-annual recovery exercises

---

## 2. Test Environments and Prerequisites

| Item | Detail |
|---|---|
| IRE-Core Account ID | `<IRE-Core-Account-ID>` |
| IRE-Security Account ID | `<IRE-Security-Account-ID>` |
| PROD Account ID | `<PROD-Account-ID>` |
| IRE Identity Center Portal URL | `https://<ire-instance>.awsapps.com/start` |
| Test Executor Role | `IRE-RecoveryAdmin` (via IRE Identity Center) |
| AWS CLI Profile | `ire-core` (configured with IRE Identity Center SSO) |
| Terraform/CDK State | Verified deployed before testing |

---

## 3. Test Summary

| Domain | Total Tests | Priority |
|---|---|---|
| Identity & Authentication | 8 | Critical |
| Network Isolation | 10 | Critical |
| SaaS Agent Isolation | 5 | Critical |
| Data Replication & Backup | 8 | High |
| Encryption (KMS) | 5 | High |
| Security Monitoring | 8 | High |
| Service Control Policies | 6 | Critical |
| Recovery Validation | 6 | High |
| **Total** | **56** | |

---

## 4. Identity & Authentication Tests

### TC-ID-01: IRE Identity Center Login — Local User
**Requirement:** FR-05, FR-07  
**Objective:** Verify a local IRE Identity Center user can log in without AD.  
**Steps:**
1. Navigate to the IRE Identity Center portal URL.
2. Enter local IRE username and password.
3. Complete MFA challenge (TOTP).
4. Verify access to `IRE-Core` account via the assigned permission set.  
**Expected Result:** Login succeeds; `IRE-Core` account appears in the account list.  
**Pass Criteria:** Successful login and account access with local credentials only.

---

### TC-ID-02: PROD AD Credentials Rejected in IRE
**Requirement:** FR-06, NFR-06  
**Objective:** Confirm PROD AD / PROD Identity Center credentials do NOT grant access to IRE.  
**Steps:**
1. Obtain a valid PROD IAM Identity Center SSO session token.
2. Attempt to assume an IRE account role using the PROD SSO token via AWS CLI.
3. Attempt to access the IRE Identity Center portal with PROD user credentials.  
**Expected Result:** All attempts fail with `AccessDenied` or authentication failure.  
**Pass Criteria:** No access to IRE accounts via PROD identity credentials.

---

### TC-ID-03: MFA Enforcement
**Requirement:** FR-08  
**Objective:** Verify MFA is required and cannot be bypassed.  
**Steps:**
1. Attempt to log in to IRE Identity Center portal with username/password only (no MFA).
2. Verify the portal demands MFA completion before granting access.  
**Expected Result:** Login blocked at MFA step; session not created without MFA.  
**Pass Criteria:** MFA challenge is mandatory; no session without MFA completion.

---

### TC-ID-04: Break-Glass Alert Fires on Login
**Requirement:** FR-11  
**Objective:** Confirm SNS alert fires when break-glass credentials are used.  
**Steps:**
1. Log in using the designated break-glass IRE Identity Center user.
2. Check the IRE-Security SNS topic for an alert within 5 minutes.
3. Verify the alert email is received by the IRE Recovery Admin distribution list.  
**Expected Result:** SNS alert received within 5 minutes of break-glass login.  
**Pass Criteria:** Alert contains username, timestamp, and account ID.

---

### TC-ID-05: Permission Set Scope — RecoveryAdmin
**Requirement:** FR-13  
**Objective:** Validate `IRE-RecoveryAdmin` has appropriate access in IRE-Core only.  
**Steps:**
1. Log in as a member of the `IRE Recovery Admins` group.
2. Verify access to EC2, S3, AWS Backup in `IRE-Core`.
3. Verify NO access to `IRE-Security` account resources via this permission set.  
**Expected Result:** Full access to IRE-Core; access denied in IRE-Security.  
**Pass Criteria:** Correct scoping as defined in permission set assignment.

---

### TC-ID-06: Permission Set Scope — ReadOnly
**Requirement:** FR-13  
**Objective:** Validate `IRE-ReadOnly` cannot make any changes.  
**Steps:**
1. Log in as a member of the `IRE Auditors` group.
2. Attempt to launch an EC2 instance in IRE-Core.
3. Attempt to delete an S3 object in IRE-Core.  
**Expected Result:** Both mutating operations return `AccessDenied`.  
**Pass Criteria:** Read operations succeed; all write/mutate operations denied.

---

### TC-ID-07: IRE Identity Center Has No AD Connector
**Requirement:** FR-06  
**Objective:** Confirm no external IdP is configured in IRE Identity Center.  
**Steps:**
1. Log in as `IRE-SecurityAdmin` to the IRE Identity Center management console.
2. Navigate to Settings → Identity Source.
3. Verify identity source is "Identity Center directory" (built-in), not "Active Directory" or "External identity provider".  
**Expected Result:** Identity source shows "Identity Center directory".  
**Pass Criteria:** No AD connector or SAML IdP configured.

---

### TC-ID-08: Credential Rotation Verification
**Requirement:** FR-12  
**Objective:** Verify IRE local user password was rotated post-exercise.  
**Steps:**
1. After a recovery exercise, retrieve the updated credentials from the physical vault.
2. Log in using the new credentials.
3. Confirm the pre-exercise credentials are no longer valid.  
**Expected Result:** Old credentials rejected; new credentials accepted.  
**Pass Criteria:** Authentication fails with old password; succeeds with new password.

---

## 5. Network Isolation Tests

### TC-NET-01: No Internet Gateway in IRE VPC
**Requirement:** FR-14  
**Objective:** Confirm no IGW is attached to the IRE VPC.  
**Steps:**
```bash
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=<IRE-VPC-ID>" \
  --profile ire-core
```
**Expected Result:** Empty `InternetGateways` list.  
**Pass Criteria:** Command returns `{ "InternetGateways": [] }`.

---

### TC-NET-02: No NAT Gateway in IRE VPC
**Requirement:** FR-15  
**Steps:**
```bash
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=<IRE-VPC-ID>" \
            "Name=state,Values=available" \
  --profile ire-core
```
**Expected Result:** Empty or no `available` NAT Gateways.  
**Pass Criteria:** `NatGateways` list is empty.

---

### TC-NET-03: No VPC Peering Connections
**Requirement:** FR-16  
**Steps:**
```bash
aws ec2 describe-vpc-peering-connections \
  --filters "Name=requester-vpc-info.vpc-id,Values=<IRE-VPC-ID>" \
  --profile ire-core
```
Also check accepter side:
```bash
aws ec2 describe-vpc-peering-connections \
  --filters "Name=accepter-vpc-info.vpc-id,Values=<IRE-VPC-ID>" \
  --profile ire-core
```
**Expected Result:** Empty `VpcPeeringConnections` list.  
**Pass Criteria:** No peering connections found in either direction.

---

### TC-NET-04: No Transit Gateway Attachments
**Requirement:** FR-17  
**Steps:**
```bash
aws ec2 describe-transit-gateway-vpc-attachments \
  --filters "Name=vpc-id,Values=<IRE-VPC-ID>" \
             "Name=state,Values=available,pending" \
  --profile ire-core
```
**Expected Result:** Empty `TransitGatewayVpcAttachments` list.  
**Pass Criteria:** No TGW attachments found.

---

### TC-NET-05: Route Tables Have No Internet Route
**Requirement:** FR-20  
**Steps:**
1. List all route tables in the IRE VPC.
2. For each route table, verify no route has destination `0.0.0.0/0` with a TGW, IGW, or NAT GW target.
```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<IRE-VPC-ID>" \
  --query "RouteTables[*].Routes[?DestinationCidrBlock=='0.0.0.0/0']" \
  --profile ire-core
```
**Expected Result:** All route tables return empty result for `0.0.0.0/0` destination.  
**Pass Criteria:** No default routes to internet exist in any IRE route table.

---

### TC-NET-06: EC2 Instance Cannot Reach Internet
**Requirement:** FR-14, FR-20  
**Objective:** Empirically confirm no internet path from IRE EC2 instance.  
**Steps:**
1. Launch a test EC2 instance in the IRE private subnet using SSM Session Manager.
2. Open SSM session and run:
```bash
curl --max-time 10 https://www.google.com
curl --max-time 10 https://api.crowdstrike.com
```
**Expected Result:** Both commands time out or return connection refused.  
**Pass Criteria:** No HTTP response from any external internet endpoint.

---

### TC-NET-07: VPC Endpoints Reachable from IRE
**Requirement:** FR-19  
**Objective:** Confirm AWS services are accessible via PrivateLink.  
**Steps:**
1. From an IRE EC2 instance (via SSM), run:
```bash
aws sts get-caller-identity --region <region>
aws s3 ls s3://ire-backup-data-<acct> --region <region>
aws secretsmanager list-secrets --region <region>
```
**Expected Result:** All commands succeed with valid responses.  
**Pass Criteria:** AWS service calls succeed without internet connectivity.

---

### TC-NET-08: SCP Blocks IGW Creation Attempt
**Requirement:** FR-14, SCP-01  
**Objective:** Verify SCP prevents IGW creation even by admin users.  
**Steps:**
1. As `IRE-RecoveryAdmin`, attempt to create an IGW:
```bash
aws ec2 create-internet-gateway --profile ire-core
```
**Expected Result:** `AccessDenied` error from SCP.  
**Pass Criteria:** Error message references an explicit deny.

---

### TC-NET-09: SCP Blocks TGW Attachment Attempt
**Requirement:** FR-17, SCP-01  
**Steps:**
1. As `IRE-RecoveryAdmin`, attempt to create a TGW VPC attachment:
```bash
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-<PROD-TGW-ID> \
  --vpc-id <IRE-VPC-ID> \
  --subnet-ids <IRE-Subnet-ID> \
  --profile ire-core
```
**Expected Result:** `AccessDenied` error.  
**Pass Criteria:** SCP deny is enforced.

---

### TC-NET-10: SCP Blocks NAT Gateway Creation Attempt
**Requirement:** FR-15, SCP-01  
**Steps:**
1. Attempt to create a NAT Gateway in the IRE VPC:
```bash
aws ec2 create-nat-gateway \
  --subnet-id <IRE-Subnet-ID> \
  --allocation-id <EIP-ID> \
  --profile ire-core
```
**Expected Result:** `AccessDenied` error.  
**Pass Criteria:** SCP deny is enforced.

---

## 6. SaaS Agent Isolation Tests

### TC-SAAS-01: No CrowdStrike Agent Running
**Requirement:** FR-23  
**Steps:**
1. On a restored EC2 instance in IRE, open an SSM session.
2. Check for Falcon agent process:
```bash
# Linux
ps aux | grep falcon
systemctl status falcon-sensor

# Windows
Get-Service -Name CSFalconService -ErrorAction SilentlyContinue
```
**Expected Result:** No Falcon process or service found.  
**Pass Criteria:** CrowdStrike Falcon is not installed or running.

---

### TC-SAAS-02: No Qualys Agent Running
**Requirement:** FR-24  
**Steps:**
1. On a restored EC2 instance in IRE, via SSM session:
```bash
# Linux
ps aux | grep qualys
systemctl status qualys-cloud-agent

# Windows
Get-Service -Name "QualysAgent" -ErrorAction SilentlyContinue
```
**Expected Result:** No Qualys agent process or service.  
**Pass Criteria:** Qualys is not installed or running.

---

### TC-SAAS-03: No Tanium Agent Running
**Requirement:** FR-25  
**Steps:**
```bash
# Linux
ps aux | grep tanium

# Windows
Get-Service -Name "Tanium Client" -ErrorAction SilentlyContinue
```
**Expected Result:** No Tanium client process or service.  
**Pass Criteria:** Tanium is not installed or running.

---

### TC-SAAS-04: No Rubrik Agent Attempting Callbacks
**Requirement:** FR-22  
**Steps:**
1. Enable VPC Flow Logs and monitor for outbound traffic to known Rubrik IP ranges or DNS names.
2. After restoring an EC2 instance from PROD AMI, check flow logs for any outbound connection attempts:
```bash
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs/ire \
  --filter-pattern "[version, account, eni, src, dst, srcport, dstport, protocol, packets, bytes, start, end, action=REJECT, status]" \
  --profile ire-security
```
**Expected Result:** No successful outbound connections to Rubrik endpoints; any attempts are REJECT in flow logs.  
**Pass Criteria:** All SaaS callback attempts are blocked and appear as REJECT in VPC Flow Logs.

---

### TC-SAAS-05: Security Group Denies Non-PrivateLink Outbound
**Requirement:** FR-21  
**Steps:**
1. Review Security Group outbound rules on IRE workload instances:
```bash
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=sg-ire-workloads" \
  --query "SecurityGroups[*].IpPermissionsEgress" \
  --profile ire-core
```
**Expected Result:** Outbound rules allow only HTTPS to the endpoint subnet CIDRs or AWS prefix lists; no `0.0.0.0/0` allow.  
**Pass Criteria:** No unrestricted outbound allow rule present.

---

## 7. Data Replication & Backup Tests

### TC-DATA-01: S3 Replication Delivers Data to IRE
**Requirement:** FR-27, FR-28  
**Steps:**
1. Upload a test object to the PROD source S3 bucket.
2. Wait up to 15 minutes.
3. Verify the object appears in the IRE destination bucket:
```bash
aws s3 ls s3://ire-backup-data-<acct-id>/test-object \
  --profile ire-core
```
**Expected Result:** Object present in IRE bucket.  
**Pass Criteria:** Replication completes; object encrypted with `ire-s3-cmk`.

---

### TC-DATA-02: Replicated Object Encrypted with IRE KMS CMK
**Requirement:** FR-30  
**Steps:**
1. Retrieve object metadata from IRE S3 bucket:
```bash
aws s3api head-object \
  --bucket ire-backup-data-<acct-id> \
  --key test-object \
  --profile ire-core
```
2. Verify `ServerSideEncryption: aws:kms` and `SSEKMSKeyId` matches `ire-s3-cmk` ARN.  
**Pass Criteria:** Object is encrypted with the IRE-owned CMK, not PROD CMK.

---

### TC-DATA-03: S3 Object Lock Prevents Deletion
**Requirement:** FR-29  
**Steps:**
1. Attempt to delete a replicated object in the IRE S3 bucket:
```bash
aws s3api delete-object \
  --bucket ire-backup-data-<acct-id> \
  --key test-object \
  --profile ire-core
```
**Expected Result:** Delete rejected due to Object Lock.  
**Pass Criteria:** Error: `Object is WORM protected and cannot be overwritten`.

---

### TC-DATA-04: AWS Backup Cross-Account Copy Succeeds
**Requirement:** FR-28  
**Steps:**
1. Trigger a manual backup copy job from PROD to IRE vault.
2. Verify backup recovery point appears in IRE backup vault:
```bash
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name ire-backup-vault \
  --profile ire-core
```
**Expected Result:** Recovery point visible in IRE vault.  
**Pass Criteria:** Recovery point status is `COMPLETED`.

---

### TC-DATA-05: Backup Vault Lock Prevents Deletion
**Requirement:** FR-32  
**Steps:**
1. Attempt to delete a recovery point from the IRE backup vault:
```bash
aws backup delete-recovery-point \
  --backup-vault-name ire-backup-vault \
  --recovery-point-arn <ARN> \
  --profile ire-core
```
**Expected Result:** Delete rejected due to Vault Lock.  
**Pass Criteria:** Operation fails with vault lock enforcement error.

---

### TC-DATA-06: PROD Cannot Access IRE KMS CMK
**Requirement:** FR-31, NFR-03  
**Steps:**
1. From PROD account, attempt to use the IRE KMS CMK:
```bash
aws kms describe-key \
  --key-id <ire-s3-cmk-arn> \
  --profile prod
```
**Expected Result:** `AccessDeniedException`.  
**Pass Criteria:** PROD account explicitly denied by key policy.

---

### TC-DATA-07: AMI Re-encryption with IRE CMK After Copy
**Requirement:** FR-33  
**Steps:**
1. After AMI copy into IRE-Core, verify the AMI block device mappings use the IRE KMS CMK:
```bash
aws ec2 describe-images \
  --image-ids <IRE-AMI-ID> \
  --query "Images[*].BlockDeviceMappings[*].Ebs.KmsKeyId" \
  --profile ire-core
```
**Expected Result:** KMS Key ID matches `ire-ebs-cmk` ARN.  
**Pass Criteria:** AMI uses IRE-owned CMK; no reference to PROD CMK.

---

### TC-DATA-08: PROD AMI Share Revoked After Copy
**Requirement:** FR-33  
**Steps:**
1. From IRE-Core account, verify the PROD-shared AMI is no longer visible:
```bash
aws ec2 describe-images \
  --executable-users self \
  --filters "Name=image-id,Values=<PROD-AMI-ID>" \
  --profile ire-core
```
**Expected Result:** AMI not visible or not launchable from IRE-Core.  
**Pass Criteria:** PROD AMI share has been revoked; only the copied IRE AMI is available.

---

## 8. Encryption (KMS) Tests

### TC-KMS-01: All IRE EBS Volumes Encrypted
**Requirement:** NFR-01  
**Steps:**
```bash
aws ec2 describe-volumes \
  --filters "Name=encrypted,Values=false" \
  --profile ire-core
```
**Expected Result:** Empty `Volumes` list (no unencrypted volumes).  
**Pass Criteria:** All EBS volumes are encrypted.

---

### TC-KMS-02: KMS Key Rotation Enabled
**Requirement:** NFR-04  
**Steps:**
```bash
aws kms get-key-rotation-status \
  --key-id <ire-ebs-cmk-arn> \
  --profile ire-core
```
Repeat for all IRE CMKs.  
**Expected Result:** `KeyRotationEnabled: true` for all CMKs.  
**Pass Criteria:** Rotation enabled on every IRE CMK.

---

### TC-KMS-03: IRE CMK Key Policy Denies PROD
**Requirement:** NFR-03  
**Steps:**
1. View the key policy for `ire-s3-cmk`:
```bash
aws kms get-key-policy \
  --key-id <ire-s3-cmk-arn> \
  --policy-name default \
  --profile ire-security
```
2. Verify presence of a `Deny` statement for PROD account ID.  
**Pass Criteria:** Explicit `Deny` for PROD account principal exists in key policy.

---

### TC-KMS-04: CloudTrail Logs Encrypted with IRE Log CMK
**Requirement:** NFR-01  
**Steps:**
```bash
aws cloudtrail describe-trails \
  --trail-name-list IRE-OrganizationTrail \
  --profile ire-security \
  --query "trailList[*].KMSKeyId"
```
**Expected Result:** Returns `ire-log-cmk` ARN.  
**Pass Criteria:** Trail encryption uses IRE-owned log CMK.

---

### TC-KMS-05: Secrets Manager Secrets Use IRE CMK
**Requirement:** NFR-01  
**Steps:**
```bash
aws secretsmanager list-secrets \
  --query "SecretList[*].{Name:Name,KmsKeyId:KmsKeyId}" \
  --profile ire-core
```
**Expected Result:** All secrets reference `ire-secrets-cmk` ARN.  
**Pass Criteria:** No secrets using default AWS managed key.

---

## 9. Security Monitoring Tests

### TC-MON-01: GuardDuty Enabled in IRE Accounts
**Requirement:** FR-34  
**Steps:**
```bash
aws guardduty list-detectors --profile ire-core
aws guardduty list-detectors --profile ire-security
```
**Expected Result:** Active detector ID returned for both accounts.  
**Pass Criteria:** GuardDuty detectors exist and are in `ENABLED` state.

---

### TC-MON-02: Security Hub Enabled and Standards Active
**Requirement:** FR-35  
**Steps:**
```bash
aws securityhub describe-hub --profile ire-security
aws securityhub describe-standards-subscriptions --profile ire-security
```
**Expected Result:** Hub status `ACTIVE`; CIS and FSBP standards show `READY`.  
**Pass Criteria:** Both standards enabled and active in IRE-Security.

---

### TC-MON-03: CloudTrail Active and Logging
**Requirement:** FR-36  
**Steps:**
```bash
aws cloudtrail get-trail-status \
  --name IRE-OrganizationTrail \
  --profile ire-security
```
**Expected Result:** `IsLogging: true`.  
**Pass Criteria:** Trail is active and logging.

---

### TC-MON-04: CloudTrail Log Bucket Object Lock Active
**Requirement:** FR-37  
**Steps:**
```bash
aws s3api get-object-lock-configuration \
  --bucket ire-cloudtrail-logs-<acct-id> \
  --profile ire-security
```
**Expected Result:** Object Lock enabled with retention mode `GOVERNANCE` or `COMPLIANCE`.  
**Pass Criteria:** Object Lock is configured.

---

### TC-MON-05: VPC Flow Logs Enabled
**Requirement:** FR-41  
**Steps:**
```bash
aws ec2 describe-flow-logs \
  --filter "Name=resource-id,Values=<IRE-VPC-ID>" \
  --profile ire-core
```
**Expected Result:** Flow log with status `ACTIVE` delivering to CloudWatch Logs.  
**Pass Criteria:** Flow log is active for the IRE VPC.

---

### TC-MON-06: Inspector v2 Enabled
**Requirement:** FR-39  
**Steps:**
```bash
aws inspector2 describe-organization-configuration \
  --profile ire-security
```
Also check account status:
```bash
aws inspector2 batch-get-account-status \
  --account-ids <IRE-Core-Account-ID> \
  --profile ire-security
```
**Expected Result:** EC2 scanning status `ENABLED` for IRE-Core.  
**Pass Criteria:** Inspector v2 active for EC2 and ECR in IRE.

---

### TC-MON-07: SCP Blocks GuardDuty Disablement
**Requirement:** FR-34, SCP-02  
**Steps:**
1. As `IRE-RecoveryAdmin`, attempt to delete the GuardDuty detector:
```bash
aws guardduty delete-detector \
  --detector-id <DETECTOR-ID> \
  --profile ire-core
```
**Expected Result:** `AccessDenied` from SCP.  
**Pass Criteria:** SCP deny enforced; GuardDuty remains active.

---

### TC-MON-08: SCP Blocks CloudTrail Disablement
**Requirement:** FR-36, SCP-02  
**Steps:**
```bash
aws cloudtrail stop-logging \
  --name IRE-OrganizationTrail \
  --profile ire-core
```
**Expected Result:** `AccessDenied` from SCP.  
**Pass Criteria:** SCP deny enforced; CloudTrail remains active.

---

## 10. Service Control Policy Tests

### TC-SCP-01: SCP Inventory Attached to IRE OU
**Requirement:** FR-04  
**Steps:**
1. From Management Account, list SCPs attached to the IRE OU:
```bash
aws organizations list-policies-for-target \
  --target-id <IRE-OU-ID> \
  --filter SERVICE_CONTROL_POLICY \
  --profile mgmt
```
**Expected Result:** SCPs `IRE-DenyNetworkExposure`, `IRE-DenySecurityDisablement`, `IRE-DenyIdentityMutation` are listed.  
**Pass Criteria:** All three SCPs attached to IRE OU.

---

### TC-SCP-02: SCP — Deny IGW (covered in TC-NET-08)

### TC-SCP-03: SCP — Deny TGW (covered in TC-NET-09)

### TC-SCP-04: SCP — Deny GuardDuty Disable (covered in TC-MON-07)

### TC-SCP-05: SCP — Deny CloudTrail Disable (covered in TC-MON-08)

### TC-SCP-06: SCP — Deny Identity Center User Deletion
**Requirement:** SCP-03  
**Steps:**
1. As a non-SecurityAdmin user, attempt to disable an IRE Identity Center user:
```bash
aws identitystore delete-user \
  --identity-store-id <STORE-ID> \
  --user-id <USER-ID> \
  --profile ire-core
```
**Expected Result:** `AccessDenied` from SCP.  
**Pass Criteria:** Only `IRE-SecurityAdmin` in IRE-Security account can modify Identity Center users.

---

## 11. Recovery Validation Tests

### TC-REC-01: EC2 Instance Restore from AWS Backup
**Requirement:** FR-42  
**Steps:**
1. Identify a recovery point in `ire-backup-vault`.
2. Initiate restore job:
```bash
aws backup start-restore-job \
  --recovery-point-arn <RECOVERY-POINT-ARN> \
  --metadata '{"subnetId":"<IRE-Subnet-ID>","instanceType":"t3.medium"}' \
  --iam-role-arn arn:aws:iam::<IRE-Core>:role/IRE-BackupRestoreRole \
  --profile ire-core
```
3. Monitor restore job to completion.
4. Verify instance is running in IRE VPC.  
**Expected Result:** EC2 instance launched in IRE private subnet; EBS encrypted with `ire-ebs-cmk`.  
**Pass Criteria:** Restore job status `COMPLETED`; instance in `running` state in correct subnet.

---

### TC-REC-02: Restored Instance Accessible via SSM (No SSH Required)
**Requirement:** FR-43  
**Steps:**
1. After TC-REC-01, start an SSM session to the restored instance:
```bash
aws ssm start-session \
  --target <INSTANCE-ID> \
  --profile ire-core
```
**Expected Result:** SSM session established successfully without SSH or bastion host.  
**Pass Criteria:** SSM session active via PrivateLink endpoint.

---

### TC-REC-03: Validation Playbook Executed via SSM Run Command
**Requirement:** FR-43  
**Steps:**
1. Execute the validation playbook stored in IRE S3:
```bash
aws ssm send-command \
  --instance-ids <INSTANCE-ID> \
  --document-name "AWS-RunShellScript" \
  --parameters '{"commands":["aws s3 cp s3://ire-validation-scripts/validate.sh . && bash validate.sh"]}' \
  --output-s3-bucket-name ire-validation-results \
  --profile ire-core
```
2. Verify output written to IRE S3.  
**Expected Result:** Validation script runs successfully; output written to S3.  
**Pass Criteria:** Command status `Success`; output present in S3.

---

### TC-REC-04: DNS Resolution Works Within IRE VPC
**Requirement:** FR-44  
**Steps:**
1. From a restored EC2 instance via SSM session, resolve an internal hostname defined in Route 53 PHZ:
```bash
nslookup recovery-app.ire.internal
```
**Expected Result:** Hostname resolves to an IP within the IRE VPC CIDR.  
**Pass Criteria:** DNS resolution succeeds using Route 53 Private Hosted Zone; no PROD DNS dependency.

---

### TC-REC-05: Validation Results Logged to CloudWatch
**Requirement:** FR-45  
**Steps:**
1. After TC-REC-03, check CloudWatch Logs in IRE-Security for validation output:
```bash
aws logs filter-log-events \
  --log-group-name /ire/recovery-validation \
  --profile ire-security
```
**Expected Result:** Validation results present in log stream.  
**Pass Criteria:** Log entries exist with timestamps and validation output.

---

### TC-REC-06: Evidence Archive Created in IRE-Security S3
**Requirement:** FR-46  
**Steps:**
1. After recovery exercise, run the evidence packaging script:
```bash
aws ssm send-command \
  --document-name "IRE-PackageEvidence" \
  --profile ire-security
```
2. Verify archive created in `s3://ire-audit-evidence-<acct>/<exercise-date>/`.  
**Expected Result:** Archive contains CloudTrail extract, VPC Flow Logs, SSM Run Command output.  
**Pass Criteria:** S3 archive present and non-empty; encrypted with `ire-log-cmk`.

---

## 12. Test Execution Record

| Test Case | Status | Executed By | Date | Pass/Fail | Notes |
|---|---|---|---|---|---|
| TC-ID-01 | Pending | | | | |
| TC-ID-02 | Pending | | | | |
| TC-ID-03 | Pending | | | | |
| TC-ID-04 | Pending | | | | |
| TC-ID-05 | Pending | | | | |
| TC-ID-06 | Pending | | | | |
| TC-ID-07 | Pending | | | | |
| TC-ID-08 | Pending | | | | |
| TC-NET-01 | Pending | | | | |
| TC-NET-02 | Pending | | | | |
| TC-NET-03 | Pending | | | | |
| TC-NET-04 | Pending | | | | |
| TC-NET-05 | Pending | | | | |
| TC-NET-06 | Pending | | | | |
| TC-NET-07 | Pending | | | | |
| TC-NET-08 | Pending | | | | |
| TC-NET-09 | Pending | | | | |
| TC-NET-10 | Pending | | | | |
| TC-SAAS-01 | Pending | | | | |
| TC-SAAS-02 | Pending | | | | |
| TC-SAAS-03 | Pending | | | | |
| TC-SAAS-04 | Pending | | | | |
| TC-SAAS-05 | Pending | | | | |
| TC-DATA-01 | Pending | | | | |
| TC-DATA-02 | Pending | | | | |
| TC-DATA-03 | Pending | | | | |
| TC-DATA-04 | Pending | | | | |
| TC-DATA-05 | Pending | | | | |
| TC-DATA-06 | Pending | | | | |
| TC-DATA-07 | Pending | | | | |
| TC-DATA-08 | Pending | | | | |
| TC-KMS-01 | Pending | | | | |
| TC-KMS-02 | Pending | | | | |
| TC-KMS-03 | Pending | | | | |
| TC-KMS-04 | Pending | | | | |
| TC-KMS-05 | Pending | | | | |
| TC-MON-01 | Pending | | | | |
| TC-MON-02 | Pending | | | | |
| TC-MON-03 | Pending | | | | |
| TC-MON-04 | Pending | | | | |
| TC-MON-05 | Pending | | | | |
| TC-MON-06 | Pending | | | | |
| TC-MON-07 | Pending | | | | |
| TC-MON-08 | Pending | | | | |
| TC-SCP-01 | Pending | | | | |
| TC-SCP-06 | Pending | | | | |
| TC-REC-01 | Pending | | | | |
| TC-REC-02 | Pending | | | | |
| TC-REC-03 | Pending | | | | |
| TC-REC-04 | Pending | | | | |
| TC-REC-05 | Pending | | | | |
| TC-REC-06 | Pending | | | | |

---

## 13. Acceptance Criteria for Go-Live

All **Critical** test cases must pass before IRE is considered operational. High-priority tests must pass before the first scheduled recovery exercise.

| Priority | Required Pass Rate | Test Cases |
|---|---|---|
| Critical | 100% | TC-ID-01/02/03, TC-NET-01 through 10, TC-SAAS-01 through 05, TC-SCP-01/06 |
| High | 100% | TC-DATA-01 through 08, TC-KMS-01 through 05, TC-MON-01 through 08, TC-REC-01 through 06 |

Any failed Critical test case blocks IRE go-live. A failed High test case requires a documented remediation plan with a target date before go-live.
