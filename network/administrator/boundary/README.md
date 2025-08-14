# AWS IAM Managed Policy: Permissions Boundary (Network Administrator Role)

## What this is

* A **maximum permissions ceiling** attached via the role’s `PermissionsBoundary`.
* It **does not** grant access alone. Effective permissions = **(role policies ∩ this boundary) − ExplicitDenies**.
* Uses a **broad-allow + explicit-deny guardrails** model: `Allow "*"`, then carve back risk with `Deny` statements.

## Administrator remit (ops over prototyping)

* Day-2 operations, maintenance, diagnostics, and controlled change for **network services** across accounts.
* **No general feature prototyping** for non-network platforms (that remains read-only and/or delegated to other roles).
* Prefer **CloudFormation-driven** changes; direct service API usage is constrained by guardrails.

## Conventions

* Every `Sid` has a clear purpose; **guardrail (Deny)** statements are grouped conceptually here for clean diffs/reviews.
* Single `CustomPolicyPath` for grouping (use `/` to disable).
* **No environment strings** in boundary names; env scoping belongs on roles/policies, not boundaries.

---

## Ceiling & guardrails overview

### Baseline ceiling

* **`GuardrailAllowAll`** – Set the ceiling to `Action: "*"` / `Resource: "*"`. Everything below **carves back** risk.

### Identity & PassRole

* **`DenyPassRoleUnlessCfnService`** – Only allow `iam:PassRole` **when the target service is CloudFormation** (`iam:PassedToService = cloudformation.amazonaws.com`).
* **`DenyPassRoleOutsideCfnExecRoles`** – Even for CloudFormation, the **role being passed** must match CFN execution-role name patterns:
  `role/cfn-*`, `role/cloudformation-*`, `role/service-role/cfn-*`, `role/service-role/cloudformation-*`.
* **`DenyIamAllExceptReadSimulateAndSLR`** – Block IAM admin/mutations. Permit only:

  * `iam:Get*`, `iam:List*`, `iam:Simulate*` and
  * `iam:CreateServiceLinkedRole` (see next guardrail for service allow-list).
* **`DenyCreateServiceLinkedRoleUnlessNetworkServices`** – Allow SLR creation **only** for network-relevant services:
  ELBv2, Global Accelerator, Network Firewall, Route 53 Resolver, VPC Lattice.

### KMS & Secrets boundaries

* **`DenyKmsKeyAdministration`** – Deny key/alias/grant/policy admin and lifecycle.
* **`DenyKmsEncryptAndDataKeyExceptSecretsManager`** – Deny `Encrypt`, `ReEncrypt*`, `GenerateDataKey*` **unless** `kms:ViaService` is
  **Secrets Manager** or **SSM**.
* **`DenyKmsDecryptOutsideApprovedServices`** – Allow `kms:Decrypt` **only when invoked by** `logs.amazonaws.com`, `s3.amazonaws.com`, `secretsmanager.amazonaws.com`, or `ssm.amazonaws.com`.
* **`DenySecretsManagerSensitiveOps`** – Deny deletion/rotation/policy tamper in Secrets Manager. Read/Describe remains under the ceiling via role policy.

### SSM (systems management) boundaries

* **`DenySsmHighRiskAdminOnly`** – Deny **telemetry/compliance tamper** and account-level setting changes:
  `DeleteComplianceItems`, `DeleteInventory`, `PutInventory`, `UpdateInstanceInformation`, `UpdateServiceSetting`.
* **`DenySsmRunWhenTagMissing`** / **`DenySsmRunWhenTagNotTrue`** – Require managed targets:
  `ssm:{SendCommand,StartSession}` **only when** target carries `ssm:resourceTag/NetworkManaged = "true"`.
* **`DenySsmAutomationWithAssumeRole`** – Block SSM Automation executions that specify an **assumeRole** (prevents privilege pivot).
* **`DenySsmHighRiskExtrasTight`** – Additional tightening for high-risk SSM operations beyond the above.
* > **Allowed via role policy (intentional difference vs Architecture boundary):**
  > **Associations, Maintenance Windows, Automation, Parameters**. Boundary blocks only tamper-style/unsafe ops.

### Change-control & observability tamper-proofing

* **`DenyCloudTrailTamper`** – Deny trail deletion/stop/policy tamper (`DeleteTrail`, `StopLogging`, `PutEventSelectors`, `UpdateTrail`).
* **`DenyConfigTamper`** – Deny `config:Delete*`, rule/recorder/delivery-channel create/update/tamper, remediation tamper.
* **`DenyLogsTamper`** – Deny Logs deletes, resource policy/retention changes, KMS association changes, subscription filters (exfil path).
* **`DenyMetricsTamper`** – Deny `cloudwatch:DeleteAlarms`, `DisableAlarmActions`, `SetAlarmState` and `xray:Delete*`.

### CloudFormation safety rails

* **`DenyCloudFormationPolicyAndDeletes`** – Deny StackSet deletions/instance removals, stack policy fiddling, and toggling termination protection.

### CloudShell containment

* **`DenyCloudShellFileXferAndCredInject`** – Deny UI file **upload/download** and `cloudshell:PutCredentials` (exfil/infil prevention).

  > Shell sessions can run via role policy; only file/cred side-channels are blocked.

### Security services – disable prevention

* **`DenySecurityServicesDisable`** – Prevent disabling/weakening: GuardDuty, Inspector2, Security Hub.
* **`DenyMacieDisableAndOrgAdmin`** – Prevent disabling Macie / org admin changes.
* **`DenyAccessAnalyzerAdministration`** – Block Access Analyzer lifecycle/admin.

### Eventing, scheduling, catalogs & tagging controls

* **`DenyEventsAndSchedulerAdministration`** – Block EventBridge rule/target/bus admin, bus permissions, and AWS Scheduler create/update/delete/tag.
* **`DenyResourceExplorerAndTaggingAdministration`** – Block Resource Explorer 2 create/update/delete/default-view changes;
  block Resource Groups create/delete/update and **bulk tag editor** (`tag:TagResources`, `tag:UntagResources`).

### OAM & RAM guardrails

* **`DenyOamSinkAdministration`** – Block OAM **sink** create/delete/policy/tag (prevent cross-account telemetry tamper/exfil).
* **`DenyOrgWideRAMSharing`** – Block org-wide RAM toggles (`Enable/DisableSharingWithAwsOrganization`). Explicit share patterns stay in role policy.

### Container/Kubernetes/Registry ops containment

* **`DenyEcsClusterServiceTaskLifecycle`** – Block ECS cluster/service/task lifecycle (incl. Run/Start/Stop/Update/Register/Deregister).
* **`DenyEksDestructiveOps`** – Block EKS destructive ops and control-plane/nodegroup **updates**.
* **`DenyEcrPushDeleteAndPolicyAdmin`** – Block ECR push path and destructive actions (UploadLayer, PutImage, BatchDeleteImage, repo policy admin, DeleteRepository).

### Messaging hardening

* **`DenySnsDestructiveAndPolicyAdmin`** – Block `DeleteTopic`, `SetTopicAttributes`, and permission changes on topics.
* **`DenySqsDeleteAndPolicyAdmin`** – Block `DeleteQueue`, `SetQueueAttributes`, and permission changes on queues.

### S3 account safety

* **`DenyS3AccountPublicAccessBlockDelete`** – Prevent removal of **Account-level** Public Access Block.

### EC2 instance-profile & peering controls

* **`DenyRunInstancesWithoutInstanceProfile`** – Disallow launching instances **without** an instance profile.
* **`DenyRunInstancesWithUnapprovedInstanceProfile`** – Disallow launching with **non-approved** instance profiles (allow-list is enforced via Conditions).
* **`DenyAssociateOrReplaceUnapprovedInstanceProfile`** – Disallow associating/replacing to a **non-approved** profile.
* **`DenyDisassociateInstanceProfile`** – Disallow disassociating an instance profile from a running instance.
* **`DenyVpcPeeringAll`** – Block VPC peering lifecycle (create/accept/modify/delete).

### Additional service guardrails

* **`DenyAmazonQAdministration`** – Block admin/lifecycle and tag ops across **Amazon Q** and **Q Business**.
* **`DenyDetectiveAdministration`**, **`DenySecurityLakeAdministration`**, **`DenyAuditManagerAdministration`** – Block admin/lifecycle & tag tamper across these services.

### MFA session requirements

* **`DenyRequestsWithoutMFAFlag`** – Deny all requests if `aws:MultiFactorAuthPresent` is false (using `BoolIfExists`).
* **`DenyUnlessSessionTaggedMFA`** – Deny all requests unless the **principal session** carries `aws:PrincipalTag/mfa = "true"`.
* **`DenyIfMFAOlderThanOneHour`** – Deny if `aws:MultiFactorAuthAge > 3600` seconds.

---

## What remains under the ceiling

* Role policies supply positive grants for **network operations** (VPC/TGW/RAM/Endpoints/ELB/Route 53, etc.).
* Observability (read/emit) is available except where **deletes or policy tamper** are explicitly denied.
* **OAM linking** may be permitted via role policy; **sink admin** stays denied at the boundary.
* **SSM** workflows (Associations, MW, Automation, Parameters) can be allowed by role policy; boundary only blocks high-risk/tamper paths.

## Notes on quotas

* This boundary does **not** itself open Service Quotas. If needed, allowlists for **network quota families** (EC2/ELB/Route 53/Resolver/TGW/Direct Connect/Global Accelerator/Network Firewall/VPC Lattice/Network Manager/RAM) should live in the **role policy**.

---

## Testing & change control

### Validate behavior

* Use IAM simulation with real resources and context:

  * PassRole path:

    ```bash
    aws iam simulate-principal-policy \
      --policy-source-arn arn:aws:iam::<acct>:role/<YourAdminRole> \
      --action-names iam:PassRole \
      --resource-arns arn:aws:iam::<acct>:role/cfn-exec-role \
      --context-entries ContextKeyName=iam:PassedToService,ContextKeyType=string,ContextKeyValue=cloudformation.amazonaws.com
    ```
  * KMS ViaService:

    ```bash
    aws iam simulate-principal-policy \
      --policy-source-arn arn:aws:iam::<acct>:role/<YourAdminRole> \
      --action-names kms:Decrypt \
      --resource-arns arn:aws:kms:<region>:<acct>:key/<key-id> \
      --context-entries ContextKeyName=kms:ViaService,ContextKeyType=string,ContextKeyValue=logs.amazonaws.com
    ```
* Confirm MFA conditions by testing with and without the `mfa=true` principal tag and by observing `aws:MultiFactorAuthAge`.

### Change management

* Treat **Deny** blocks as **stable guardrails**; any change requires security review + written justification.
* Keep a **diff log** between managed policy versions; review impact with `simulate-principal-policy`.
* Monitor managed policy **size limits**. Prefer category-level denies over long verb lists to stay within limits.

---

## Parameters & outputs (template alignment)

* **Parameters**

  * `PermissionsBoundaryName` – Managed policy name (default: `network-administrator-permissions-boundary`).
  * `CustomPolicyPath` – IAM path for grouping (default: `/managed/network/`).
* **Outputs**

  * `NetworkAdministratorPermissionsBoundaryArn` – Reference to the boundary managed policy ARN for downstream stacks/automation.

---

## Quick SID → purpose map

* `GuardrailAllowAll` – Set broad ceiling before carving back risk.

* `DenyAccessAnalyzerAdministration` – Block Access Analyzer lifecycle/admin.
* `DenyAmazonQAdministration` – Block Amazon Q / Q Business administration.
* `DenyAssociateOrReplaceUnapprovedInstanceProfile` – Block associating/replacing to **non-approved** profiles.
* `DenyAuditManagerAdministration` – Block Audit Manager admin/lifecycle.
* `DenyCloudFormationPolicyAndDeletes` – Block StackSet deletes/policy fiddling/termination protection toggles.
* `DenyCloudShellFileXferAndCredInject` – Block CloudShell file xfer & credential injection.
* `DenyCloudTrailTamper` – Block CloudTrail tamper.
* `DenyConfigTamper` – Block AWS Config tamper.
* `DenyCreateServiceLinkedRoleUnlessNetworkServices` – SLR creation allow-list for network services.
* `DenyDetectiveAdministration` – Block Detective admin/lifecycle.
* `DenyDisassociateInstanceProfile` – Block disassociating instance profiles.
* `DenyEcrPushDeleteAndPolicyAdmin` – Block ECR push/delete/policy admin.
* `DenyEcsClusterServiceTaskLifecycle` – Block ECS cluster/service/task lifecycle and ad-hoc runs.
* `DenyEksDestructiveOps` – Block EKS destructive ops and upgrades.
* `DenyEventsAndSchedulerAdministration` – Block EventBridge/Scheduler admin.
* `DenyIamAllExceptReadSimulateAndSLR` – IAM read/simulate + SLR only.
* `DenyIfMFAOlderThanOneHour` – Enforce MFA max age of **1 hour**.
* `DenyKmsDecryptOutsideApprovedServices` – Allow decrypt **only via** Logs/S3/Secrets Manager/SSM.
* `DenyKmsEncryptAndDataKeyExceptSecretsManager` – Allow KMS encrypt/datakey **only via** Secrets Manager or SSM.
* `DenyKmsKeyAdministration` – Block KMS admin & lifecycle.
* `DenyLogsTamper` – Block Logs tamper/exfil paths.
* `DenyMacieDisableAndOrgAdmin` – Prevent disabling Macie / org admin tamper.
* `DenyMetricsTamper` – Block alarm deletes/mutes/state forcing and X-Ray deletes.
* `DenyOamSinkAdministration` – Block OAM sink admin/policy/tag.
* `DenyOrgWideRAMSharing` – Block org-wide RAM sharing toggles.
* `DenyPassRoleOutsideCfnExecRoles` – Pass only **CFN exec roles** (name-pattern bound).
* `DenyPassRoleUnlessCfnService` – PassRole only to **CloudFormation**.
* `DenyRequestsWithoutMFAFlag` – Deny if `MultiFactorAuthPresent=false`.
* `DenyResourceExplorerAndTaggingAdministration` – Block Resource Explorer/Groups admin & **bulk** Tag Editor.
* `DenyRunInstancesWithUnapprovedInstanceProfile` – Require an **approved** instance profile.
* `DenyRunInstancesWithoutInstanceProfile` – Require instance profile on launch.
* `DenyS3AccountPublicAccessBlockDelete` – Keep account-level PAB intact.
* `DenySecretsManagerSensitiveOps` – Block Secrets Manager delete/rotate/policy tamper.
* `DenySecurityLakeAdministration` – Block Security Lake admin/lifecycle.
* `DenySecurityServicesDisable` – Prevent disabling GuardDuty/Inspector2/SecurityHub.
* `DenySnsDestructiveAndPolicyAdmin` – Block SNS topic delete/policy/attr changes.
* `DenySqsDeleteAndPolicyAdmin` – Block SQS queue delete/policy/attr changes.
* `DenySsmAutomationWithAssumeRole` – Block Automation executions that specify an assume role.
* `DenySsmHighRiskAdminOnly` – Block SSM compliance/telemetry tamper & account-level setting changes.
* `DenySsmHighRiskExtrasTight` – Extra SSM hardening.
* `DenySsmRunWhenTagMissing` / `DenySsmRunWhenTagNotTrue` – Enforce `NetworkManaged=true` on SSM Run/Session targets.
* `DenyUnlessSessionTaggedMFA` – Require `PrincipalTag/mfa=true`.
* `DenyVpcPeeringAll` – Block VPC peering lifecycle.

---

### Summary

This boundary **keeps the administrator powerful for network operations** while enforcing non-bypassable controls on identity usage (`PassRole`), encryption boundaries (KMS/Secrets/SSM), security/observability tamper, eventing/scheduling sprawl, cross-account telemetry sinks, registry/container/k8s lifecycle, strict SSM target gating, EC2 instance-profile discipline, VPC-peering lockdown, and mandatory MFA posture. It matches the template SIDs and intent without dropping any guardrail.
