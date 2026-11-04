# AWS Isolated Recovery Environment (IRE) — Solution Design

**Document Type:** Technical Design Document  
**Version:** 1.0  
**Date:** November 2026  
**Status:** Draft

---

## 1. Executive Summary

This document describes the detailed technical design for the AWS Isolated Recovery Environment (IRE). The IRE is a clean-room AWS environment within the existing AWS Organization that provides independent recovery capability in the event of a compromise of Production (PROD) or Disaster Recovery (DR) environments. It is isolated at the identity, network, and security tooling layers.

---

## 2. Architecture Overview

### 2.1 AWS Organization Layout

```
AWS Organization (Management Account)
├── Root
│   ├── OU: Production
│   │   └── Account: PROD
│   │         Identity: IAM Identity Center → Microsoft AD (federated)
│   │         Network:  TGW, IGW, NAT GW
│   │         SaaS:     Rubrik, CrowdStrike, Qualys, Tanium
│   │
│   ├── OU: DR
│   │   └── Account: DR
│   │         Identity: IAM Identity Center → Microsoft AD (federated)
│   │         Network:  TGW attachment to PROD TGW
│   │
│   └── OU: IRE                          ◄── New OU
│       ├── Account: IRE-Core            (recovery workloads, validation)
│       └── Account: IRE-Security        (logging, monitoring, alerting)
```

### 2.2 Component Map

| Component | Service | Account | Purpose |
|---|---|---|---|
| Recovery Administrator Identity | IAM Identity Center (local) | IRE OU | Independent authentication |
| Recovery VPC | Amazon VPC | IRE-Core | Isolated network boundary |
| VPC Endpoints | AWS PrivateLink | IRE-Core | AWS service access without internet |
| Backup Vault | AWS Backup | IRE-Core | Immutable recovery point storage |
| Replicated Data | S3 + Object Lock | IRE-Core | Backup data from PROD |
| Encryption Keys | AWS KMS CMKs | IRE-Core, IRE-Security | IRE-exclusive data encryption |
| Threat Detection | Amazon GuardDuty | IRE-Security | Replace CrowdStrike |
| Vulnerability Scanning | Amazon Inspector v2 | IRE-Core | Replace Qualys |
| Patch Management | AWS Systems Manager | IRE-Core | Replace Tanium |
| Audit Logging | AWS CloudTrail | IRE-Security | Independent audit trail |
| Configuration Compliance | AWS Config | IRE-Security | Isolation rule enforcement |
| Security Aggregation | AWS Security Hub | IRE-Security | Independent security posture |
| Alerting | Amazon SNS | IRE-Security | Decoupled notifications |

---

## 3. Identity and Authentication Design

### 3.1 IAM Identity Center — Standalone Instance

A separate IAM Identity Center instance is enabled and delegated to the `IRE-Security` account. This instance is completely independent of the PROD IAM Identity Center instance.

```
PROD IAM Identity Center                IRE IAM Identity Center
─────────────────────────               ──────────────────────────
Identity Source: Microsoft AD           Identity Source: Built-in
Federation: SAML 2.0 / SCIM            Federation: NONE
Users: AD-synced                        Users: Manually provisioned
MFA: AD-controlled                      MFA: TOTP enforced (hardware preferred)
Admin: PROD Cloud Platform team         Admin: IRE-Security account role only
```

### 3.2 IAM Identity Center Permission Sets

| Permission Set | Policies | Assigned Accounts | Assigned Group |
|---|---|---|---|
| `IRE-RecoveryAdmin` | `AdministratorAccess` | IRE-Core | IRE Recovery Admins |
| `IRE-SecurityAdmin` | `AdministratorAccess` | IRE-Security | IRE Security Admins |
| `IRE-ReadOnly` | `ReadOnlyAccess` | IRE-Core, IRE-Security | IRE Auditors |
| `IRE-NetworkAdmin` | Custom: EC2/VPC/Route53 | IRE-Core | IRE Network Team |

### 3.3 Break-Glass Procedure

```
Step 1: CISO or delegate authorizes IRE activation
Step 2: Two designated custodians retrieve physical envelope from vault
Step 3: IAM Identity Center local user credentials extracted
Step 4: Login to IRE Identity Center portal (separate URL)
Step 5: TOTP hardware token used for MFA
Step 6: SNS alert fires automatically upon console login (CloudTrail → EventBridge → SNS)
Step 7: Post-use: credentials rotated immediately
```

### 3.4 IAM Role Design (IRE-Core)

```
IRE-Core Account IAM Roles (assumed via IAM Identity Center permission sets):

arn:aws:iam::<IRE-Core>:role/IRE-RecoveryAdmin-Role
  Trust: IAM Identity Center SAML provider (IRE instance only)
  Permissions: AdministratorAccess

arn:aws:iam::<IRE-Core>:role/IRE-BackupRestoreRole
  Trust: AWS Backup service principal
  Permissions: Backup restore, EC2 AMI launch, EBS snapshot restore

arn:aws:iam::<IRE-Core>:role/IRE-SSMInstanceRole
  Trust: EC2 instance profile
  Permissions: SSM core + CloudWatch Logs write
```

---

## 4. Network Design

### 4.1 IRE VPC Layout

```
Region: ap-southeast-1 (or as designated)

IRE VPC: 10.200.0.0/16
├── Subnet: ire-private-a  10.200.1.0/24  AZ-a
├── Subnet: ire-private-b  10.200.2.0/24  AZ-b
└── Subnet: ire-endpoint-a 10.200.3.0/24  AZ-a  (VPC Endpoint ENIs)
    Subnet: ire-endpoint-b 10.200.4.0/24  AZ-b  (VPC Endpoint ENIs)

Route Tables:
  ire-private-rt:
    Destination: 10.200.0.0/16  Target: local
    (NO default route — no IGW, no NAT GW, no TGW)
```

### 4.2 VPC Endpoints Specification

| Endpoint | Type | Service | Used By |
|---|---|---|---|
| `vpce-s3` | Gateway | com.amazonaws.region.s3 | All (S3 access) |
| `vpce-ssm` | Interface | com.amazonaws.region.ssm | EC2 instances (SSM) |
| `vpce-ssmmessages` | Interface | com.amazonaws.region.ssmmessages | EC2 instances (Session Manager) |
| `vpce-ec2messages` | Interface | com.amazonaws.region.ec2messages | EC2 instances |
| `vpce-ec2` | Interface | com.amazonaws.region.ec2 | Recovery operations |
| `vpce-kms` | Interface | com.amazonaws.region.kms | Encryption/decryption |
| `vpce-secretsmanager` | Interface | com.amazonaws.region.secretsmanager | Secrets retrieval |
| `vpce-logs` | Interface | com.amazonaws.region.logs | CloudWatch Logs |
| `vpce-sts` | Interface | com.amazonaws.region.sts | Role assumption |
| `vpce-backup` | Interface | com.amazonaws.region.backup | AWS Backup operations |
| `vpce-backup-gateway` | Interface | com.amazonaws.region.backup-gateway | Backup gateway |

### 4.3 Security Group Rules

**sg-ire-endpoints** (attached to VPC endpoint ENIs):
```
Inbound:
  Allow HTTPS (443) from 10.200.0.0/16

Outbound:
  Deny All
```

**sg-ire-workloads** (attached to recovery EC2 instances):
```
Inbound:
  Allow all from 10.200.0.0/16 (intra-VPC)

Outbound:
  Allow HTTPS (443) to 10.200.3.0/24, 10.200.4.0/24 (endpoint subnets)
  Deny All other outbound
```

### 4.4 Network Isolation Enforcement (SCP)

The following actions are denied at the OU level via SCP `IRE-DenyNetworkExposure`:

```json
Denied Actions:
  ec2:AttachInternetGateway
  ec2:CreateInternetGateway
  ec2:CreateNatGateway
  ec2:CreateVpcPeeringConnection
  ec2:AcceptVpcPeeringConnection
  ec2:CreateTransitGatewayVpcAttachment
  ec2:AcceptTransitGatewayVpcAttachment
  ram:AcceptResourceShareInvitation  (from non-IRE accounts)
```

---

## 5. Data Replication Design

### 5.1 S3 Cross-Account Replication

```
PROD Account                              IRE-Core Account
─────────────────────────────────────     ────────────────────────────────────
S3 Bucket: prod-backup-data               S3 Bucket: ire-backup-data-<acct-id>
  Replication Rule:                         Bucket Policy:
    Destination: ire-backup-data             - Allow s3:PutObject from
    Encryption: IRE KMS CMK                    arn:aws:iam::PROD:role/BackupRole
    Storage Class: S3 Standard-IA             - Deny s3:DeleteObject from *
                                              - Deny s3:PutObject from non-PROD
                                            Object Lock: Governance mode
                                            Retention: 365 days minimum
                                            Encryption: ire-s3-cmk (KMS)
```

### 5.2 AWS Backup Cross-Account Copy

```
Management Account — Backup Policy
  Source Vault:  PROD Backup Vault
  Destination:   ire-backup-vault (IRE-Core account)
  Copy Role:     arn:aws:iam::PROD:role/BackupCrossAcctRole
  Schedule:      Daily (aligned to PROD backup window)

IRE-Core Account — Backup Vault
  Vault Name:    ire-backup-vault
  Encryption:    ire-backup-cmk (KMS)
  Vault Lock:    Compliance mode, min 365 days
  Access Policy: Deny all delete actions from all principals
```

### 5.3 AMI Replication

```
PROD Account
  1. Create AMI from EC2 instance
  2. Share AMI with IRE-Core account (temporary)

IRE-Core Account
  3. Copy AMI: ec2:CopyImage (re-encrypts with ire-ebs-cmk)
  4. Verify copy complete

PROD Account
  5. Revoke AMI share (ec2:ModifyImageAttribute)

IRE-Core Account owns final AMI — PROD cannot modify or delete it
```

---

## 6. KMS Key Design

| Key Alias | Account | Purpose | Key Policy Principals |
|---|---|---|---|
| `alias/ire-ebs-cmk` | IRE-Core | EBS volume encryption | IRE-RecoveryAdmin-Role, EC2 service |
| `alias/ire-s3-cmk` | IRE-Core | S3 bucket encryption | IRE-SecurityAdmin-Role, S3 service, BackupRole (PROD replication only — encrypt) |
| `alias/ire-backup-cmk` | IRE-Core | AWS Backup vault | IRE-SecurityAdmin-Role, AWS Backup service |
| `alias/ire-secrets-cmk` | IRE-Core | Secrets Manager | IRE-RecoveryAdmin-Role, Secrets Manager service |
| `alias/ire-log-cmk` | IRE-Security | CloudTrail, CloudWatch Logs | IRE-SecurityAdmin-Role, CloudTrail service, CW Logs service |

**Universal Key Policy Deny Statement (applied to all IRE CMKs):**
```json
{
  "Sid": "DenyPRODDRAccess",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "kms:*",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:PrincipalAccount": ["<PROD-Account-ID>", "<DR-Account-ID>"]
    }
  }
}
```

---

## 7. Security Monitoring Design

### 7.1 GuardDuty Configuration

```
IRE-Security Account: GuardDuty Delegated Administrator
  Member Accounts: IRE-Core
  Features enabled:
    - S3 Protection
    - EKS Audit Logs (if applicable)
    - EC2 Malware Protection
    - RDS Login Activity (if applicable)
  Findings export: S3 bucket in IRE-Security (ire-gd-findings-<acct>)
  Findings retention: 90 days in GuardDuty + S3 export
```

### 7.2 Security Hub Configuration

```
IRE-Security Account: Security Hub Aggregator
  Member Accounts: IRE-Core
  Standards enabled:
    - AWS Foundational Security Best Practices (FSBP)
    - CIS AWS Foundations Benchmark v1.4
  Custom Insights:
    - "IRE Network Isolation" — tracks IGW/TGW/NAT GW config findings
    - "IRE Identity Findings" — tracks IAM anomalies
  Findings export: EventBridge → SNS → IRE on-call
```

### 7.3 CloudTrail Configuration

```
Trail Name: IRE-OrganizationTrail
Scope: IRE OU (IRE-Core + IRE-Security)
Log destination: s3://ire-cloudtrail-logs-<IRE-Security-acct-id>/
Encryption: ire-log-cmk
CloudWatch Logs: /aws/cloudtrail/IRE  (in IRE-Security)
Object Lock: Governance mode, 365 days
Multi-region: Yes
Management Events: All (Read + Write)
Data Events:
  - S3: Write events on ire-backup-data-* buckets
  - Lambda: All invocations (if applicable)

Bucket Policy:
  Allow: arn:aws:sts::*:assumed-role/cloudtrail-service/*
  Deny:  s3:DeleteObject from *
  Deny:  s3:PutBucketPolicy from non-IRE-Security accounts
```

### 7.4 AWS Config Rules

| Rule Name | Type | Check |
|---|---|---|
| `ire-no-igw` | Custom | No IGW attached to IRE VPCs |
| `ire-no-tgw-attachment` | Custom | No TGW VPC attachments exist |
| `ire-no-vpc-peering` | Custom | No VPC peering connections |
| `ire-s3-object-lock-enabled` | Managed | S3 Object Lock on all IRE buckets |
| `ire-kms-rotation-enabled` | Managed | KMS key rotation enabled |
| `ire-guardduty-enabled` | Managed | GuardDuty active in all IRE accounts |
| `ire-cloudtrail-enabled` | Managed | CloudTrail active |
| `ire-mfa-enabled-for-iam-access` | Managed | MFA on all console users |
| `ire-encrypted-volumes` | Managed | All EBS volumes encrypted |

### 7.5 Alerting Architecture

```
EventBridge (IRE-Security) ──► SNS Topic: ire-security-alerts
  Rules:
    - GuardDuty finding severity >= HIGH
    - Root account login (CloudTrail event)
    - Break-glass role assumption (CloudTrail: AssumeRoleWithSAML)
    - SCP violation attempt (CloudTrail: AccessDenied on SCP-blocked actions)
    - Config rule NON_COMPLIANT transition

SNS Subscriptions:
  - IRE Recovery Admin email distribution list
  - IRE Security team email distribution list
  - Dedicated PagerDuty integration (NOT shared with PROD PagerDuty)
```

---

## 8. SaaS Replacement Design

### 8.1 CrowdStrike → Amazon GuardDuty

| CrowdStrike Capability | AWS Replacement | Notes |
|---|---|---|
| Host-based threat detection | GuardDuty EC2 findings | Network-level; no agent required |
| Malware detection | GuardDuty Malware Protection | On-demand EBS scan |
| Process/file telemetry | GuardDuty Runtime Monitoring | Agent-based optional feature (AWS-managed agent only) |

### 8.2 Qualys → Amazon Inspector v2

| Qualys Capability | AWS Replacement | Notes |
|---|---|---|
| OS vulnerability scanning | Inspector v2 EC2 scanning | Agentless (SSM-based) |
| Package CVE detection | Inspector v2 | Integrated with NVD/CVE feeds |
| Container image scanning | Inspector v2 ECR scanning | Registry-level scanning |

### 8.3 Tanium → AWS Systems Manager

| Tanium Capability | AWS Replacement | Notes |
|---|---|---|
| Patch management | SSM Patch Manager | Patch baselines, maintenance windows |
| Remote command execution | SSM Run Command | Audited via CloudTrail |
| Inventory | SSM Inventory | Software/config inventory |
| Compliance reporting | SSM Compliance | Patch compliance dashboard |

### 8.4 Rubrik → AWS Backup + S3 Replication

| Rubrik Capability | AWS Replacement | Notes |
|---|---|---|
| Backup copy to isolated store | AWS Backup cross-account copy | Vault Lock for immutability |
| File-level recovery | EBS snapshot restore | Granularity depends on snapshot frequency |
| Immutable backup | S3 Object Lock + Vault Lock | Compliance-mode immutability |

---

## 9. Service Control Policies (Full Specification)

### SCP-01: `IRE-DenyNetworkExposure`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyIGW",
      "Effect": "Deny",
      "Action": [
        "ec2:AttachInternetGateway",
        "ec2:CreateInternetGateway"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyNATGW",
      "Effect": "Deny",
      "Action": ["ec2:CreateNatGateway"],
      "Resource": "*"
    },
    {
      "Sid": "DenyVPCPeering",
      "Effect": "Deny",
      "Action": [
        "ec2:CreateVpcPeeringConnection",
        "ec2:AcceptVpcPeeringConnection"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyTGW",
      "Effect": "Deny",
      "Action": [
        "ec2:CreateTransitGatewayVpcAttachment",
        "ec2:AcceptTransitGatewayVpcAttachment"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyRAMFromNonIRE",
      "Effect": "Deny",
      "Action": ["ram:AcceptResourceShareInvitation"],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/IRE-*"
        }
      }
    }
  ]
}
```

### SCP-02: `IRE-DenySecurityDisablement`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDisableGuardDuty",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "guardduty:StopMonitoringMembers",
        "guardduty:DisableOrganizationAdminAccount"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDisableCloudTrail",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDisableSecurityHub",
      "Effect": "Deny",
      "Action": [
        "securityhub:DisableSecurityHub",
        "securityhub:DisassociateMembers",
        "securityhub:DeleteMembers"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDisableConfig",
      "Effect": "Deny",
      "Action": [
        "config:DeleteConfigurationRecorder",
        "config:StopConfigurationRecorder",
        "config:DeleteDeliveryChannel"
      ],
      "Resource": "*"
    }
  ]
}
```

### SCP-03: `IRE-DenyIdentityMutation`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyIdCChangesFromNonSecurityAdmin",
      "Effect": "Deny",
      "Action": [
        "sso:DeletePermissionSet",
        "sso:DetachManagedPolicyFromPermissionSet",
        "sso:DeleteAccountAssignment",
        "sso:DisableUser",
        "sso:DeleteUser"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalAccount": "<IRE-Security-Account-ID>"
        }
      }
    }
  ]
}
```

---

## 10. Infrastructure as Code (IaC) Strategy

- All IRE infrastructure defined in **Terraform** (or AWS CDK)
- IRE IaC stored in a **dedicated Git repository** with no access from PROD CI/CD pipelines or PROD IAM roles
- Deployment pipeline uses an IRE-specific IAM role (`IRE-TerraformDeployRole`) with trust only to IRE Identity Center
- State backend: S3 bucket in `IRE-Security` account, encrypted with `ire-s3-cmk`
- State lock: DynamoDB table in `IRE-Security` account

---

## 11. Architecture Decision Records (ADRs)

### ADR-001: Standalone IAM Identity Center vs. PROD IdC Delegation
**Decision:** Standalone IAM Identity Center with local identity store  
**Rationale:** PROD IdC federated to Microsoft AD means a compromised AD controls all federated identities. A standalone instance with local users and hardware MFA severs this dependency. The operational overhead of managing local users is acceptable given the IRE is activated infrequently.  
**Alternatives Rejected:** PROD IdC delegation (shares AD trust chain), separate AWS SSO (same service, same concern).

### ADR-002: No NAT Gateway or Internet Gateway
**Decision:** IRE VPC has no internet path of any kind  
**Rationale:** All four SaaS tools (Rubrik, CrowdStrike, Qualys, Tanium) require outbound internet to function. Removing internet egress eliminates SaaS callback surface. AWS-native replacements operate entirely via PrivateLink endpoints.  
**Alternatives Rejected:** Network Firewall with FQDN blocklist (complex, operational burden, risk of misconfiguration bypassing the block).

### ADR-003: Push-only data replication from PROD to IRE
**Decision:** S3 replication and AWS Backup cross-account copy use push-from-PROD model  
**Rationale:** If IRE could pull from PROD, a compromised IRE could exfiltrate PROD data. One-way push limits blast radius in both directions.  
**Alternatives Rejected:** IRE-initiated pull (bidirectional trust concern), AWS DataSync (requires network path between accounts).

### ADR-004: AWS-Native Security Tooling in IRE
**Decision:** GuardDuty, Inspector v2, SSM replace CrowdStrike, Qualys, Tanium  
**Rationale:** SaaS security agents require internet connectivity and are managed from PROD consoles. In a compromise scenario, PROD SaaS consoles may be under adversary control. AWS-native services operate within the AWS control plane accessible via PrivateLink.  
**Alternatives Rejected:** Offline/disconnected agent mode for CrowdStrike (still requires periodic check-in; complex to configure and maintain).
