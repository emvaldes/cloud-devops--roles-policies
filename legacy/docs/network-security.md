# NetworkSecurityRole

## Purpose

The `network-security-role` is a **human-assumed IAM role** designated for performing **read-only security audits** of the AWS network infrastructure. It enables members of the network security team to evaluate the compliance posture, monitor findings from security services, and review IAM configurations — without having the ability to modify, suppress, or interfere with any resources or findings.

This role is used to **audit enterprise network infrastructure** across all AWS accounts controlled by the network team.

---

## Key Capabilities

- Monitor and evaluate **SecurityHub**, **GuardDuty**, **Inspector**, and **AccessAnalyzer** findings
- Audit IAM roles, policies, trust relationships, and pass-role usage
- Review CloudTrail logs and Config state for compliance validation
- Access diagnostic tools such as CloudShell and Amazon Q for assistance and analysis
- Support internal or external audit events, compliance assessments, and forensic reviews

---

## Justified AWS Services and Actions

| AWS Service | Reason for Access |
| - | - |
| `iam:Get*`, `iam:List*` | Review trust policies, managed policies, role usage, and privilege boundaries. |
| `access-analyzer:Get*`, `List*`, `Describe*` | View external access paths, resource-sharing analyzers, and findings. |
| `config:Get*`, `Describe*` | Access compliance history, resource states, and drift detection. |
| `cloudtrail:LookupEvents`<br>`DescribeTrails` | Trace user and role actions across environments. |
| `securityhub:Get*`, `Describe*`, `List*` | Read compliance standards and security findings. |
| `guardduty:Get*`, `Describe*`, `List*` | Monitor threat intelligence and attack surface detection. |
| `inspector2:Get*`, `Describe*`, `List*` | Review vulnerability scan results across workloads. |
| `cloudshell:*` | Perform interactive review and diagnostics using AWS CloudShell. |
| `qbusiness:QueryAssistant` | Get contextual help or explanations from Amazon Q for security services. |

---

## Security Controls

| Control | Description |
| - | - |
| **Permissions Boundary** | Denies all mutation actions, ensuring read-only behavior. |
| **MFA Enforcement** | Required for role assumption. |
| **No Suppression/Response** | Cannot suppress, archive, or delete findings in any security service. |
| **Tagged and Auditable** | All actions traceable to the network security team for governance validation. |

---

## Tags

| Key | Value |
| - | - |
| Purpose | NetworkSecurity. |
| ManagedBy | CloudFormation. |
| Role | network-security. |
| Team | Network. |

---

## Outputs

| Logical Name | Description |
| - | - |
| `NetworkSecurityRoleArn` | ARN of the IAM role for security audits. |
| `NetworkSecurityPolicyArn` | ARN of the managed policy attached to the security role. |
| `NetworkSecurityBoundaryArn` | ARN of the permissions boundary enforcing read-only behavior. |
