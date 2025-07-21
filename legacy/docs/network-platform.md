# NetworkPlatformRole

## Purpose

The `network-platform-role` is a high-trust, human-assumed IAM role designated for **provisioning and operating shared enterprise network infrastructure** within AWS. It is used exclusively by the network team to deploy and manage core infrastructure components that form the backbone of the enterprise cloud network.

This includes infrastructure deployed in **network-designated accounts** (e.g., `network`), serving as foundational services for other business units and platform teams.

---

## Key Responsibilities

- Provision, operate, and manage shared AWS network infrastructure at enterprise scale
- Deploy and maintain **production-grade services** such as:
  - VPCs, subnets, route tables, NAT gateways
  - Security groups, NACLs, IPAM configurations
  - Transit gateways and attachments
  - Gateway Load Balancers, Network Load Balancers
  - FortiGate firewalls and third-party firewall integrations
  - AWS Network Firewall, Route 53 Resolver rules
- Own and operate **platform infrastructure** (e.g., shared ingress/egress zones, service mesh gateways)

---

## Approved Use Cases

- Deploying core network components in the `network` AWS account
- Building regional hub/spoke architectures
- Maintaining shared network services across accounts and organizations
- Operating ECS/EKS clusters hosting network-focused control plane workloads
- Executing CloudFormation stacks, Lambdas, and pipelines that manage network infrastructure

---

## Justified AWS Services and Actions

| AWS Service | Reason for Access |
| - | - |
| `ec2:*` | Provision VPCs, subnets, route tables, ENIs, gateways, security groups. |
| `elasticloadbalancing:*` | Deploy GWLBs, NLBs, ALBs for internal or edge routing. |
| `network-firewall:*` | Operate AWS Network Firewall-based controls. |
| `vpc-lattice:*` | Manage Lattice service communication boundaries. |
| `route53resolver:*` | Configure DNS forwarding, resolver endpoints. |
| `directconnect:*` | Manage DX gateways, virtual interfaces, failover routes. |
| `cloudformation:*` | Execute stacks that provision and maintain the above components. |
| `lambda:*` | Automate resource orchestration, tagging, and validation workflows. |
| `codepipeline:*` | Run CI/CD for network stack delivery and promotion. |
| `ssm:*`, `ec2messages:*` | Access and manage EC2 systems tied to platform automation or diagnostics. |
| `logs:*`, `cloudwatch:*` | Observe deployed services, route logs, alert on failures. |
| `iam:PassRole` | Pass execution roles to Lambda, ECS, pipelines, and CloudFormation. |
| `qbusiness:QueryAssistant` | Use Amazon Q for deployment validation, runtime diagnostics. |

---

## Security Controls

| Control | Description |
| - | - |
| **Permissions Boundary** | Enforces explicit restrictions against IAM privilege escalation. |
| **MFA Required** | Human-assumed role; requires MFA at time of use. |
| **Tagging and Auditing** | All resources provisioned are expected to conform to network team standards. |
| **No IAM mutation rights** | Cannot create, attach, or alter IAM roles or policies. |

---

## Tags

| Key | Value |
| - | - |
| Purpose | NetworkPlatform. |
| ManagedBy | CloudFormation. |
| Role | network-platform. |
| Team | Network. |

---

## Outputs

| Logical Name | Description |
| - | - |
| `NetworkPlatformRoleArn` | ARN of the IAM role for platform operations. |
| `NetworkPlatformPolicyArn` | ARN of the IAM policy attached to the role. |
| `NetworkPlatformBoundaryArn` | ARN of the IAM permissions boundary used by the role. |
