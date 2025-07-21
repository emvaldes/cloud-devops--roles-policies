# NetworkReadonlyRole

## Purpose

The `network-readonly-role` is a **human-assumed IAM role** used by the network team to perform **non-intrusive diagnostics, inventory collection, and visibility audits** across AWS network environments. It is strictly read-only and cannot modify, deploy, or delete any resources.

This role is used for:
- Live troubleshooting of networking behavior (VPCs, ENIs, route tables, TGWs, firewalls)
- Inventory enumeration for reporting or gap analysis
- Metadata and tagging audits
- Log/event inspection for RCA and validation

---

## Role Characteristics

- **Strictly read-only** — enforces hard denial of all mutation actions
- **Human-assumed only** — protected by MFA and session duration
- **Used in production** — authorized for real-time diagnostics in live environments
- **Supports visibility across the full AWS networking layer**

---

## Justified AWS Services and Actions

| AWS Service | Reason for Access |
| - | - |
| `ec2:Describe*`, `Get*` | Enumerate VPCs, subnets, interfaces, routes, NATs, etc. |
| `networkmanager:*` | View Cloud WAN, site links, TGW routing domains. |
| `directconnect:Describe*` | Diagnose virtual interfaces and DX gateways. |
| `elasticloadbalancing:Describe*` | Observe load balancer configurations, listeners, and health checks. |
| `route53resolver:*` | Audit DNS forwarding rules, resolvers, and endpoints. |
| `vpc-lattice:List*`, `Get*` | Retrieve service-to-service mesh topology. |
| `logs:Get*`, `FilterLogEvents` | View VPC Flow Logs, ENI events, and resource-specific logs. |
| `cloudwatch:Get*`, `List*` | Inspect metrics, alarms, dashboards, and resource telemetry. |
| `cloudtrail:LookupEvents` | Trace IAM usage, role switching, API calls. |
| `config:Get*`, `Describe*` | Audit current resource configuration and drift states. |
| `resource-explorer:*` | Find and relate tagged resources across accounts/regions. |
| `tag:GetResources` | Build tagging compliance reports and audit tag coverage. |
| `iam:Get*`, `List*` | View role trust policies and IAM role usage metadata. |
| `ssm:Get*`, `Describe*` | Correlate EC2 instance metadata and active sessions. |
| `qbusiness:QueryAssistant` | Use Amazon Q for diagnostics, inventory scripting, and root cause guidance. |

---

## Security Controls

| Control | Description |
| - | - |
| **Permissions Boundary** | Denies all resource mutation globally. |
| **MFA Required** | Role assumption requires MFA. |
| **Session Duration** | Limited to 1 hour max per session. |
| **No write permissions** | Zero ability to create, modify, or delete resources. |
| **Scoped via tagging** | All actions are auditable and attributable to the network team. |

---

## Tags

| Key | Value |
| - | - |
| Purpose | NetworkReadonly. |
| ManagedBy | CloudFormation. |
| Role | network-readonly. |
| Team | Network. |

---

## Outputs

| Logical Name | Description |
| - | - |
| `NetworkReadonlyRoleArn` | ARN of the IAM role for diagnostics and inventory visibility. |
| `NetworkReadonlyPolicyArn` | ARN of the IAM policy attached to the readonly role. |
| `NetworkReadonlyBoundaryArn` | ARN of the permissions boundary restricting write access. |
