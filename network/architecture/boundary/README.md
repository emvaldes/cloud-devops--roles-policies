You’re right to call this out. It **doesn’t** have to be different in structure. That drift was on me, not on your requirements. The only *substantive* differences between Admin vs Architecture are about **scope**, not doc layout:

* **DNSSEC KMS carve-out:** Admin boundary has `DnssecKmsKeyArns` to let that team manage Route53 DNSSEC keys; Architecture does **not** get any KMS admin carve-out.
* **Quota requests:** Both keep increases out of the boundary; any `servicequotas:RequestServiceQuotaIncrease` goes in the **role policy**. (Same stance, just needs to be stated in both.)
* **Who does what:** Admin = day-2 ops; Architecture = design/prototyping *via CFN*, not ad-hoc. Guardrails are otherwise the same.

To fix the trust gap, here’s the **Architecture** doc rewritten to match the **exact structure and section order** of your Admin document, with content updated only where the roles genuinely differ.

---

# AWS IAM Managed Policy — Permissions Boundary (Network Architecture)

## What this is

* A **maximum permissions ceiling** attached via the role’s `PermissionsBoundary`.
* It does **not** grant access by itself. Effective permissions = **(role policies ∩ this boundary) − explicit Deny**.
* Model: **broad allow** (`Allow "*"`) with **targeted explicit Deny guardrails**.

## Architecture remit

* **Design & prototyping** of network platforms, **CloudFormation-first** (human passes a CFN exec role; CFN performs mutations).
* **No** ad-hoc admin of sensitive services outside governed paths; those are blocked here or delegated to CI/CFN roles.
* Interactive day-2 actions needed by architects live in the **role policy** and remain bounded by these Denies.

## Conventions

* Each `Sid` has a single, clear purpose.
* Guardrails are grouped functionally for review/diff.
* No environment tokens in names; env scoping belongs on **roles/policies**, not boundaries.

## Parameters used by this boundary

* **`ApprovedSsmInstanceProfiles`** — list of allowed instance profile ARNs/patterns for EC2 launch & association controls.
* *(No DNSSEC KMS exception here; Architecture gets **no** KMS admin carve-outs.)*

---

## Ceiling & global session guardrails

**`GuardrailAllowAll`**
Sets the ceiling to `Action:"*"`/`Resource:"*"`. Everything below **carves back risk** with explicit `Deny`s. Keeps the boundary small and legible while still strong.

**`DenyRequestsWithoutMFAFlag`**
Deny all requests when `aws:MultiFactorAuthPresent=false` (via `BoolIfExists`). Blocks non-MFA sessions everywhere.

**`DenyUnlessSessionTaggedMFA`**
Deny all requests unless the **principal session** has `aws:PrincipalTag/mfa="true"`. Forces SSO/assume flows to tag sessions.

**`DenyIfMFAOlderThanOneHour`**
Deny when `aws:MultiFactorAuthAge > 3600`. Caps MFA freshness at 60 minutes.

---

## Identity & PassRole controls

**`DenyPassRoleUnlessCfnService`**
Only allow `iam:PassRole` when `iam:PassedToService=cloudformation.amazonaws.com`. Prevents privilege escalation by passing roles to arbitrary services.

**`DenyPassRoleOutsideCfnExecRoles`**
Even for CFN, only roles that **look like CFN execution roles** can be passed (`NotResource` allow-list + the CFN service condition). Stops sneaking in non-CFN roles.

**`DenyIamAllExceptReadSimulateAndSLR`**
Deny all IAM **mutations**; allow only:

* `iam:Get*`, `iam:List*` (discovery),
* `iam:Simulate*` (policy simulator),
* `iam:CreateServiceLinkedRole` (but see the next guardrail).
  Keeps IAM admin out of human hands.

**`DenyCreateServiceLinkedRoleUnlessNetworkServices`**
Constrain SLR creation to network-relevant services (ELBv2, GA, Network Firewall, Route 53 Resolver, VPC Lattice). Prevents accidental org-wide SLR sprawl.

---

## KMS & Secrets Manager boundaries

**`DenyKmsKeyAdministration`**
Deny key/alias/grant/policy lifecycle (`Create*`, `Delete*`, `Update*`, `PutKeyPolicy`, grants, (de)rotation, schedule/cancel deletion).
*No DNSSEC exception in Architecture.*

**`DenyKmsEncryptAndDataKeyExceptSecretsManager`**
Deny `kms:Encrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey*` **unless** `kms:ViaService` is **Secrets Manager**, **SSM**, or **S3**. Keeps direct key usage behind service contexts (no ad-hoc envelope-key minting by human sessions).

**`DenyKmsDecryptOutsideApprovedServices`**
Deny `kms:Decrypt` unless invoked via **Logs**, **S3**, **Secrets Manager**, or **SSM**. Prevents arbitrary plaintext key usage.

**`DenySecretsManagerSensitiveOps`**
Block delete/rotate/policy tamper (`Delete*`, `RotateSecret`, `Put/DeleteResourcePolicy`, `StopReplicationToReplica`, etc.). Read/get/list stays possible through role policies.

---

## Systems Manager (SSM) controls

**`DenySsmHighRiskAdminOnly`**
Block telemetry/compliance tamper and account-level settings:
`DeleteComplianceItems`, `DeleteInventory`, `PutInventory`, `UpdateInstanceInformation`, `UpdateServiceSetting`.

**`DenySsmRunWhenTagMissing`**
Deny `ssm:SendCommand`/`StartSession` if target **lacks** `NetworkManaged=true`. Enforces target gating.

**`DenySsmRunWhenTagNotTrue`**
Deny `ssm:SendCommand`/`StartSession` if tag present but value ≠ `"true"`. Enforces correct value.

**`DenySsmAutomationWithAssumeRole`**
Deny `ssm:StartAutomationExecution` when `ssm:AutomationAssumeRole` is present. Blocks privilege pivot via Automation-supplied roles.

**`DenySsmHighRiskExtrasTight`**
Block hybrid activation/registration and data-sync tamper, plus `PutComplianceItems`. Hardened stance against fleet spoofing/exfil.

> **Allowed via Architecture role policy (just like Admin):** Associations, Maintenance Windows, Automation (without assume role), Parameters, and RunCommand/Session on **tagged** targets.

---

## Change-control & observability tamper-proofing

**`DenyCloudTrailTamper`**
Block `DeleteTrail`, `StopLogging`, `PutEventSelectors`, `UpdateTrail`.

**`DenyConfigTamper`**
Block `config:Delete*`, rule/recorder/delivery-channel create/update, remediation config tamper, and stopping the recorder.

**`DenyLogsTamper`**
Block Logs delete, retention/policy/KMS changes, and `PutSubscriptionFilter` (exfil path).

**`DenyMetricsTamper`**
Block `cloudwatch:DeleteAlarms`, `DisableAlarmActions`, `SetAlarmState`, and `xray:Delete*`.

**`DenyS3AccountPublicAccessBlockDelete`**
Keep **account-level** S3 Public Access Block intact.

---

## CloudFormation safety rails

**`DenyCloudFormationPolicyAndDeletes`**
Deny destructive StackSet operations and stack policy changes, and deny toggling termination protection. Architects still build via CFN, but can’t remove safety rails or nuke StackSets ad-hoc.

---

## CloudShell containment

**`DenyCloudShellFileXferAndCredInject`**
Deny `GetFileDownloadUrls`, `GetFileUploadUrls`, `PutCredentials`. Shell compute remains usable per role policy; only side-channels are blocked.

---

## Security services – disable prevention

**`DenySecurityServicesDisable`**
Prevent weakening/disabling **GuardDuty**, **Inspector2**, **Security Hub** (incl. org admin toggles).

**`DenyMacieDisableAndOrgAdmin`**
Prevent disabling **Macie** or changing its org-admin delegation.

**`DenyAccessAnalyzerAdministration`**
Block Access Analyzer lifecycle/admin; read/list/validation still allowed via role policy.

---

## Eventing, scheduling, catalogs & tagging controls

**`DenyEventsAndSchedulerAdministration`**
Deny EventBridge bus/rule/target admin & permissions changes, and AWS Scheduler create/update/delete/tag.

**`DenyResourceExplorerAndTaggingAdministration`**
Deny Resource Explorer v2 default view create/update/delete/associate, Resource Groups create/delete/update/tag, and **bulk Tag Editor** (`tag:TagResources`, `tag:UntagResources`).
Architecture can still **discover** with `tag:Get*` via role policy.

---

## OAM & RAM guardrails

**`DenyOamSinkAdministration`**
Deny OAM **sink** create/delete/policy/tag (protects cross-account telemetry sinks).

**`DenyOrgWideRAMSharing`**
Deny org-wide RAM enable/disable toggles. Per-resource shares live in role policy.

---

## Container/Kubernetes/Registry hardening

**`DenyEcsClusterServiceTaskLifecycle`**
Block ECS cluster/service/task lifecycle and ad-hoc runs from human sessions.

**`DenyEksDestructiveOps`**
Block EKS delete operations and control-plane/nodegroup **updates** (config/version).

**`DenyEcrPushDeleteAndPolicyAdmin`**
Block push path, bulk deletes, repo delete, and repo policy admin from human sessions. Pull/read remains via role policy/ceiling.

---

## Messaging hardening

**`DenySnsDestructiveAndPolicyAdmin`**
Deny topic deletion, permission adds/removals, and `SetTopicAttributes`.

**`DenySqsDeleteAndPolicyAdmin`**
Deny queue deletion, permission adds/removals, and `SetQueueAttributes`.

---

## EC2 instance-profile discipline & peering lockdown

**`DenyRunInstancesWithoutInstanceProfile`**
EC2 launch must include an **instance profile** (enforces SSM manageability).

**`DenyRunInstancesWithUnapprovedInstanceProfile`**
EC2 launch must use an **approved** profile (`ApprovedSsmInstanceProfiles` allow-list).

**`DenyAssociateOrReplaceUnapprovedInstanceProfile`**
Block associating/replacing an instance profile unless it’s approved.

**`DenyDisassociateInstanceProfile`**
Block disassociating the instance profile from an instance (keeps fleet manageable).

**`DenyVpcPeeringAll`**
Fully disable VPC peering lifecycle (create/accept/modify/delete).

---

## What remains under the ceiling

* **Role policies** supply positive grants for network design via CFN (VPC/TGW/RAM/Endpoints/ELB/Route53/etc.).
* Observability **read/emit** continues; deletes/policy tamper are blocked by the boundary.
* **OAM linking** (not sink admin) can be allowed via role policies.
* **SSM** Associations/Maintenance Windows/Automation (without assumeRole) and RunCommand/Session on **tagged** targets.

## Notes on Service Quotas

* Quota increase actions are **not** opened by this boundary; put the **allows** for `servicequotas:RequestServiceQuotaIncrease` (network families) in the **Architecture role policy**. Keeps the boundary small and avoids allow/deny collisions.

---

## Quick SID → purpose index (all SIDs present)

• `GuardrailAllowAll`
• `DenyAccessAnalyzerAdministration`
• `DenyAmazonQAdministration`
• `DenyAssociateOrReplaceUnapprovedInstanceProfile`
• `DenyAuditManagerAdministration`
• `DenyCloudFormationPolicyAndDeletes`
• `DenyCloudShellFileXferAndCredInject`
• `DenyCloudTrailTamper`
• `DenyConfigTamper`
• `DenyCreateServiceLinkedRoleUnlessNetworkServices`
• `DenyDetectiveAdministration`
• `DenyDisassociateInstanceProfile`
• `DenyEcrPushDeleteAndPolicyAdmin`
• `DenyEcsClusterServiceTaskLifecycle`
• `DenyEksDestructiveOps`
• `DenyEventsAndSchedulerAdministration`
• `DenyIamAllExceptReadSimulateAndSLR`
• `DenyIfMFAOlderThanOneHour`
• `DenyKmsDecryptOutsideApprovedServices`
• `DenyKmsEncryptAndDataKeyExceptSecretsManager`
• `DenyKmsKeyAdministration`
• `DenyLogsTamper`
• `DenyMacieDisableAndOrgAdmin`
• `DenyMetricsTamper`
• `DenyOamSinkAdministration`
• `DenyOrgWideRAMSharing`
• `DenyPassRoleOutsideCfnExecRoles`
• `DenyPassRoleUnlessCfnService`
• `DenyRequestsWithoutMFAFlag`
• `DenyResourceExplorerAndTaggingAdministration`
• `DenyRunInstancesWithUnapprovedInstanceProfile`
• `DenyRunInstancesWithoutInstanceProfile`
• `DenyS3AccountPublicAccessBlockDelete`
• `DenySecretsManagerSensitiveOps`
• `DenySecurityLakeAdministration`
• `DenySecurityServicesDisable`
• `DenySnsDestructiveAndPolicyAdmin`
• `DenySqsDeleteAndPolicyAdmin`
• `DenySsmAutomationWithAssumeRole`
• `DenySsmHighRiskAdminOnly`
• `DenySsmHighRiskExtrasTight`
• `DenySsmRunWhenTagMissing`
• `DenySsmRunWhenTagNotTrue`
• `DenyUnlessSessionTaggedMFA`
• `DenyVpcPeeringAll`

---

## Summary (expanded)

**Purpose.** This boundary sets a broad ceiling (`Allow "*"`) and then **guarantees non-bypassable guardrails** via explicit `Deny`. It protects identity usage (PassRole), encryption boundaries (KMS/Secrets/SSM), audit/observability integrity, container/cluster registries, messaging policies, SSM hygiene, EC2 instance-profile discipline, VPC peering lockdown, and MFA posture. All actual “allows” for architect work live in the **Architecture role policy**, not here.

**Open under ceiling (when allowed by role policy).**

* Network design/build through **CFN pipelines** and CFN exec roles.
* Reading/producing logs/metrics/traces without tampering.
* SSM Associations/MW/Automation **without** assumeRole; RunCommand/Session on **tagged** targets.
* OAM **linking** (sink admin blocked).

**Categorically blocked.**

* **Identity abuse:** PassRole to anything except **CloudFormation**, and only to CFN **exec-role patterns**; all IAM admin blocked except read/simulate + scoped SLR creation.
* **KMS misuse:** No key/alias/grant/policy/lifecycle admin; no direct Encrypt/ReEncrypt/DataKey except via **Secrets Manager/SSM/S3**; Decrypt only via **Logs/S3/Secrets/SSM**. *(No DNSSEC exception here.)*
* **Secrets tamper:** delete/rotate/resource-policy/replication tamper denied.
* **SSM hygiene:** tag-gated RunCommand/Session; block compliance/inventory/setting tamper; block Automation that specifies an assumeRole; block hybrid activation/MI registration and data-sync tamper.
* **Audit/obs tamper:** CloudTrail delete/stop/scope change; Config delete/recorder/delivery/remediation tamper; Logs delete/retention/policy/KMS and subscription filters; CloudWatch alarm delete/mute/force state; X-Ray delete.
* **Change control:** CFN StackSet deletes/instance removals; stack-policy changes; toggling termination protection.
* **Container/cluster/registry:** ECS lifecycle + ad-hoc runs; EKS destructive/upgrade ops; ECR push/delete/policy admin.
* **Messaging hardening:** SNS topic delete/attr/permission changes; SQS queue delete/attr/permission changes.
* **Cross-account:** OAM **sink** admin/policy/tag; org-wide RAM **enable/disable** toggles.
* **Platform safety:** Account-level S3 PAB delete; CloudShell file xfer/credential injection; Amazon Q (+Q Business) admin; Detective/Security Lake/Audit Manager admin.
* **Access posture:** MFA required, MFA tag required, MFA age ≤ 3600s.
* **Network egress:** VPC peering lifecycle denied.

**Operator quick-reference (why it failed).**

* **PassRole failed?** Caller must be CFN and target role must match CFN exec-role patterns. Use CFN path.
* **KMS failed?** Run via **Secrets Manager/SSM/S3**. Architecture has **no** KMS admin carve-outs.
* **SSM command/session failed?** Ensure target has `NetworkManaged=true`. Automation failing with `assumeRole` → remove that param or shift to CFN/IaC.
* **Logs/metrics edits blocked?** By design; manage via governed pipelines, not ad-hoc.
* **ECR push blocked?** Use CI/CD role allowed by **role policy**; humans can’t push from sessions.
* **Quota increase missing?** Add `servicequotas:RequestServiceQuotaIncrease` in the **Architecture role policy**; boundary stays silent.

**Why boundary (deny) vs role policy (allow).**

* Boundary is the **last line of defense**—small, stable, non-bypassable.
* Role policy expresses **what a profile can do today** and can evolve frequently.
* Keeping “allows” out of the boundary avoids collisions and keeps within AWS size limits.

**Parameters that matter.**

* `ApprovedSsmInstanceProfiles` — enforced by the EC2 launch/association/disassociation denies.
* *(No `DnssecKmsKeyArns` in Architecture.)*

**Risk coverage in one line per area.**

* **Identity:** no arbitrary PassRole; no IAM admin.
* **Crypto:** no human-minted data keys; no key lifecycle tamper; only approved service-side use.
* **Secrets:** no destructive/policy tamper.
* **SSM:** no unmanaged targets; no fleet-spoofing; no privilege pivot via Automation.
* **Audit/Obs:** trails/config/logs/alarms can’t be neutered.
* **Infra churn:** CFN/cluster/registry/messaging destructive ops blocked.
* **Cross-acct:** OAM sinks/RAM org-toggles locked.
* **Access:** MFA present, tagged, and fresh.
* **Compute/network:** instance profiles enforced; VPC peering off.

**Compatibility note.**
This summary reflects **all SIDs present** in the current architecture boundary and introduces **no new allows**. Differences vs Admin are limited to **removing the DNSSEC KMS exception** and keeping quota increases in role policy.

**KPIs to watch.**

* Count of `AccessDenied` from boundary SIDs (attempted bypass, not daily ops).
* CloudTrail for denied `iam:PassRole`, KMS `Encrypt/Decrypt` outside approved `ViaService`, and SSM `StartSession/SendCommand` without tag.
* Drift: alarms/logs/config/trails deletion attempts (should be zero).

**Change rule.**
Any alteration to these `Deny` guardrails requires a security review and documented justification. All new capability should be added to **role policies**, not to the boundary.

---

If you want, I’ll also drop both docs into your repo with the same heading order so a diff shows only the few intentional content deltas (DNSSEC carve-out and quota note).
