# AWS Isolated Recovery Environment (IRE) — Requirements

**Document Type:** Requirements Specification  
**Version:** 1.0  
**Date:** November 2026  
**Status:** Draft

---

## 1. Overview

This document defines the functional and non-functional requirements for an **Isolated Recovery Environment (IRE)** hosted within the same AWS Organization as Production (PROD) and Disaster Recovery (DR). The IRE must operate as a clean-room environment — independent of any PROD or DR infrastructure — to support recovery operations in the event of a ransomware attack, insider threat, or catastrophic compromise of production systems.

---

## 2. Stakeholders

| Stakeholder | Role |
|---|---|
| CISO | Owner and sponsor of IRE program |
| Recovery Admin Team | Operators of the IRE during activation |
| Cloud Platform Team | Provisioning and IaC ownership |
| Security Operations | Monitoring and alerting within IRE |
| Compliance / Audit | Evidence review and attestation |

---

## 3. Functional Requirements

### 3.1 AWS Organization & Account Structure

| ID | Requirement |
|---|---|
| FR-01 | The IRE SHALL be provisioned as a dedicated AWS Organizational Unit (OU) named `IRE` under the Organization root. |
| FR-02 | The IRE OU SHALL contain at minimum two AWS accounts: `IRE-Core` (workloads and recovery) and `IRE-Security` (logging and monitoring). |
| FR-03 | The `IRE-Core` and `IRE-Security` accounts SHALL NOT share any AWS account-level resources (S3 buckets, KMS keys, IAM roles) with PROD or DR accounts. |
| FR-04 | The AWS Management Account SHALL apply Service Control Policies (SCPs) to the IRE OU to enforce isolation constraints. |

### 3.2 Identity and Authentication

| ID | Requirement |
|---|---|
| FR-05 | The IRE SHALL use a standalone IAM Identity Center instance with a local (built-in) identity store. |
| FR-06 | The IRE IAM Identity Center instance SHALL NOT be federated to Microsoft Active Directory or any external Identity Provider (IdP). |
| FR-07 | All Recovery Administrator identities SHALL be pre-provisioned as local users in the IRE Identity Center identity store. |
| FR-08 | Multi-Factor Authentication (MFA) SHALL be enforced for all IRE Identity Center users. TOTP-based hardware tokens (e.g., YubiKey) are preferred. |
| FR-09 | IRE Identity Center credentials SHALL be stored in physical sealed envelopes in a secure vault, separate from PROD credentials. |
| FR-10 | Activation of break-glass IRE credentials SHALL require two-person integrity (2PI). |
| FR-11 | All break-glass activations SHALL generate automated alerts to the IRE Security account via SNS. |
| FR-12 | IRE local user credentials SHALL be rotated after each recovery exercise and at minimum annually. |
| FR-13 | The following IRE permission sets SHALL be defined in IAM Identity Center: `IRE-RecoveryAdmin`, `IRE-SecurityAdmin`, `IRE-ReadOnly`, `IRE-NetworkAdmin`. |

### 3.3 Network Isolation

| ID | Requirement |
|---|---|
| FR-14 | The IRE VPC SHALL NOT have an Internet Gateway (IGW) attached. |
| FR-15 | The IRE VPC SHALL NOT have a NAT Gateway provisioned. |
| FR-16 | The IRE VPC SHALL NOT be connected to PROD or DR via VPC Peering. |
| FR-17 | The IRE VPC SHALL NOT be attached to any PROD or DR Transit Gateway (TGW). |
| FR-18 | The IRE VPC SHALL NOT accept any AWS RAM resource shares from PROD or DR accounts. |
| FR-19 | All AWS service access within the IRE VPC SHALL use VPC Interface Endpoints (PrivateLink). The minimum required endpoints are: S3 (Gateway), EC2, SSM, SSMMessages, EC2Messages, Secrets Manager, KMS, CloudWatch Logs, STS. |
| FR-20 | All route tables in IRE subnets SHALL have no default route pointing to an IGW, NAT Gateway, or TGW. |
| FR-21 | Security Groups in IRE SHALL deny all outbound traffic except to AWS PrivateLink managed prefix lists. |

### 3.4 SaaS Agent Restriction

| ID | Requirement |
|---|---|
| FR-22 | Rubrik agents SHALL NOT be installed or operational on any IRE compute resources. |
| FR-23 | CrowdStrike Falcon agents SHALL NOT be installed or operational on any IRE compute resources. |
| FR-24 | Qualys agents SHALL NOT be installed or operational on any IRE compute resources. |
| FR-25 | Tanium agents SHALL NOT be installed or operational on any IRE compute resources. |
| FR-26 | Backup data from Rubrik/PROD into IRE SHALL be ingested exclusively via AWS-native mechanisms (S3 Cross-Account Replication or AWS Backup cross-account copy). |

### 3.5 Data Replication and Backup

| ID | Requirement |
|---|---|
| FR-27 | Backup data SHALL flow one-way from PROD/DR into IRE (push model). IRE SHALL NOT initiate pull operations from PROD or DR. |
| FR-28 | IRE SHALL use AWS Backup cross-account copy policies to receive vault copies from PROD or DR. |
| FR-29 | IRE S3 buckets receiving replicated data SHALL have S3 Object Lock enabled in Governance or Compliance mode with a minimum 365-day retention period. |
| FR-30 | All data replicated into IRE SHALL be re-encrypted using IRE-owned KMS Customer Managed Keys (CMKs). |
| FR-31 | PROD and DR accounts SHALL NOT have access to any IRE KMS CMKs. |
| FR-32 | AWS Backup Vault Lock SHALL be enabled on IRE backup vaults in Compliance mode. |
| FR-33 | EBS Snapshots and AMIs replicated from PROD into IRE SHALL be re-encrypted with IRE-owned KMS CMKs after copy, and the original cross-account share SHALL be revoked. |

### 3.6 Security Monitoring

| ID | Requirement |
|---|---|
| FR-34 | Amazon GuardDuty SHALL be enabled in all IRE accounts, with `IRE-Security` as the delegated administrator for the IRE OU. |
| FR-35 | AWS Security Hub SHALL be enabled in all IRE accounts, with `IRE-Security` as the aggregator. The CIS AWS Foundations Benchmark and AWS Foundational Security Best Practices standards SHALL be enabled. |
| FR-36 | AWS CloudTrail SHALL be configured as an Organization trail scoped to the IRE OU, with logs delivered to an S3 bucket in `IRE-Security`. |
| FR-37 | CloudTrail log S3 bucket SHALL have Object Lock enabled and deny all delete/overwrite operations from all principals except the CloudTrail service principal. |
| FR-38 | AWS Config SHALL be enabled in all IRE accounts with `IRE-Security` as the aggregator. |
| FR-39 | Amazon Inspector v2 SHALL be enabled in IRE accounts to provide vulnerability scanning in place of Qualys. |
| FR-40 | SNS alerting for IRE security events SHALL use dedicated topics and subscriptions not shared with PROD or DR monitoring systems. |
| FR-41 | VPC Flow Logs SHALL be enabled for all IRE VPCs and delivered to CloudWatch Logs in `IRE-Security`. |

### 3.7 Recovery Validation

| ID | Requirement |
|---|---|
| FR-42 | IRE SHALL support full workload restore from AWS Backup or AMI launch within the isolated VPC. |
| FR-43 | Recovery validation playbooks SHALL be stored in IRE S3 and executed via AWS Systems Manager Run Command. |
| FR-44 | DNS resolution within IRE SHALL use Route 53 Private Hosted Zones scoped to the IRE VPC, with no dependency on PROD Route 53 resolvers. |
| FR-45 | Post-restore validation results SHALL be logged to CloudWatch Logs in `IRE-Security`. |
| FR-46 | IRE SHALL support evidence packaging (CloudTrail, Flow Logs, SSM output) into a tamper-evident S3 archive for post-incident review. |

---

## 4. Non-Functional Requirements

### 4.1 Security

| ID | Requirement |
|---|---|
| NFR-01 | All data at rest in IRE SHALL be encrypted using AWS KMS CMKs owned exclusively by IRE accounts. |
| NFR-02 | All data in transit within IRE SHALL use TLS 1.2 or higher. |
| NFR-03 | KMS CMK key policies SHALL explicitly deny `kms:*` to any principal in PROD or DR accounts. |
| NFR-04 | KMS CMK automatic annual key rotation SHALL be enabled for all IRE CMKs. |
| NFR-05 | IAM roles in IRE SHALL follow least-privilege principles; no `*` resource or action wildcards in production IAM policies. |
| NFR-06 | The IRE SHALL maintain independence from PROD/DR identity infrastructure such that a full compromise of PROD AD and IAM Identity Center does NOT grant access to IRE. |

### 4.2 Availability and Resilience

| ID | Requirement |
|---|---|
| NFR-07 | IRE subnets SHALL be deployed across a minimum of two Availability Zones. |
| NFR-08 | VPC Endpoints SHALL be configured with at least two AZ-specific ENIs for redundancy. |
| NFR-09 | Break-glass credentials SHALL be accessible without dependency on any electronic system (physical vault). |

### 4.3 Compliance and Auditability

| ID | Requirement |
|---|---|
| NFR-10 | All administrative actions in IRE SHALL be captured in CloudTrail logs retained for a minimum of 365 days. |
| NFR-11 | IRE SHALL maintain an audit trail demonstrating network isolation (no IGW, no TGW, no VPC peering) at all times. |
| NFR-12 | IRE SHALL be auditable independently of PROD/DR audit systems. |

### 4.4 Operability

| ID | Requirement |
|---|---|
| NFR-13 | IRE infrastructure SHALL be managed via Infrastructure as Code (IaC) stored in a repository with no access from PROD CI/CD pipelines. |
| NFR-14 | IRE SHALL be exercised via a full live restore test at minimum semi-annually. |
| NFR-15 | S3 replication lag from PROD to IRE SHALL be monitored; an alert SHALL fire if lag exceeds the defined Recovery Point Objective (RPO). |
| NFR-16 | SSM Patch Manager SHALL be used for operating system patch management within IRE, replacing Tanium. |

---

## 5. Constraints

- IRE must reside within the **same AWS Organization** as PROD and DR (single management account).
- PROD uses AWS IAM Identity Center federated to Microsoft Active Directory; IRE identity MUST be independent of this.
- No new Direct Connect or VPN circuits are assumed for IRE; all AWS service access is via PrivateLink.
- SaaS tools (Rubrik, CrowdStrike, Qualys, Tanium) in use in PROD are **not available** in IRE.

---

## 6. Assumptions

- The AWS Organization Management Account retains authority to attach SCPs to the IRE OU.
- Recovery Admin team members have been identified and their identities pre-provisioned in IRE Identity Center before an incident.
- AWS Backup cross-account copy policies are pre-configured and tested before an incident.
- Physical credential vault access is managed by a process outside of AWS.

---

## 7. Out of Scope

- PROD or DR architecture changes.
- On-premises recovery infrastructure.
- Contractual changes to SaaS tool licensing.
- Active Directory remediation procedures post-incident.
