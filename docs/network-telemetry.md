# NetworkTelemetryRole

## Purpose

The `NetworkTelemetryRole` enables trusted telemetry services to collect, transform, and forward observability data related to AWS network infrastructure. This includes integration with log and metric ingestion platforms, event pipelines, and external analysis services.

This role is **not human-assumed**. It is used by telemetry agents and services (Lambda, ECS, EC2, Firehose) responsible for routing real-time diagnostics and metrics from VPCs, subnets, gateways, endpoints, and flow logs into observability stacks.

## Key Capabilities

- Stream logs and metrics to Firehose, Kinesis, and S3
- Forward VPC Flow Logs, ENI telemetry, NAT metrics, and TGW events
- Integrate with CloudWatch, X-Ray, and AWS Config for network performance baselining
- Publish alerts and summaries to SNS and SQS for downstream automation
- Securely read all relevant EC2 and VPC metadata for correlation

---

## Justified AWS Services and Actions

| AWS Service | Reason for Access |
| - | - |
| `logs:*`, `cloudwatch:*`, `xray:*` | Ingest and forward logs and metrics from ECS, Lambda, EC2, NAT Gateways, etc. |
| `ec2:Describe*`, `networkmanager:*` | Read metadata about interfaces, subnets, attachments, and topology. |
| `firehose:PutRecord*`<br>`kinesis:PutRecord*` | Send telemetry streams to observability pipelines. |
| `sns:Publish`, `sqs:SendMessage` | Emit alerts and structured summaries into automation queues. |
| `s3:PutObject`, `s3:GetObject` | Store long-term telemetry data, buffer for offline analytics. |
| `config:Get*`, `config:Describe*` | Query resource configuration snapshots and detect drift. |
| `qbusiness:QueryAssistant` | Use Amazon Q to assist in telemetry pipeline validation and optimization. |

---

## Security Controls

| Control | Description |
| - | - |
| **Permissions Boundary** | Prevents the role from modifying IAM or non-telemetry infrastructure. |
| **Service Principal Constraint** | Only trusted services (Lambda, EC2, ECS tasks, Firehose) may assume this role. |
| **Scoped Write Permissions** | Only delivery services (logs, metrics, streams) are allowed — no EC2, IAM, VPC mutation. |
| **Tags** | Used for auditing telemetry function scope and team ownership. |

---

## Tags

| Key | Value |
| - | - |
| Purpose | NetworkTelemetry. |
| ManagedBy | CloudFormation. |
| Role | network-telemetry. |
| Team | Network. |

---

## Outputs

| Logical Name | Description |
| - | - |
| `NetworkTelemetryRoleArn` | ARN of the IAM role for telemetry services. |
| `NetworkTelemetryPolicyArn` | ARN of the IAM policy attached to the telemetry role. |
| `NetworkTelemetryBoundaryArn` | ARN of the permissions boundary applied to the role. |
