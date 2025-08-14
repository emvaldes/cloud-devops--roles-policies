# AWS IAM Managed Policy: Network Architecture Policy (Network Architecture Role)

## What this is

* IAM managed policy attached to the **NetworkArchitecture** role.
* Effective permissions = **(this policy ∩ the role’s permissions boundary) − explicit denies**.

## Key capabilities granted (aligned to boundary)

* **IaC & artifacts:** CloudFormation, Code\*, S3, CloudShell.
* **Networking:** EC2 VPC/TGW/NAT/IGW/Endpoints, ELBv2, Route 53 — **full scope** (authoritative + Domains/ARC/Profiles/Zonal Shift), Route 53 Resolver, **ACM**, Direct Connect, Network Firewall, VPC Lattice, Global Accelerator.
* **Containers & artifacts:** ECR, ECS, EKS, CodeArtifact.
* **Observability & audit:** CloudWatch, Logs, CloudTrail, X-Ray, Config, OAM linking.
* **Ops & integration:** SSM/SSM Messages/EC2 Messages, EventBridge, Scheduler, Pipes, RAM admin.
* **Discovery:** Resource Explorer v2, Resource Groups, tagging, `ListRegions`, Orgs list.
* **Security (read-only):** Security Hub, GuardDuty, Inspector2, Macie, Security Lake, FMS RO, WAFv2/Shield RO; plus Health/Trusted Advisor RO and Support case management.
* **Monitoring admin:** Internet Monitor, VPC Network Monitor.

## Guardrails/constraints (enforced by the boundary, not here)

* MFA required (`mfa=true` session tag; optional age ≤ 3600s).
* `PassRole` only to roles tagged `PassRole=true`.
* **IAM limited to Get/List/PassRole/SLR (no broader IAM); org-wide RAM sharing denied.**
* Service Quotas increases limited to approved network services.

## Tagging gates (enforced here via Conditions)

* SNS/SQS creation/ops require `Role=network-architecture` and `TagKeys` ∈ {`Role`,`Owner`,`DevOps`}.

## Change control & validation

* Validate with: `aws accessanalyzer validate-policy`.
* Test with: `aws iam simulate-principal-policy` using real ARNs where tag conditions apply.
