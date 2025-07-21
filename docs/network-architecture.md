# `NetworkArchitectureRole` – IAM Policy Justification

> **Purpose**: This role enables AWS network architects and engineers to prototype, simulate, and validate advanced networking topologies and services in non-production environments. All actions are bound by a permissions boundary and scoped via tagging to prevent misuse.

---

## Infrastructure as Code (IaC)

| **Service / Action** | **Justification** |
| - | - |
| `cloudformation:*` | To define, deploy, update, and test full network stacks declaratively using CloudFormation templates. |
| `codecommit:*` | To host architectural templates, modules, and version-controlled documentation in Git repositories. |
| `codebuild:*` | To validate CloudFormation/Terraform syntax, run integration tests, or generate architecture diagrams as part of CI. |
| `codedeploy:*`, `codepipeline:*` | To simulate full deployment pipelines, allowing blue/green, canary, or multi-region experiments using network IaC workflows. |

---

## Networking Services

| **Service / Action** | **Justification** |
| - | - |
| `ec2:*` (scoped) | To model networking constructs like subnets, ENIs, VPCs, security groups, NATs, and test with EC2 for traffic validation. |
| `elasticloadbalancing:*` | To test NLB/TLS termination, DNS resolution, cross-zone routing, and integration with internal backends. |
| `networkmanager:*` | To design and visualize AWS Cloud WAN, SD-WAN overlays, and core routing domains. |
| `route53resolver:*` | To simulate DNS forwarding rules, conditional resolvers, and hybrid resolution scenarios. |
| `globalaccelerator:*` | To evaluate low-latency failover and global routing scenarios with multi-region endpoints. |
| `vpc-lattice:*` | To test service-to-service communication across VPCs using Lattice constructs. |
| `network-firewall:*` | To test stateful and stateless rules, alerting, and inspection flows across subnet boundaries. |
| `directconnect:*` | To simulate DX attachments and path redundancy in hybrid topologies. |

---

## Discovery, Tagging & Inventory

| **Service / Action** | **Justification** |
| - | - |
| `resource-explorer:*`                    | To find and correlate resources across regions/accounts by tag, type, or name. |
| `resource-groups:*`                      | To group and model components of logical network stacks. |
| `tag:GetResources`<br>`tag:GetTagValues` | To support compliance tagging checks and dynamic environment-scoped experiments. |

---

## Identity & Role Delegation

| **Service / Action** | **Justification** |
| - | - |
| `iam:PassRole` | Required to allow execution roles for CloudFormation and automation to assume minimal privilege. |
| `iam:Get*`, `iam:List*` | To discover available IAM roles and validate policy attachment in templates. Prevents accidental privilege escalations. |

---

## Observability & Drift Detection

| **Service / Action** | **Justification** |
| - | - |
| `logs:*` | To write and read logs generated from architectural testing (e.g., flow logs, error logs). |
| `cloudwatch:*` | To monitor metrics like NAT usage, VPC traffic, network anomalies, etc. |
| `cloudtrail:*` | To trace IAM actions, API calls, template updates, and validate changes made during testing. |
| `xray:*` | To simulate distributed network tracing across services. |
| `config:*`, `config:Put*`, `config:Delete*` | To track resource drift, validate guardrails, and simulate config rules on prototypes. |

---

## EC2 Diagnostics

| **Service / Action** | **Justification** |
| - | - |
| `ssm:*`, `ssmmessages:*`, `ec2messages:*` | To run traceroute, dig, tcpdump, or curl commands on deployed EC2 instances for validation. Enables packet path tracing. |

---

## Optional Artifact Storage

| **Service / Action** | **Justification**                                                                               |
| - | - |
| `s3:PutObject` | To upload rendered diagrams, logs, or tested templates into a designated S3 bucket for reviews. |

---

## Developer Tools

| **Service / Action** | **Justification** |
| - | - |
| `cloudshell:*` | Enables secure CLI usage in AWS browser-based shell for ad-hoc validations and low-friction prototyping. |
| `qbusiness:QueryAssistant` | Use of Amazon Q for architectural guidance, troubleshooting, and documentation discovery. |

---

## EventBridge Prototyping

| **Service / Action** | **Justification** |
| - | - |
| `events:PutRule`<br>`events:PutTargets`<br>`events:DescribeRule` | To simulate event-driven architectures (e.g., detect TGW route changes, auto-label new subnets). Enables dynamic workflows. |

---

## Security Controls (Boundary Enforced)

| **Condition / Control** | **Justification** |
| - | - |
| `Deny if Environment=prod` | Enforces the role is **only used in non-prod environments**. Prevents accidental real-world modifications. |
| `MFA required for sts:AssumeRole` | Ensures secure assumption of the role only by authenticated engineers. |

---
