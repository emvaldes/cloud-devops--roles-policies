# network-breakglass-role

## Purpose

The `network-breakglass-role` is a **high-privilege, short-lived, emergency-use IAM role** designed exclusively for **production environments**. It enables network engineers to respond to **critical incidents** where all other access paths are insufficient or unavailable.

---

## Characteristics

| Attribute | Value |
| - | - |
| Session Duration | 5 minutes. |
| Assumable By | Only Okta federated users (SAML). |
| Assumable From IAM Role | Never allowed. |
| MFA Requirement | Enforced at assume-role time. |
| Permissions Boundary | Not used – would block emergency ops. |
| Environment | Production only. |
| Monitoring | CloudTrail + EventBridge triggers. |
| Risk Level | High – post-use review required. |

---

## Security Controls

- **Trust policy** enforces assumption only via `Okta` SAML provider
- **MFA required**
- **Cannot be assumed by other IAM roles**
- **No permissions boundary** (not appropriate for this use case)
- **Session TTL set to 300 seconds (5 minutes)**
- **Must be tightly integrated with**:
  - CloudTrail (activity logging)
  - EventBridge rules (assumption alerting)
  - Incident ticketing system (justification + traceability)

---

## Operational Guidance

- This role must only be used when:
  - All automated and scoped roles have failed
  - Incident response requires immediate, unrestricted access
  - Justification and notification protocols are triggered

- After each use:
  - Triggered alarms should be reviewed
  - Post-incident analysis must be performed
  - Access logs should be audited for anomalies or misuse

---

## Outputs

| Logical Name | Description |
| - | - |
| `NetworkBreakglassRoleArn`  | ARN of the breakglass IAM role. |
| `NetworkBreakglassPolicyArn`| ARN of the unrestricted policy for breakglass access. |

---

## Warning

> This role grants full access to all AWS services and resources.
> **Use is strictly limited to emergency production incidents.**
> Every assumption must be logged, alarmed, and followed by a forensic review.
