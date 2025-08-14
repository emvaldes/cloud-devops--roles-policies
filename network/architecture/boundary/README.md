# AWS IAM Managed Policy: Permissions Boundary (Network Architecture Role)

## What this is

* A **maximum permissions ceiling** for any role that references it via `Role.Properties.PermissionsBoundary`.
* It does **not** grant access by itself; effective permissions are: **(attached policies ∩ this boundary) − ExplicitDenies**.
* **Explicit Deny** here precedes and overrides any **Allow** in attached policies.
* Must be attached on the **Role** (`PermissionsBoundary` property).

## Goals (network ownership)

* Full prototyping surface for network platforms (VPC/TGW/RAM/Endpoints/LB/etc.).
* **Route 53 – full scope**: authoritative DNS **plus** registrar, ARC, DNS Profiles, and zonal shift.
* **ACM admin** for certificate lifecycle supporting ELB/GA/CloudFront integrations.
* Internet & VPC Network Monitor **admin** (traffic/latency/availability visibility).
* RAM admin for explicit cross-account sharing (**no** org-wide toggle).
* Messaging for workflows (SNS/SQS create/manage).
* Support case management (`support:*`), Health and Trusted Advisor visibility.
* Observability & audit (CloudWatch/Logs/CloudTrail/X-Ray/Config), OAM linking.
* Security services visibility (SecurityHub/GuardDuty/Inspector2/Macie **RO**), Detective **admin**, Security Lake **RO**, Audit Manager **admin**.

## Conventions

* Every Sid is commented with purpose and whether it’s a grant or guardrail.
* Single `CustomPolicyPath` used for grouping (set `/` to disable).
* No environment strings in names; env lives in **roles**, not the boundary.
* Keep all **guardrail (Deny)** statements grouped at the end for audit diffs.
* Avoid overlapping read-only catchalls with full-access blocks to reduce noise.

## Guardrails (non-bypassable)

* **IAM limited to Get/List/PassRole/SLR** for prototyping (no general IAM mutations in the boundary).
* `PassRole` allowed **only** when the **target role** is tagged `PassRole=true`.
* **MFA required** on all requests; session must carry `PrincipalTag mfa=true`.
* Optional: deny if `aws:MultiFactorAuthAge > 3600`.

## Messaging policy (ceiling)

* SNS/SQS provisioning/ops **allowed** (no tag gate in this boundary).

## Service Quotas (canonical scope)

* Allowed to request increases **only** for:
  `ec2`, `elasticloadbalancing`, `route53`, `route53resolver`, `directconnect`, `globalaccelerator`, `network-firewall`, `vpc-lattice`, `networkmanager`, `ram`.
* Do **not** edit this list without explicit change approval.

## Testing & change control

* Validate ceilings with: `aws iam simulate-principal-policy` (use real ARNs where conditions/tags apply).
* Treat **Deny** blocks as stable guardrails; changes require security review + recorded justification.
* Compare managed-policy actions vs. boundary allows to detect drift.
