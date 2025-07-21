## NetworkEngineeringRole

### Purpose
The `NetworkEngineeringRole` enables trusted engineers to build, maintain, and extend the infrastructure and automation tooling that powers the network team. This includes services like Ansible Automation Platform, Nautobot, ECS/EKS automation layers, pipeline integrations, and diagnostics frameworks.

### Key Responsibilities
- Maintain and evolve automation platforms used by the network team
- Deploy and manage production-grade systems like Nautobot, AAP, or related tools
- Build automation pipelines using CloudFormation, SSM, Secrets Manager, and EventBridge
- Integrate tooling with observability platforms like CloudWatch and Amazon Q

### Justified AWS Services and Actions

| AWS Service | Reason for Access |
| - | - |
| `ec2:*`                | Provision underlying compute for automation systems, testing environments. |
| `ssm:*`, `ssmmessages:*`, `ec2messages:*` | Execute automation scripts and manage fleet diagnostics. |
| `cloudformation:*`     | Deploy reusable network infrastructure templates. |
| `logs:*`, `cloudwatch:*` | Push and monitor automation logs, configure dashboards. |
| `s3:*`                 | Store artifacts, configuration files, and backups. |
| `secretsmanager:Get*`  | Securely inject secrets into automation environments. |
| `iam:PassRole`, `iam:Get*`, `iam:List*` | Use predefined execution roles and inspect IAM hierarchy. |
| `events:*`             | Trigger automation or validation events across workflows. |
| `qbusiness:QueryAssistant` | Use Amazon Q for diagnostics, templates help, and command generation. |

### Security Controls
- **Permissions Boundary**: Explicitly denies IAM escalation paths
- **MFA Enforcement**: Required for all role assumptions
- **Scoped Tags**: Tags are used for auditing and boundary evaluation
serve internal and enterprise-wide network functions. Environment tagging is used for visibility, not restriction.
- **Internal production scope only**: This role is authorized to deploy and manage production-grade services owned and operated by the network team (e.g., Nautobot, AAP, Jenkins). It is not used for external project delivery. Environment tagging is for visibility, not restriction.

### Tags Applied
- `Purpose=NetworkEngineering`
- `ManagedBy=CloudFormation`
- `Role=network-engineering`
- `Team=Network`
