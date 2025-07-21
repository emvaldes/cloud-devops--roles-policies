# AWS IAM Managed Policy: **Network Administrator Policy** (Operations)

## What this is

* A **managed policy** attached to the Network Administrator role that provides **positive Allows** for day-2 network operations.
* Effective permissions = **(this policy’s Allows ∩ the permissions boundary ceiling) − boundary ExplicitDenies**.
* Pairs with a boundary that uses **broad-allow + explicit-deny guardrails**. This policy contains **no Deny**.

## Administrator remit (ops over prototyping)

* Day-2 operations, maintenance, diagnostics, and controlled change for **network services** across accounts.
* **No general feature prototyping** for non-network platforms (that remains read-only and/or delegated to other roles).
* Prefer **CloudFormation-driven** changes; direct service API usage is bounded by the boundary’s guardrails.

## Conventions

* Every statement has a **`Sid`**; resource scopes and conditions are explicit where applicable.
* Aligns with boundary gates (e.g., **CFN-only** `PassRole`, **SSM tag** requirement).
* Uses a single `CustomPolicyPath` for grouping (use `/` to disable). **No environment strings** in names.

---

## Ceiling & guardrails overview

> This policy grants capabilities that **operate beneath** the boundary’s ceiling. Where the boundary denies (CloudShell file xfer, VPC peering, ECR pushes, ECS/EKS lifecycle, etc.), those paths remain **blocked even if allowed nearby here**.

### Baseline capability set

* Operational **Describe/List/Get** across network planes and supporting observability to enable administration.

### Identity & PassRole

* **`PassRoleForCloudFormationExecRoles`** — `iam:PassRole` **only** to CFN execution roles
  (`role/{cfn-*,cloudformation-*,service-role/cfn-*,service-role/cloudformation-*}`) with
  `iam:PassedToService = cloudformation.amazonaws.com`.
* **`IdentityDiscoveryAndSLR`** — `iam:Get*/List*` and **`iam:CreateServiceLinkedRole`**
  (the boundary allow-lists network services only).

### KMS & Secrets boundaries

* **`KeyAndSecretReadUse`** — KMS Describe/List/Encrypt/Decrypt/GenerateDataKey(WithoutPlaintext);
  Secrets Manager Get/Describe/List.
  *Note*: Boundary enforces **ViaService** constraints and blocks Secrets destructive/policy ops.

### SSM (systems management) boundaries

* **`SsmCoreOps`** — Describe/List/Get, **Start/Resume/TerminateSession**, **Send/CancelCommand**, **GetCommandInvocation**;
  **only when** targets carry `NetworkManaged=true`; allowed docs: `AWS-RunShellScript`, `AWS-RunPowerShellScript`, `network-*`.
* **`SsmAssocAndMW`** — Manage **Associations** and **Maintenance Windows** (create/update/delete/register/describe).
* **`SsmParametersNetworkPath`** — Manage **Parameters** under `arn:aws:ssm:*:*:parameter/network/*`.
* **`SsmDocsNetwork`** — Manage **Documents** matching `arn:aws:ssm:*:*:document/network-*`.
* **`SsmDocsRead`** — Read/Describe/List/Versions (any doc).
* **`SsmAutomationStartNetworkDocsOnly`** — Start Automation on `network-*` docs **only without** an `assumeRole`.
  Boundary blocks Automation with assumeRole.
* **`SsmAutomationReadResults`** — Read/List/Describe automation executions and step results.

### Change-control & observability tamper-proofing

* **Read/query only** (no tamper):
  **`LogsReadAndQuery`**, **`CloudWatchReadOnly`**, **`CloudTrailReadOnly`**, **`ConfigReadOnly`**, **`XRayReadOnly`**.
  *Note*: Boundary denies deletes, retention/policy/KMS/subscription tamper.

### CloudFormation safety rails

* **`CloudFormationDeploy`** — Create/Update stacks & change sets, drift, describe/list/get, cancel/continue rollback.
* CI/CD helpers: **`CodePipelineOperations`**, **`CodeBuildOperations`**, **`CodeDeployOperations`**, **`CodeCommitReadOnly`**.

### CloudShell containment

* **`CloudShellFull`** — Shell runtime access for operators.
  *Note*: Boundary **denies** file upload/download and `cloudshell:PutCredentials`.

### Security services – disable prevention

* **Read-mostly posture**: **`SecurityServicesReadOnly`**, **`MacieReadOnly`**, **`SecurityLakeReadOnly`**,
  **`DetectiveReadOnly`**, **`AuditManagerReadOnly`**, **`FirewallManagerReadOnly`**,
  **`AccessAnalyzerReadOnlyAndValidation`**.
  *Note*: Boundary prevents disable/admin paths.

### Eventing, scheduling, catalogs & tagging controls

* **`EventsAndSchedulerReadOnly`**, **`EventBridgePipesReadOnly`**, **`ServiceDiscoveryAndCatalogReadOnly`** (read/visibility).
  *Note*: Boundary denies admin create/update/delete/tag on these planes.

### OAM & RAM guardrails

* **`ObservabilityAccessManagerLinking`** — Manage **Links** (sink admin stays denied by boundary).
* **`RamNetworkSharesAdmin`**, **`RamPermissionAuthoring`** — Administer shares/perms for network resources
  (org-wide toggles denied by boundary).

### Container/Kubernetes/Registry ops containment

* **`ContainerAndArtifactReadOnly`** — Read/list across ECR/ECS/EKS/CodeArtifact.
  *Note*: Boundary denies lifecycle/push/admin.

### Messaging hardening

* **`MessagingPublishConsume`** — SNS/SQS publish/consume.
* **`MessagingOpsMinimalWrites`** — SNS subscribe/unsubscribe; SQS DeleteMessage/ChangeMessageVisibility, **scoped** to `network-*`.
  *Note*: Topic/queue policy and destructive controls remain denied by boundary.

### S3 account safety

* **Network buckets & objects**:
  **`S3ListAllMyBuckets`**; **`S3NetworkBucketsAdminSafe`** (versioning/SSE/PAB-friendly settings);
  **`S3BucketPolicyViaCloudFormationOnly`** (bucket policy CRUD on `network-*` **only via CFN**);
  **`S3NetworkObjectsStrict`** (object CRUD under `arn:aws:s3:::network-*/*` with required ACL/SSE, KMS alias `alias/network-*` if KMS).

### EC2 instance-profile & peering controls

* **(Informational)** — This policy does not enforce profile gating or peering; the **boundary** carries:
  *DenyRunInstancesWithoutInstanceProfile*, *DenyRunInstancesWithUnapprovedInstanceProfile*,
  *DenyAssociateOrReplaceUnapprovedInstanceProfile*, *DenyDisassociateInstanceProfile*, *DenyVpcPeeringAll*.
  Your **`NetworkingAdmin`** SID provides EC2/ELB/GA/NFW/Resolver/Lattice/RAM admin under those guardrails.

### Additional service guardrails

* **(Informational)** — Admin/lifecycle blocks for Amazon Q / OAM sink / Detective / Security Lake / Audit Manager live in the **boundary**.
  Here you have **read or link-level** capabilities as listed above.

### MFA session requirements

* **(Informational)** — Enforced by the **boundary** (require MFA present, `PrincipalTag/mfa=true`, max age 1h).

---

## What remains under the ceiling

* Anything **not allowed** here remains inaccessible.
* Where this policy **allows** but the boundary **denies** (CloudShell file xfer, VPC peering, ECR push, ECS/EKS lifecycle, logs/admin tamper, etc.), the **deny wins**.
* **PassRole** requires both CFN target and an approved CFN exec-role name pattern (enforced by both).

## Notes on quotas

* **`ServiceQuotasNetworkScoped`** limits increases to network families
  (`ec2, elasticloadbalancing, route53, route53resolver, directconnect, globalaccelerator, network-firewall, vpc-lattice, networkmanager, ram`).
* Keep this policy **service-scoped**; leave risk controls in the **boundary**.

---

## Testing & change control

### Validate behavior

* **PassRole to CFN exec**

  ```bash
  aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::<acct>:role/<AdminRole> \
    --action-names iam:PassRole \
    --resource-arns arn:aws:iam::<acct>:role/cfn-exec-role \
    --context-entries ContextKeyName=iam:PassedToService,ContextKeyType=string,ContextKeyValue=cloudformation.amazonaws.com
  ```
* **SSM tag-gate**

  ```bash
  aws ssm send-command \
    --instance-ids i-0123456789abcdef0 \
    --document-name AWS-RunShellScript \
    --parameters commands='["echo ok"]'
  # Allowed only if instance has tag NetworkManaged=true
  ```
* **S3 put with required headers**

  ```bash
  aws s3api put-object \
    --bucket network-ops-bucket --key test.txt --body ./test.txt \
    --acl bucket-owner-full-control --server-side-encryption AES256
  ```

### Change management

* Keep a **diff log** for each managed policy version; review with `simulate-principal-policy`.
* Avoid scope creep; **non-network prototyping** belongs to other roles.

---

## Parameters & outputs (template alignment)

* **Parameters**

  * `ManagedPolicyName` — default `network-administrator-policy`.
  * `CustomPolicyPath` — default `/managed/network/`.
* **Outputs**

  * `ManagedPolicyArn` — ARN of the managed policy.

---

## Quick SID → purpose map (complete, 54/54)

**Cloud/CI/CD**

* `CloudFormationDeploy`, `CodePipelineOperations`, `CodeBuildOperations`, `CodeDeployOperations`, `CodeCommitReadOnly`

**Identity**

* `PassRoleForCloudFormationExecRoles`, `IdentityDiscoveryAndSLR`

**S3**

* `S3ListAllMyBuckets`, `S3NetworkBucketsAdminSafe`, `S3BucketPolicyViaCloudFormationOnly`, `S3NetworkObjectsStrict`

**Shell**

* `CloudShellFull`

**Networking**

* `NetworkingAdmin`, `DirectConnectReadOnly`, `Route53AndARCAdmin`, `CertificateManagerAdmin`

**Edge/Containers/Artifacts**

* `EdgeProtectionReadOnly`, `ContainerAndArtifactReadOnly`

**Discovery/Catalog**

* `ServiceDiscoveryAndCatalogReadOnly`

**Observability**

* `LogsReadAndQuery`, `CloudWatchReadOnly`, `CloudTrailReadOnly`, `ConfigReadOnly`, `XRayReadOnly`

**SSM**

* `SsmCoreOps`, `SsmAssocAndMW`, `SsmParametersNetworkPath`, `SsmDocsNetwork`, `SsmDocsRead`,
  `SsmAutomationStartNetworkDocsOnly`, `SsmAutomationReadResults`

**Analysis/Eventing**

* `NetworkReachabilityAnalysis`, `EventsAndSchedulerReadOnly`, `EventBridgePipesReadOnly`

**KMS/Secrets**

* `KeyAndSecretReadUse`

**Discovery/Tags/OAM**

* `DiscoveryAndTagReadOnly`, `ObservabilityAccessManagerLinking`

**Messaging**

* `MessagingPublishConsume`, `MessagingOpsMinimalWrites`

**RAM**

* `RamNetworkSharesAdmin`, `RamPermissionAuthoring`

**Quotas**

* `ServiceQuotasNetworkScoped`

**Security/Support**

* `SecurityServicesReadOnly`, `HealthReadOnly`, `TrustedAdvisorReadOnly`, `SupportFull`,
  `InternetAndNetworkMonitorAdmin`, `FirewallManagerReadOnly`, `MacieReadOnly`, `SecurityLakeReadOnly`,
  `DetectiveReadOnly`, `AuditManagerReadOnly`, `AccessAnalyzerReadOnlyAndValidation`

---

### Summary

This README mirrors your boundary doc’s structure and fully documents the **administrator policy**.
All **54 SIDs** are included and categorized; interplay with the boundary is explicitly called out where relevant.
