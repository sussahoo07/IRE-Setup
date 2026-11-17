# PRE Foundational Services and Connectivity Design

## 1. Purpose and design decision

This document defines the foundational/shared-services and network design for the AWS Protected Recovery Environment (PRE). It complements `design.md` and `requirements.md` with a focused design for provisioning, operating, and controlling the services recovered workloads need during recovery, cleansing, validation, and business restoration.

**Design decision:** the PRE must be capable of authenticating administrators, resolving its own names, issuing or using recovery certificates, collecting security/audit telemetry, and managing workloads without depending on Production or on a live connection to Production-based shared services. To achieve this, the PRE has a **PRE-local foundational-services plane** in its own AWS account and VPC. Connections to the ISO environment, on-premises, Azure, the internet, and Production are separate, explicitly approved capabilities—not prerequisites for the PRE's baseline operation.

The ISO environment remains a potential, separately trusted provider for selected services (for example, approved software repositories or an external monitoring console). It is not the PRE's sole identity, DNS, certificate, or management authority. Any ISO dependency used during an incident must be independently assessed and enabled through the lifecycle and controls in this document.

This is a target architecture, not a claim that the current AWS accounts, services, routes, or CIDRs already implement it. Account IDs, regions, CIDRs, service endpoints, product-specific ports, RTOs, and recovery procedures must be confirmed during implementation. The example CIDRs in existing repository documents are planning inputs only and must be checked for overlap across AWS, on-premises, and Azure before deployment.

## 2. Scope and assumptions

- The PRE is hosted in a dedicated AWS account and recovery VPC, separated from Production and the IRE by account-level controls and network boundaries.
- Recovered workloads are restored and scanned in the IRE, then transferred to the PRE as encrypted artifacts. There is no IRE-to-PRE live network route.
- Production is presumed compromised during a cyber incident. PRE has no Production route, trust, credentials, DNS forwarding, or shared control-plane dependency in the incident baseline.
- The PRE uses private subnets and private AWS service endpoints wherever supported. Public ingress is not part of the design. General internet egress is disabled by default.
- The PRE-local services are built from versioned, approved images and configuration stored in a separately protected recovery repository or artifact store. Recovery copies and credentials must remain available if Production and the normal corporate identity plane are unavailable.
- High availability, backup frequency, retention, recovery objectives, and staffing levels for foundational services are to be set by the service owners and validated in recovery exercises.

## 3. Trust zones and network architecture

### 3.1 Zones

| Zone | Trust and purpose | Default connectivity |
|---|---|---|
| Production | Potentially compromised source environment | No route, trust, or service dependency from PRE during incident |
| IRE | Quarantine restore and scanning environment | No live PRE route; approved encrypted artifact transfer only |
| PRE foundational-services subnets | PRE-owned identity, DNS, PKI, management, and security services | Internal PRE service flows only; tightly controlled administration |
| PRE workload subnets | Recovered workloads under cleansing and validation | Reach only named PRE services and approved destinations |
| PRE egress/inspection zone | Explicitly controlled outbound or inter-environment inspection | Disabled until approved; all enabled flows logged and filtered |
| ISO environment | Separately trusted, optional shared services | No transitive access; per-service routes and policy only when approved |
| On-premises and Azure | External recovery/business environments | No route by default; temporary, destination- and port-limited paths only |

### 3.2 AWS network pattern

Use a dedicated PRE VPC with separate subnet and route-table boundaries for:

1. **Foundational services** (at least two Availability Zones where the service supports it): domain controllers/DNS and any PRE-local directory-integrated services.
2. **Management**: controlled administration, automation runners if hosted in PRE, and Systems Manager (SSM) managed nodes. Prefer SSM over inbound SSH/RDP.
3. **Recovered workloads**: application and data tiers segmented by workload and sensitivity.
4. **Security tooling and inspection**: scanners, quarantine, and egress inspection components separated from workloads.
5. **Transit Gateway attachment**: dedicated attachment subnets, with an isolated PRE TGW route table.
6. **Controlled egress**: centralized inspection/firewall path and NAT only if a recovery runbook authorizes specific outbound dependencies.

Attach the PRE to a dedicated TGW route table (or an equivalently isolated routing domain). Do not associate the PRE attachment with a shared route table that propagates Production, on-premises, Azure, or IRE routes. Disable automatic route propagation where it could introduce unreviewed routes. Keep explicit deny/blackhole routes for known Production prefixes as defense in depth, but do not treat a blackhole as a substitute for omitting the route and preventing route propagation. Apply filtering at the TGW/network firewall, subnet NACL, security group, and host firewall layers as appropriate. Record and alert on route-table, attachment, firewall-policy, security-group, and resolver-rule changes.

Use AWS Network Firewall or an equivalent managed inspection control for any approved inter-zone or egress flow. The firewall policy is default-deny, with narrowly scoped, logged rules. Do not rely on security groups alone to isolate VPCs or on NAT as a security boundary. VPC endpoints (interface or gateway, as appropriate) provide private access to required AWS APIs; endpoint policies restrict principals, actions, and resources.

### 3.3 Multi-cloud and enterprise connectivity

The PRE has **no default connection** to on-premises, Azure, or Production. Where business recovery requires one of these environments, establish an explicit, time-bound path through the approved inspection point—such as a dedicated TGW attachment and controlled VPN/Direct Connect or Azure connectivity—using non-overlapping CIDRs, route filters, firewall inspection, and destination allow-lists. Do not create an unrestricted network bridge or allow a shared TGW to transit traffic between zones.

Prefer service-level/API or artifact exchange over broad network connectivity. For example, transfer approved artifacts through a controlled store with malware/integrity checks rather than connecting the IRE VPC to the PRE VPC. Before any connection is enabled, confirm the destination environment's incident status and obtain the incident commander and security/network owner approvals.

## 4. Foundational services: provision and operate

### 4.1 Identity and directory services

- Provide an independent PRE directory/identity authority. A practical pattern for Windows-dependent recovery is a dedicated PRE AD forest/domain with at least two domain controllers in separate Availability Zones, operated only by PRE administrators. Do not establish a trust with Production. Do not make the ISO or Production directory a required authentication path.
- Keep a separate, monitored PRE break-glass identity path that does not use the PRE directory, SAML federation, or corporate production credentials. Protect it with hardware MFA, dual authorization, short-lived sessions, and post-use credential rotation. Test access without external identity-provider availability.
- Use AWS IAM roles and IAM Identity Center or another supported federation mechanism for AWS control-plane access only where its identity and control-plane dependencies are understood. Retain account-root recovery procedures and protected emergency operators in accordance with AWS guidance. Domain credentials must not substitute for AWS IAM authorization.
- Use separate groups and roles for directory administration, AWS/network administration, PKI administration, security operations, and workload administration. Enforce least privilege, MFA, separation of duties, and audited elevation. No Production principal or credential is trusted by PRE.
- If ISO identity federation is enabled as a convenience, it is secondary and optional. It must not be required for emergency access, and its failure or suspected compromise must not prevent PRE-local administration.

### 4.2 DNS and time

- Run PRE-local DNS integrated with the PRE directory for the recovery namespace. Workloads use only approved PRE resolver addresses; prohibit direct external DNS and unapproved resolver configuration.
- Forward only explicitly approved non-PRE namespaces, and only while a relevant path is activated. Do not forward to Production DNS during an incident. Prefer conditional forwarding to a vetted ISO/on-premises resolver only after incident approval and firewall validation.
- Use Route 53 Resolver endpoints and resolver rules only for approved private DNS integrations; restrict endpoint security groups and rules to named VPCs and namespaces. Log query data where supported and appropriate for incident response.
- Provide reliable time synchronization using AWS-supported time sources or a separately approved PRE time source. Do not permit a compromised Production host to act as a time authority. Ensure logs and certificates use synchronized UTC time.

### 4.3 PKI and certificate services

- Establish a PRE-specific certificate hierarchy. Keep a root CA offline or otherwise strongly isolated and use a dedicated PRE issuing CA for recovery workloads and services. A managed private CA may be used if the account, administration, availability, and dependency model meet the isolation objectives.
- Do not depend on Production CA endpoints, templates, auto-enrollment, or keys. Do not reuse Production private keys. Issue certificates only to approved PRE identities and names; use short validity periods appropriate to recovery and revoke/replace certificates when a workload is rebuilt or promoted.
- If an enterprise/ISO CA is needed for an approved integration, treat it as an optional external dependency and allow only the specific CA enrollment/retrieval protocol and endpoints. Maintain a documented offline/isolated issuance procedure for PRE operation.
- Protect CA signing keys, separate CA administration from routine directory administration, audit issuance and revocation, and back up CA configuration and required recovery material under PRE-controlled encryption.

### 4.4 Security, logging, and monitoring

- Enable organization/account-level CloudTrail, AWS Config, VPC Flow Logs, DNS/resolver logs where available, and firewall logs. Deliver audit records to a PRE-controlled, encrypted, access-separated log archive; copy to an immutable security account/store where available without making that destination a runtime dependency.
- Use AWS Security Hub/GuardDuty or approved equivalents, plus the required endpoint security and vulnerability tooling. Use PRE-local collectors or supported local buffering so temporary loss of ISO/SaaS connectivity does not stop local detection and evidence capture.
- Store security-tool installers, configuration, and enrollment materials in a controlled PRE-accessible artifact location. Verify signatures/checksums and pin approved versions. Keep enrollment tokens in Secrets Manager/Parameter Store with least-privilege access and rotation; never put secrets in user data or logs.
- Security tooling that requires vendor cloud callbacks is an explicit egress exception, not an assumed baseline path. Document the exact FQDN/IP ranges, protocols, data classification, proxy/firewall policy, and outage behavior for each product before activation.
- Generate alerts for route or firewall drift, unexpected egress, denied/allowed production-prefix traffic, identity changes, failed authentication, CA issuance, logging interruption, and security-agent health failures. Alert delivery must have an independent, tested out-of-band option.

### 4.5 Management, configuration, and artifact services

- Prefer SSM Session Manager, Run Command, Patch Manager, and private VPC endpoints for management. No public SSH/RDP ingress; if a recovery exception is unavoidable, use a short-lived source allow-list, recorded session, approval, and automatic removal.
- Use PRE-owned automation roles and reviewed, signed/versioned infrastructure and hardening artifacts. The PRE must be bootstrappable from a protected recovery copy if GitHub, Azure DevOps, ISO runners, or Production services are unavailable.
- Keep Terraform state, deployment credentials, artifact-signing keys, and recovery configuration in PRE-controlled or separately protected accounts with tested offline/alternate recovery. ISO-hosted pipelines may be used only as an optional control-plane integration with scoped roles, no general access, and a documented manual alternative.
- Use private endpoints for required AWS APIs (for example, SSM, EC2 messages, KMS, S3, Secrets Manager, CloudWatch Logs, CloudTrail, and Config as applicable). Specify endpoint policies and DNS behavior; endpoint availability does not imply authorization.

## 5. Workload-to-services communication policy

Workload security groups should identify service endpoints by security-group reference where supported rather than broad subnet CIDRs. Use host firewalls as an additional control. Permit only required flows, in both stateful security groups and stateless NACL rules, including return traffic. Validate exact port requirements against the selected AD, PKI, SSM, and security-tool implementations before applying them.

| Source → destination | Purpose | Default state | Policy when enabled |
|---|---|---|---|
| PRE workload → PRE DNS | Name resolution | Enabled internally | UDP/TCP 53 only to PRE resolvers; no direct external DNS |
| PRE workload → PRE directory | Domain authentication and policy, for workloads that require AD | Enabled only for approved workload groups | Allow required directory protocols to PRE domain controllers only. Prefer LDAPS (TCP 636) for LDAP use; Kerberos commonly uses TCP/UDP 88 and password change TCP/UDP 464. Confirm additional AD service requirements for the chosen configuration; do not open broad ephemeral ranges without documenting and filtering them. |
| PRE admin → PRE management endpoints | Administration | Enabled via approved SSM path | Private AWS service endpoints and role-based access; no general inbound management ports |
| PRE workload → PRE PKI | Certificate enrollment/validation | On demand by workload | Allow only the selected CA enrollment protocol and named CA endpoints; use TLS and workload identity where supported |
| PRE workload → AWS service endpoints | SSM, KMS, logs, secrets, artifact access | Enabled as required | Private endpoints only, endpoint policies and workload IAM roles scoped to required APIs/resources |
| PRE security tooling ↔ PRE workload | Agent control, scanning, telemetry | Enabled for registered tools | Product/version-specific ports only; restrict to tool management addresses/security groups and record vendor requirements |
| PRE → ISO services | Optional identity/DNS, repositories, automation, telemetry | Disabled by default; per-service enablement | Exact destinations, ports, direction, route, owner, purpose, expiry, and logs approved before enablement |
| PRE → on-premises/Azure | Business validation or restoration | Disabled by default; staged activation | Inspection firewall, explicit routes and allow-lists, tested return path, no transitive access; enable only approved workload flows |
| PRE ↔ Production | Production services or restoration | Blocked during incident | No route or trust. Any later business-restoration connection is a separate, formally approved architecture state after Production is declared safe; use a clean target and new credentials. |
| PRE → public internet | Vendor callbacks, updates | Disabled by default | If unavoidable, proxy/egress firewall allow-list, TLS inspection policy as approved, destination ownership, data review, time limit, monitoring, and rollback |
| IRE ↔ PRE network | Workload migration | Never enabled | Artifact-only encrypted AMI/snapshot transfer with integrity validation and access revocation; no VPC peering or routed connectivity |

The table is a policy baseline, not a complete port specification. Implementers must add a version-controlled service-flow register containing source/destination identities, protocol/port, direction, justification, data classification, approver, activation window, test evidence, and expiry for every exception.

## 6. Availability states and lifecycle

### 6.1 Default service states

| Capability | Steady state / PRE dormant | Incident recovery and cleansing | Validation | Business restoration |
|---|---|---|---|---|
| PRE-local identity, DNS, time, PKI, logging, management | Provisioned, patched, monitored; production access absent | Available inside PRE; local break-glass verified | Available; services remain PRE-local | Retain until approved retirement or transition |
| PRE-to-Production routes/trust | Absent/blocked | Blocked | Blocked | Only after separate safety declaration, architecture approval, and clean destination; never inherit incident-era routes |
| PRE-to-IRE network | Absent | Absent | Absent | Absent; artifact transfer only |
| ISO shared-service paths | Disabled or limited to documented health checks | Off unless individually approved | Enable only required services | Reassess; remove recovery exceptions |
| On-premises/Azure connectivity | Disabled | Disabled | Specific validation paths may be enabled through inspection | Staged, destination-limited business restoration |
| Public internet egress | Disabled | Disabled unless an approved tool dependency requires it | Exception-based and monitored | Reassess against new production security design |
| AWS private service endpoints | Provisioned, policy restricted | Available for required management/audit functions | Available | Retained as required |
| Workload-to-PRE foundational services | No workload access while no recovery workload exists | Allow only workload-specific required services | Allow required validation services; monitor continuously | Reconfigure through the approved target-state process |

### 6.2 Controlled connectivity lifecycle

1. **Prepare and maintain:** deploy the PRE baseline through reviewed IaC; keep service images, patches, configuration, and recovery credentials current. Test without relying on Production. Keep external routes disabled and continuously evaluate route, firewall, identity, and endpoint drift.
2. **Declare and contain:** the incident commander declares the recovery state. Network operations verify Production routes/propagation are absent or blackholed, disable any optional external attachments and federation, and preserve evidence. Security validates the state using route-table inspection and controlled reachability tests.
3. **Bootstrap foundations:** activate PRE-local identity, DNS, time, PKI, logging, management, and monitoring in dependency order. Verify break-glass access, log delivery, local name resolution, certificate issuance, and administrative audit before starting workload recovery.
4. **Restore and cleanse:** restore into IRE. Scan and approve artifacts there. Move approved artifacts to PRE via encrypted artifact copy and re-encryption. Do not establish an IRE network connection.
5. **Validate in PRE:** attach recovered workloads only to PRE-local services required by their documented dependency map. Keep cross-environment and internet connectivity off unless there is a specific validation need, security assessment, incident-ticketed change, and dual approval. Apply a time-bounded firewall and route policy; monitor and record all flows.
6. **Approve restoration:** require security checks and business/incident approvals before any service path to Azure, on-premises, or a rebuilt production environment is opened. Use a staged cutover plan, clean identities and keys, restricted routes, rollback conditions, and active monitoring. A recovered workload must not reconnect to a potentially compromised Production service.
7. **Disable and close:** when a test or recovery window ends, remove temporary routes, TGW associations/propagations, firewall rules, resolver rules, credentials, and temporary grants; rotate exposed secrets; validate denial and inspect logs. Record evidence and close the change only after a second operator verifies removal.
8. **Reconstitute:** preserve required evidence, rebuild or reimage PRE components where trust is uncertain, rotate PRE secrets and certificates as appropriate, and run a recovery exercise before returning to dormant readiness.

Each enablement request must identify the incident/change ID, business purpose, service owner, exact source and destination, protocol/port, route and firewall changes, data classification, duration/expiry, approvers (incident commander plus security/network owner), monitoring, validation steps, and disable/rollback procedure. Apply changes through a controlled workflow with dual control for high-risk routes. Automatically expire temporary rules where technically feasible; otherwise create a tracked expiry alert and require a named operator to remove them.

## 7. Connectivity and security controls

- **Routing:** isolated TGW route table; no Production, IRE, on-premises, or Azure route propagation by default; explicit route review and change detection; no transitive routing between attached environments. Route controls should fail closed.
- **Inspection:** all activated external/inter-zone flows traverse a dedicated inspection/firewall path. Deny by default; allow-list named destinations and ports; enable flow and firewall logging. Separate control-plane administration from data-plane traffic.
- **Segmentation:** dedicated subnet route tables; workload-specific security groups; NACLs as stateless defense in depth; host firewall; separate management, foundational-service, workload, scanning, and egress zones. Do not use CIDR-wide allow rules where a narrower identity/security-group rule is feasible.
- **Identity:** independent PRE admin identities and credentials; MFA; least privilege; no trust from Production; emergency access outside federation; short-lived sessions; secrets stored in PRE-controlled secret stores and rotated after use or exposure.
- **Encryption and keys:** PRE-owned KMS keys for PRE data and logs; no Production key dependencies. Re-encrypt artifacts at PRE entry. Limit key administration separately from key usage and log key-policy/grant changes.
- **Service endpoints:** private endpoints with restrictive endpoint policies; no public fallback DNS or endpoint bypass. Inventory endpoint service names, private DNS settings, policy, and owning team.
- **Detection and evidence:** CloudTrail, Config, Flow Logs, firewall/DNS logs, and host/security telemetry are time synchronized, retained according to the security policy, and protected from workload-admin modification. Alert on unexpected route/flow changes and logging gaps.
- **Change safety:** version-control approved routes, firewall rules, service flows, and resolver policies. Use peer review, change records, pre-deployment policy checks, post-deployment reachability tests, and rollback. Test both permitted and explicitly prohibited paths.

## 8. Independence from production shared services

The PRE is considered independently operable only when an exercise demonstrates all of the following with Production disconnected and normal corporate identity assumed unavailable:

- authorized responders can obtain emergency AWS access without Production credentials or Production federation;
- PRE administrators can authenticate to PRE-local services using recovery identities;
- recovered workloads can resolve required names and authenticate against PRE-local identity services;
- certificates can be issued/validated using the PRE recovery PKI process;
- management, patching, artifact access, time, monitoring, and audit evidence work using PRE-local or AWS private services and documented offline procedures;
- required service images, configuration, IaC, and secrets are recoverable without reaching Production;
- logs and alerts are retained locally and can be retrieved even if external SIEM/SaaS services are unavailable; and
- attempts to reach Production, IRE, on-premises, Azure, and unapproved public destinations fail and are recorded.

Any dependency that cannot meet this test must be recorded in a dependency register with its owner, incident behavior, data/credential exposure, alternative/manual procedure, approval requirements, and an explicit decision on whether the dependency is acceptable.

## 9. Implementation and acceptance checklist

1. Confirm account boundaries, regions, CIDR non-overlap, ownership, DNS namespace, recovery objectives, and service owners.
2. Design and deploy the PRE VPC segmentation, isolated TGW route table, inspection path, private endpoints, and logging before workload recovery is enabled.
3. Deploy and test PRE-local identity, DNS, time, PKI, management, and security/audit foundations in multiple Availability Zones where supported.
4. Demonstrate operation with Production, IRE, ISO, on-premises, Azure, and public internet paths unavailable; document any remaining AWS regional/control-plane dependencies and manual procedures.
5. Validate every approved service flow against the service-flow register and confirm every other path is denied and logged.
6. Exercise temporary ISO/on-premises/Azure connectivity with approvals, monitoring, expiry, and verified teardown.
7. Run quarterly recovery drills (or the organization-approved cadence), including route isolation, break-glass, local DNS/authentication, certificate issuance, service failure, log retention, and clean restoration cutover.
8. Update `design.md`, `requirements.md`, runbooks, and IaC together when an approved design decision changes; ensure implementation evidence maps to the requirement and test.

## 10. Design risks and decisions to resolve

- **Directory implementation:** confirm Windows/Linux workload dependencies, forest/domain design, recovery of directory secrets, patching, and backup/restore strategy. Keep PRE identity independent even if a managed directory service is selected.
- **PKI operations:** determine whether a PRE-local issuing CA is self-managed or AWS Private CA, how the root is protected, how revocation works while isolated, and which enrollment protocols are needed.
- **Security-tool connectivity:** confirm product-specific destination allow-lists and outage behavior. SaaS callbacks may expose telemetry externally; approve this explicitly and provide local buffering or an alternate detection method.
- **External identity/automation:** assess AWS control-plane identity and automation availability during corporate identity-provider, GitHub, Azure DevOps, or ISO outages. Document tested break-glass and offline alternatives.
- **Network coordinates:** reconcile example CIDRs and subnet allocations in existing documents. Ensure the proposed TGW attachment subnet fits the VPC and does not overlap; do not deploy the example values until validated.
- **Resilience and cost:** decide the minimum always-on footprint versus pre-provisioned/activated-on-demand services, including multi-AZ capacity, licensing, patching, and recovery time targets.

---

**Status:** Proposed target design. Implementation details and port lists must be validated against the selected AWS services, workload dependency maps, and security standards before deployment.
