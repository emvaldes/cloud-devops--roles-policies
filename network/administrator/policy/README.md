# AWS IAM Managed Policy: **Network Administrator (Operations)**

## What this is

* A **managed policy** attached to the **Network Administrator role** that grants **positive permissions** for day-2 network operations.
* Works **together** with the **permissions boundary** (broad-allow + explicit-deny guardrails).
  Effective permissions = **(this policy’s Allows ∩ boundary ceiling) − boundary Denies**.

## Operating scope (admin over network platforms; read-mostly elsewhere)

* Full **network control-plane operations** (VPC/TGW/Endpoints/ELB/GA/NFW/Resolver/Lattice/RAM).
* **CloudFormation-first** changes; direct API is allowed but bounded by guardrails.
* **Read-only** posture for security services and most non-network platforms (containers/K8s/artifacts = RO; edge/WAF/Shield = RO).

## Conventions

* Every statement has a **`Sid`** and clear purpose.
* **No denies here** (denies live in the boundary).
* Resource scoping where applicable (e.g., `network-*` buckets, `network-*` SSM docs).
* CLI-safe: uses conditions that match boundary gates (e.g., CFN-only `PassRole`, SSM tag requirements).

---

## Major capabilities (by service)

### CloudFormation & CI/CD

* **`CloudFormationDeploy`** — Create/Update stacks, change sets, drift detection, read/list/get/cancel/continue rollback.
* **`CodePipelineOperations`** — Get/List, retry executions, **approve/reject** Manual Approval, put job success/failure.
* **`CodeBuildOperations`** — BatchGet/List, **StartBuild/StopBuild/RetryBuild**.
* **`CodeDeployOperations`** — Get/List, **CreateDeployment/StopDeployment**.
* **`CodeCommitReadOnly`** — Get/List, **GitPull** (clone/fetch).

### Identity & PassRole (aligned with boundary)

* **`PassRoleForCloudFormationExecRoles`** — `iam:PassRole` to CFN **execution roles only**:

  * Resources: `arn:aws:iam::*:role/{cfn-*,cloudformation-*,service-role/cfn-*,service-role/cloudformation-*}`
  * Condition: `iam:PassedToService = cloudformation.amazonaws.com`
    (Boundary enforces the same gate; both must be satisfied.)

### S3 (network buckets)

* **`S3ListAllMyBuckets`** — Enumerate buckets (inventory/selection).
* **`S3NetworkBucketsAdminSafe`** — Create/manage **`network-*`** buckets; enforce PAB, ownership controls, versioning, default SSE, tagging, location, list.
* **`S3BucketPolicyViaCloudFormationOnly`** — Get/Put/Delete **bucket policy** on `network-*` **only when called via CloudFormation**
  (`ForAnyValue:StringEquals aws:CalledVia = cloudformation.amazonaws.com`).
* **`S3NetworkObjectsStrict`** — CRUD objects under `arn:aws:s3:::network-*/*` with **required** request headers:

  * `s3:x-amz-acl = bucket-owner-full-control`
  * `s3:x-amz-server-side-encryption ∈ {AES256, aws:kms}`
  * If KMS: key must match `arn:aws:kms:*:*:alias/network-*`

### Core networking admin

* **`NetworkingAdmin`** — `ec2:*`, `elasticloadbalancing:*`, `globalaccelerator:*`, `network-firewall:*`, `networkmanager:*`, `ram:*`, `route53resolver:*`, `vpc-lattice:*` (full admin across core network services).
* **`DirectConnectReadOnly`** — Describe/View only (no lifecycle).
* **`Route53AndARCAdmin`** — Full Route 53 (hosted zones/records/health checks/traffic policies) and **ARC** cluster/zonal shift APIs.
* **`CertificateManagerAdmin`** — `acm:*` (issue/import/renew/delete).

### Edge, containers, artifacts (read-only)

* **`EdgeProtectionReadOnly`** — `wafv2:Get/List*`, `shield:Describe/List*`.
* **`ContainerAndArtifactReadOnly`** — CodeArtifact Get/List/ReadFromRepository; ECR Describe/List/BatchGetImage/GetAuthorizationToken; ECS/EKS Describe/List.

### Service discovery & catalog (read-only)

* **`ServiceDiscoveryAndCatalogReadOnly`** — Cloud Map Get/List; Service Catalog Get/List/Describe/Search.

### Observability (read/queries, no tamper)

* **`LogsReadAndQuery`** — Logs Describe/Get/FilterLogEvents/StartQuery/GetQueryResults/List.
* **`CloudWatchReadOnly`** — Describe/Get/ List (dashboards/alarms/metrics).
* **`CloudTrailReadOnly`** — Describe/LookupEvents/GetTrailStatus/List\*.
* **`ConfigReadOnly`** — BatchGet/Get\*/List\* (resources/rules/config/history).
* **`XRayReadOnly`** — Get/BatchGet/List (traces/insights).

### Systems Manager (SSM)

* **`SsmCoreOps`** — Describe/List/Get, **Start/Resume/TerminateSession**, **Send/CancelCommand**, **GetCommandInvocation**
  with **conditions**:

  * Target must have `ssm:resourceTag/NetworkManaged = "true"`.
  * Allowed documents: `AWS-RunShellScript`, `AWS-RunPowerShellScript`, and `network-*`.
* **`SsmAssocAndMW`** — `ssm:*Association`, `ssm:*MaintenanceWindow*` (create/update/delete/register/describe/list).
* **`SsmParametersNetworkPath`** — `ssm:*Parameter*` under `arn:aws:ssm:*:*:parameter/network/*`.
* **`SsmDocsNetwork`** — `ssm:*Document*` **only** on `arn:aws:ssm:*:*:document/network-*`.
* **`SsmDocsRead`** — Get/Describe/List/Versions (any doc).
* **`SsmAutomationStartNetworkDocsOnly`** — `ssm:StartAutomationExecution` for `arn:aws:ssm:*:*:document/network-*` **only if**
  `Null ssm:AutomationAssumeRole = true` (no assumeRole pivot).
* **`SsmAutomationReadResults`** — Get/List/Describe automation executions/step results.

### Analysis & eventing

* **`NetworkReachabilityAnalysis`** — Create/Start/Describe/Delete **VPC Reachability Analyzer** paths/analyses.
* **`EventsAndSchedulerReadOnly`** — EventBridge Get/Describe/List & AWS Scheduler Get/Describe/List.
* **`EventBridgePipesReadOnly`** — Pipes Describe/List/ListTagsForResource.

### Identity discovery & SLR (writes limited)

* **`IdentityDiscoveryAndSLR`** — `iam:Get*/List*`, **`iam:CreateServiceLinkedRole`** (boundary allow-lists network services only).

### KMS + Secrets (read/use; admin constrained by boundary)

* **`KeyAndSecretReadUse`** — KMS Describe/List/GetKeyPolicy/Decrypt/Encrypt/GenerateDataKey/WithoutPlaintext;
  Secrets Manager Get/Describe/List (boundary gates KMS **ViaService** and Secrets **sensitive ops**).

### Discovery, tagging, and OAM

* **`DiscoveryAndTagReadOnly`** — Resource Explorer 2 Search/Get/List; Resource Groups Get/List.
* **`ObservabilityAccessManagerLinking`** — OAM Create/Delete/UpdateLink + Get/List (sink admin remains denied by boundary).

### Messaging (incident-friendly)

* **`MessagingPublishConsume`** — SNS Publish/Get/List/ConfirmSubscription; SQS Send/Receive/GetQueueAttributes/GetQueueUrl/ListQueues.
* **`MessagingOpsMinimalWrites`** — SNS **Subscribe/Unsubscribe** and SQS **DeleteMessage/ChangeMessageVisibility**
  **scoped to**:

  * `arn:aws:sns:*:*:network-*` (and `network-*: *`)
  * `arn:aws:sqs:*:*:network-*`

### Resource sharing (RAM)

* **`RamNetworkSharesAdmin`** — Create/Update/Delete/Associate/Disassociate shares & perms; Get/List; Tag/Untag.
* **`RamPermissionAuthoring`** — Create/Update/Delete/Version/Promote permissions; Replace associations; Get/List.

### Quotas (network-only)

* **`ServiceQuotasNetworkScoped`** — Get/List and **RequestServiceQuotaIncrease**
  with `servicequotas:service` restricted to:
  `ec2, elasticloadbalancing, route53, route53resolver, directconnect, globalaccelerator, network-firewall, vpc-lattice, networkmanager, ram`.

### Security & support (read-mostly)

* **`SecurityServicesReadOnly`** — GuardDuty/Inspector2/Security Hub Get/Describe/List.
* **`HealthReadOnly`** — `health:Describe*`.
* **`TrustedAdvisorReadOnly`** — `trustedadvisor:Describe*/List*`.
* **`SupportFull`** — `support:*` (case management).
* **`InternetAndNetworkMonitorAdmin`** — `internetmonitor:*`, `networkmonitor:*` (service admins for network observability).
* **`FirewallManagerReadOnly`** — `fms:Get*/List*`.
* **`MacieReadOnly`** — `macie2:Describe*/Get*/List*`.
* **`SecurityLakeReadOnly`** — `securitylake:Get*/List*`.
* **`DetectiveReadOnly`** — `detective:Describe*/Get*/List*`.
* **`AuditManagerReadOnly`** — `auditmanager:Describe*/Get*/List*`.
* **`AccessAnalyzerReadOnlyAndValidation`** — `access-analyzer:Get*/List*`, `ValidatePolicy`.

---

## Interplay with the Permissions Boundary (heads-up)

* **CloudShell**: This policy allows `cloudshell:*`; the boundary **blocks** file upload/download and `PutCredentials`.
* **VPC peering**: Network admin here is broad; the boundary **denies** VPC peering lifecycle.
* **ECS/EKS/ECR**: Reads allowed here; boundary **denies** lifecycle/push/admin.
* **Logs/CloudWatch/Trail/Config**: Reads/queries allowed; boundary **denies** deletes/policy tamper/retention/KMS changes.
* **KMS/Secrets**: Read/use here; boundary **restricts** KMS to **ViaService** paths and **denies** Secrets sensitive ops.
* **`PassRole`**: Must be CFN-targeted and to CFN exec roles (enforced by both policy and boundary).
* **SSM**: Sessions/RunCommand allowed here but **only** on targets with `NetworkManaged=true`; Automation **must not** use `assumeRole`.

---

## Parameters & Outputs (template alignment)

* **Parameters**

  * `ManagedPolicyName` — Default `network-administrator-policy`.
  * `CustomPolicyPath` — IAM path (default `/managed/network/`).
* **Outputs**

  * `ManagedPolicyArn` — ARN of the managed policy (`NetworkAdministratorManagedPolicy`).

---

## Quick SID → purpose map (1:1 with template)

* **Cloud/CI/CD**: `CloudFormationDeploy`, `CodePipelineOperations`, `CodeBuildOperations`, `CodeDeployOperations`, `CodeCommitReadOnly`
* **Identity**: `PassRoleForCloudFormationExecRoles`, `IdentityDiscoveryAndSLR`
* **S3**: `S3ListAllMyBuckets`, `S3NetworkBucketsAdminSafe`, `S3BucketPolicyViaCloudFormationOnly`, `S3NetworkObjectsStrict`
* **Shell**: `CloudShellFull`
* **Networking**: `NetworkingAdmin`, `DirectConnectReadOnly`, `Route53AndARCAdmin`, `CertificateManagerAdmin`
* **Edge/Containers**: `EdgeProtectionReadOnly`, `ContainerAndArtifactReadOnly`
* **Discovery/Catalog**: `ServiceDiscoveryAndCatalogReadOnly`
* **Observability**: `LogsReadAndQuery`, `CloudWatchReadOnly`, `CloudTrailReadOnly`, `ConfigReadOnly`, `XRayReadOnly`
* **SSM**: `SsmCoreOps`, `SsmAssocAndMW`, `SsmParametersNetworkPath`, `SsmDocsNetwork`, `SsmDocsRead`, `SsmAutomationStartNetworkDocsOnly`, `SsmAutomationReadResults`
* **Analysis/Eventing**: `NetworkReachabilityAnalysis`, `EventsAndSchedulerReadOnly`, `EventBridgePipesReadOnly`
* **KMS/Secrets**: `KeyAndSecretReadUse`
* **Discovery/Tags/OAM**: `DiscoveryAndTagReadOnly`, `ObservabilityAccessManagerLinking`
* **Messaging**: `MessagingPublishConsume`, `MessagingOpsMinimalWrites`
* **RAM**: `RamNetworkSharesAdmin`, `RamPermissionAuthoring`
* **Quotas**: `ServiceQuotasNetworkScoped`
* **Security/Support**: `SecurityServicesReadOnly`, `HealthReadOnly`, `TrustedAdvisorReadOnly`, `SupportFull`,
  `InternetAndNetworkMonitorAdmin`, `FirewallManagerReadOnly`, `MacieReadOnly`, `SecurityLakeReadOnly`,
  `DetectiveReadOnly`, `AuditManagerReadOnly`, `AccessAnalyzerReadOnlyAndValidation`

---

## Validation snippets (targeted)

> Replace `<acct>`, `<region>`, `<role>`, `<bucket>`, `<key>`, `<doc>`.

**PassRole to CFN exec role**

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<acct>:role/<role> \
  --action-names iam:PassRole \
  --resource-arns arn:aws:iam::<acct>:role/cfn-exec-role \
  --context-entries ContextKeyName=iam:PassedToService,ContextKeyType=string,ContextKeyValue=cloudformation.amazonaws.com
```

**SSM RunCommand gated by tag + allowed docs**

```bash
aws ssm send-command \
  --instance-ids i-0123456789abcdef0 \
  --document-name AWS-RunShellScript \
  --parameters commands='["echo ok"]'
# Expect ALLOW only if instance has tag NetworkManaged=true
```

**SSM Automation without assumeRole (network doc only)**

```bash
aws ssm start-automation-execution \
  --document-name network-maintenance-foo \
  --parameters key=value
# Fails if you pass --mode or --parameters that imply AutomationAssumeRole
```

**S3 PutObject with required headers**

```bash
aws s3api put-object \
  --bucket network-ops-bucket \
  --key test.txt \
  --body ./test.txt \
  --acl bucket-owner-full-control \
  --server-side-encryption AES256
# For KMS: add --sse aws:kms --ssekms-key-id arn:aws:kms:<region>:<acct>:alias/network-keys
```

---

### Summary

This policy gives the **Network Administrator** everything needed to **operate** AWS networking at scale while cleanly deferring risk controls to the **permissions boundary** (identity, KMS/Secrets, observability tamper-proofing, CloudShell exfil, peering, container lifecycle, etc.). It is aligned with the template exactly — no omissions, with precise resource and condition scopes where required.
