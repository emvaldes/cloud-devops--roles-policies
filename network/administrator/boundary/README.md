# AWS IAM Managed Policy — Permissions Boundary (Network Administrator)

## What this is

* A **maximum permissions ceiling** attached via the role’s `PermissionsBoundary`.
* It does **not** grant access by itself. Effective permissions = **(role policies ∩ this boundary) − explicit Deny**.
* Model: **broad allow** (`Allow "*"`) with **targeted explicit Deny guardrails**.

## Administrator remit

* Day-2 ops, maintenance, diagnostics, and controlled changes for **network** services.
* **No** “build anything” prototyping; that lives in other roles and in role policies, not the boundary.
* Prefer **CloudFormation-driven** changes; direct APIs are constrained where risky.

## Conventions

* Each `Sid` has a single, clear purpose.
* Guardrails are grouped functionally for review/diff.
* No environment tokens in names; env scoping belongs on **roles/policies**, not boundaries.

## Parameters used by this boundary

* **`ApprovedSsmInstanceProfiles`** — list of allowed instance profile ARNs/patterns for EC2 launch & association controls.
* **`DnssecKmsKeyArns`** — **exception list** of KMS key ARNs used for **Route 53 DNSSEC**. These are *excluded* from KMS-admin denies so the right team can manage those keys.

---

## Ceiling & global session guardrails

1. **`GuardrailAllowAll`**
   Sets the ceiling to `Action:"*"`/`Resource:"*"`. Everything below **carves back risk** with explicit `Deny`s. Keeps the boundary small and legible while still strong.

2. **`DenyRequestsWithoutMFAFlag`**
   Deny all requests when `aws:MultiFactorAuthPresent=false` (via `BoolIfExists`). Blocks non-MFA sessions everywhere.

3. **`DenyUnlessSessionTaggedMFA`**
   Deny all requests unless the **principal session** has `aws:PrincipalTag/mfa="true"`. Forces SSO/assume flows to tag sessions.

4. **`DenyIfMFAOlderThanOneHour`**
   Deny when `aws:MultiFactorAuthAge > 3600`. Caps MFA freshness at 60 minutes.

---

## Identity & PassRole controls

2. **`DenyPassRoleUnlessCfnService`**
   Only allow `iam:PassRole` when `iam:PassedToService=cloudformation.amazonaws.com`. Prevents privilege escalation by passing roles to arbitrary services.

3. **`DenyPassRoleOutsideCfnExecRoles`**
   Even for CFN, only roles that **look like CFN execution roles** can be passed (`NotResource` allow-list + the CFN service condition). Stops sneaking in non-CFN roles.

4. **`DenyIamAllExceptReadSimulateAndSLR`**
   Deny all IAM **mutations**; allow only:

   * `iam:Get*`, `iam:List*` (discovery),
   * `iam:Simulate*` (policy simulator),
   * `iam:CreateServiceLinkedRole` (but see the next guardrail).
     Keeps IAM admin out of human hands.

5. **`DenyCreateServiceLinkedRoleUnlessNetworkServices`**
   Constrains SLR creation to network-relevant services (ELBv2, GA, Network Firewall, Route 53 Resolver, VPC Lattice). Prevents accidental org-wide SLR sprawl.

---

## KMS & Secrets Manager boundaries

23. **`DenyKmsKeyAdministration`**
    Deny key/alias/grant/policy lifecycle (`Create*`, `Delete*`, `Update*`, `PutKeyPolicy`, grants, (de)rotation, schedule/cancel deletion).
    **Exception**: `NotResource: DnssecKmsKeyArns` — DNSSEC keys are managed outside this guardrail so DNSSEC can be administered cleanly.

24. **`DenyKmsEncryptAndDataKeyExceptSecretsManager`**
    Deny `kms:Encrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey*` **unless** `kms:ViaService` is **Secrets Manager**, **SSM**, or **S3**. Keeps direct key usage behind service contexts (no ad-hoc envelope-key minting by human sessions).

25. **`DenyKmsDecryptOutsideApprovedServices`**
    Deny `kms:Decrypt` unless invoked via **Logs**, **S3**, **Secrets Manager**, or **SSM**. Prevents arbitrary plaintext key usage.

26. **`DenySecretsManagerSensitiveOps`**
    Block delete/rotate/policy tamper (`Delete*`, `RotateSecret`, `Put/DeleteResourcePolicy`, `StopReplicationToReplica`, etc.). Read/get/list stays possible through role policies.

---

## Systems Manager (SSM) controls

30. **`DenySsmHighRiskAdminOnly`**
    Block telemetry/compliance tamper and account-level settings:
    `DeleteComplianceItems`, `DeleteInventory`, `PutInventory`, `UpdateInstanceInformation`, `UpdateServiceSetting`.

31. **`DenySsmRunWhenTagMissing`**
    Deny `ssm:SendCommand`/`StartSession` if target **lacks** `NetworkManaged=true`. Enforces target gating.

32. **`DenySsmRunWhenTagNotTrue`**
    Deny `ssm:SendCommand`/`StartSession` if tag present but value ≠ `"true"`. Enforces correct value.

33. **`DenySsmAutomationWithAssumeRole`**
    Deny `ssm:StartAutomationExecution` when `ssm:AutomationAssumeRole` is present. Blocks privilege pivot via Automation-supplied roles.

34. **`DenySsmHighRiskExtrasTight`**
    Blocks hybrid activation/registration and cross-region data sync tamper, plus `PutComplianceItems`. Hardened stance against fleet spoofing/exfil.

> **Note**: Associations, Maintenance Windows, Automation (without assume role), Parameters, RunCommand/Session (on tagged targets) are allowed by **role policies**; the boundary only blocks high-risk paths.

---

## Change-control & observability tamper-proofing

18. **`DenyCloudTrailTamper`**
    Blocks `DeleteTrail`, `StopLogging`, `PutEventSelectors`, `UpdateTrail`. Prevents audit trail disablement/scope tamper.

19. **`DenyConfigTamper`**
    Blocks `config:Delete*`, rule/recorder/delivery-channel **create/update**, remediation config tamper, and stopping the recorder. Preserves config history and evaluation.

20. **`DenyLogsTamper`**
    Blocks Logs delete, KMS (dis)association, resource policy changes, retention changes, and `PutSubscriptionFilter`. Stops log exfil via subscriptions and retention neutering.

21. **`DenyMetricsTamper`**
    Blocks `cloudwatch:DeleteAlarms`, `DisableAlarmActions`, `SetAlarmState`, and `xray:Delete*`. Prevents silent alarm disablement and trace config deletion.

22. **`DenyS3AccountPublicAccessBlockDelete`**
    Keeps **account-level** S3 Public Access Block in place (can’t be removed).

---

## CloudFormation safety rails

35. **`DenyCloudFormationPolicyAndDeletes`**
    Deny destructive StackSet operations (`DeleteStackSet`, `DeleteStackInstances`) and **stack policy** changes (set / set-during-update), plus toggling termination protection. Protects infra managed by CFN from policy-bypass or mass deletions.

---

## CloudShell containment

27. **`DenyCloudShellFileXferAndCredInject`**
    Deny `GetFileDownloadUrls`, `GetFileUploadUrls`, `PutCredentials`. Blocks file exfil/infil and credential seeding. Shell **compute** remains usable per role policy; only the side-channels are blocked.

---

## Security services – disable prevention

28. **`DenySecurityServicesDisable`**
    Prevents weakening/disabling **GuardDuty**, **Inspector2**, and **Security Hub** (including org-admin toggles and detector updates).

29. **`DenyMacieDisableAndOrgAdmin`**
    Prevents disabling **Macie** or changing its org-admin delegation.

**Access Analyzer**

1**7**/**NEW IN DOC ORDERING CALL-OUT**: **`DenyAccessAnalyzerAdministration`**
Blocks analyzer lifecycle/admin (`Create*`, `Delete*`, `Update*`, archive rules, resource scans, tag tamper). Read/list/validation still allowed via role policy.

---

## Eventing, scheduling, catalogs & tagging controls

41. **`DenyEventsAndSchedulerAdministration`**
    Deny EventBridge bus/rule/target admin and permissions changes, and AWS Scheduler create/update/delete/tag. Prevents shadow automation/exfil paths.

42. **`DenyResourceExplorerAndTaggingAdministration`**
    Deny **Resource Explorer v2** default view create/update/delete/associate, **Resource Groups** create/delete/update/tag, and **bulk Tag Editor** (`tag:TagResources`, `tag:UntagResources`). Prevents cross-service tag churn and discovery-plane tamper.

---

## OAM & RAM guardrails

12. **`DenyOamSinkAdministration`**
    Block **OAM sink** create/delete/policy/tag. Protects cross-account telemetry aggregation points.

13. **`DenyOrgWideRAMSharing`**
    Deny org-wide RAM enable/disable toggles. Keeps sharing explicit and governed; per-resource shares live in role policy.

---

## Container/Kubernetes/Registry hardening

7. **`DenyEcsClusterServiceTaskLifecycle`**
   Blocks ECS cluster/service/task lifecycle (create/delete), ad-hoc task runs/starts/stops, task-def register/deregister, and service updates. Prevents drift and unapproved orchestration.

8. **`DenyEksDestructiveOps`**
   Blocks EKS delete operations and control-plane/nodegroup **updates** (config/version). Prevents surprise upgrades and config drift.

9. **`DenyEcrPushDeleteAndPolicyAdmin`**
   Blocks push path (`InitiateLayerUpload`, `UploadLayerPart`, `CompleteLayerUpload`, `PutImage`), bulk deletes, repo delete, and repo policy admin. Pull/read stays allowed via role policy.

---

## Messaging hardening

10. **`DenySnsDestructiveAndPolicyAdmin`**
    Deny topic deletion, permission adds/removals, and `SetTopicAttributes`. Prevents policy/KMS/delivery tamper.

11. **`DenySqsDeleteAndPolicyAdmin`**
    Deny queue deletion, permission adds/removals, and `SetQueueAttributes`. Prevents visibility/retention/policy tamper.

---

## EC2 instance-profile discipline & peering lockdown

36. **`DenyRunInstancesWithoutInstanceProfile`**
    EC2 launch must include an **instance profile** (enforces SSM manageability).

37. **`DenyRunInstancesWithUnapprovedInstanceProfile`**
    EC2 launch must use an **approved** profile (`ApprovedSsmInstanceProfiles` allow-list).

38. **`DenyAssociateOrReplaceUnapprovedInstanceProfile`**
    Block associating/replacing an instance profile unless it’s approved.

39. **`DenyDisassociateInstanceProfile`**
    Block disassociating the instance profile from an instance (keeps fleet manageable).

40. **`DenyVpcPeeringAll`**
    Fully disables VPC peering lifecycle (create/accept/modify/delete). Avoids unsanctioned cross-VPC routing.

---

## What remains under the ceiling

* **Role policies** supply positive grants for network operations (VPC/TGW/RAM/Endpoints/ELB/Route53, etc.).
* Observability **read/emit** continues, but deletes/policy tamper are blocked by the boundary.
* **OAM linking** (not sink admin) can be allowed via role policies.
* **SSM** workflows are allowed via role policies but are enforced by tag-gating and high-risk denials here.

## Notes on Service Quotas

* Quota increase actions are **not** opened by this boundary; place the **allow** for quota families in the **role policy** (keeps boundary size minimal and avoids cross-tower bleed).

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

**Purpose.** This boundary sets a broad ceiling (`Allow "*"`) and then **guarantees non-bypassable guardrails** via explicit `Deny`. It protects identity usage (PassRole), encryption boundaries (KMS/Secrets/SSM), audit/observability integrity, container/cluster registries, messaging policies, SSM hygiene, EC2 instance-profile discipline, VPC peering lockdown, and MFA posture. All actual “allows” for work live in the **role policy**, not here.

**What stays open under the ceiling (when allowed by role policy).**

* Normal day-2 network ops via service APIs (VPC/TGW/ELB/Route 53/Resolver/RAM/etc.).
* Reading/producing logs/metrics/traces without tampering.
* SSM Associations/Maintenance Windows/Automation **without** assumeRole, RunCommand/Session **on tagged targets**.
* OAM **linking** (sink admin remains blocked).
* Container image **pull/read**; no push/delete.
* Service Quotas **reads** (quota increases belong in the role policy).

**What is categorically blocked (high-level).**

* **Identity abuse:** PassRole to anything except **CloudFormation**, and only to CFN **exec-role patterns**; all IAM admin blocked except read/simulate + scoped SLR creation.
* **KMS misuse:** No key/alias/grant/policy/lifecycle admin; no direct Encrypt/ReEncrypt/DataKey except via **Secrets Manager/SSM/S3**; Decrypt only via **Logs/S3/Secrets/SSM**.
  *Exception:* DNSSEC keys listed in `DnssecKmsKeyArns` are excluded from the KMS-admin deny.
* **Secrets Manager tamper:** delete/rotate/resource-policy/replication tamper denied.
* **SSM hygiene:** tag-gated RunCommand/Session; block compliance/inventory/setting tamper; block Automation that specifies an assumeRole; block hybrid activation/MI registration and data-sync tamper.
* **Audit/obs tamper:** CloudTrail delete/stop/scope change; Config delete/recorder/delivery/remediation tamper; Logs delete/retention/policy/KMS and subscription filters; CloudWatch alarm delete/mute/force state; X-Ray delete.
* **Change control:** CFN StackSet deletes/instance removals; stack-policy changes; toggling termination protection.
* **Container/cluster/registry:** ECS lifecycle + ad-hoc runs; EKS destructive/upgrade ops; ECR push/delete/policy admin.
* **Messaging hardening:** SNS topic delete/attr/permission changes; SQS queue delete/attr/permission changes.
* **Cross-account surfaces:** OAM **sink** admin/policy/tag; org-wide RAM **enable/disable** toggles.
* **Platform safety:** Account-level S3 PAB delete; CloudShell file xfer/credential injection; Amazon Q (+Q Business) admin; Detective/Security Lake/Audit Manager admin.
* **Access posture:** MFA required, MFA tag required, MFA age ≤ 3600s.
* **Network egress paths:** VPC peering lifecycle denied.

**Operator quick-reference (when something fails).**

* **PassRole failed?** Ensure the caller is CFN and the target role matches CFN exec-role patterns. Otherwise it’s blocked by boundary—fix by using CFN path.
* **KMS failed?** Run via **Secrets Manager/SSM/S3**. If you need key admin for **DNSSEC**, add the key to `DnssecKmsKeyArns`; anything else is intentionally blocked at the boundary.
* **SSM command/session failed?** Check the target has `NetworkManaged=true`. Automation failing with `assumeRole` → remove the parameter or shift to CFN/IaC.
* **Logs/metrics edits blocked?** That’s by design; manage retention/policies via governed pipelines, not ad-hoc.
* **ECR push blocked?** Use your CI/CD role that’s allowed by **role policy**; humans can’t push from sessions.
* **Quota increase missing?** Add **allow** for `servicequotas:RequestServiceQuotaIncrease` in the **role policy** for relevant services; boundary intentionally stays silent here.

**Why boundary (deny) vs role policy (allow).**

* Boundary is the **last line of defense**—small, stable, non-bypassable.
* Role policy expresses **what a profile can do today** and can evolve frequently.
* By keeping “allows” out of the boundary, you avoid collisions and keep the boundary within AWS size limits.

**Parameters that matter.**

* `ApprovedSsmInstanceProfiles` — enforced by the EC2 launch/association/disassociation denies.
* `DnssecKmsKeyArns` — carve-out for Route 53 DNSSEC KMS admin; without it, DNSSEC ops would be blocked.

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
This summary reflects **all SIDs present** in the current admin boundary and adds **no new allows**. The only scoped exception is the DNSSEC KMS list (`DnssecKmsKeyArns`) to permit controlled DNSSEC key administration.

**KPIs to watch.**

* Count of `AccessDenied` from boundary SIDs (should correlate with attempted bypass, not daily ops).
* CloudTrail for denied `iam:PassRole`, KMS `Encrypt/Decrypt` outside approved `ViaService`, and SSM `StartSession/SendCommand` without tag.
* Drift: alarms/logs/config/trails deletion attempts (should be zero).

**Change rule.**
Any alteration to these `Deny` guardrails requires a security review and documented justification. All new capability should be added to **role policies**, not to the boundary.
