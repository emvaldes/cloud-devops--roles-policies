# NetworkArchitectureRole — Reference Implementation Guide (Current)

## 1) What this template does

Creates **one IAM role** with:

* **Trust**: SAML federation only (`AssumeRoleWithSAML`) to a provider in the same account, audience locked to `https://signin.aws.amazon.com/saml`.
* **Name**: `${PrefixProfileName}-${EnvironmentName}-${CustomRoleName}-role`.
* **Path**: `CustomRolePath` (optional grouping).
* **Session TTL**: `MaxSessionDuration` (default 3600s, min 900s, max 21600s).
* **Policies**:

  * **Attaches** an existing **managed policy** by **name + path**.
  * **Enforces** an existing **permissions boundary** by **name + path**.

**Non-goals (by design):**

* No extra trusted principals (no role-to-role trust).
* No outbound `sts:AssumeRole` granted by this role’s policy (add later if required).
* No tag-conditions or tag gates in this role.

---

## 2) Naming & identity model

**RoleName formula**

```
${PrefixProfileName}-${EnvironmentName}-${CustomRoleName}-role
```

* Keep tokens short and stable. Examples:

  * `aws-dev-network-architecture-role`
  * `net-uat-platform-role`

**Policy ARNs**

* Built from **AccountNumber + CustomPolicyPath + Name**

  * `arn:aws:iam::${AccountNumber}:policy${PathPart}${ManagedPolicyName}`
  * `arn:aws:iam::${AccountNumber}:policy${PathPart}${PermissionsBoundaryPolicyName}`
* `PathPart` resolution:

  * If `CustomPolicyPath == "/"` → `PathPart = "/"`.
  * Else → `PathPart = CustomPolicyPath` (must end with `/`).

**SAML provider ARN**

```
arn:aws:iam::${AccountNumber}:saml-provider/${SamlProviderName}
```

---

## 3) Parameters (inputs)

| Param                           | Type   | Constraints / Notes                                                  | Example                                     |
| ------------------------------- | ------ | -------------------------------------------------------------------- | ------------------------------------------- |
| `AccountNumber`                 | String | 12-digit AWS account ID that **owns** the SAML provider and policies | `123456789012`                              |
| `CustomPolicyPath`              | String | regex `^/([A-Za-z0-9+=,.@_-]+/)*$` or `/`                            | `/managed/network/`                         |
| `PrefixProfileName`             | String | first token in RoleName                                              | `aws`                                       |
| `EnvironmentName`               | String | second token in RoleName                                             | `prod`                                      |
| `CustomRoleName`                | String | third token in RoleName                                              | `network-architecture`                      |
| `ManagedPolicyName`             | String | **name** (not ARN)                                                   | `network-architecture`                      |
| `PermissionsBoundaryPolicyName` | String | **name** (not ARN)                                                   | `network-architecture-permissions-boundary` |
| `SamlProviderName`              | String | SAML provider name in the account                                    | `iam.company.com`                           |
| `CustomRolePath`                | String | IAM Role path; optional grouping                                     | `/role/network/`                            |
| `MaxSessionDuration`            | Number | default `3600`, min `900`, max `21600`                               | `21600` (6h)                                |

**Why names, not ARNs?**
Keeping names lets you switch `CustomPolicyPath` (or root `/`) without changing every parameter. ARNs are resolved centrally from `AccountNumber + path + name`.

---

## 4) Trust model

**Allowed principal**

* `Federated: arn:aws:iam::${AccountNumber}:saml-provider/${SamlProviderName}`

**Action**

* `sts:AssumeRoleWithSAML`

**Condition**

* `StringEquals: { SAML:aud: https://signin.aws.amazon.com/saml }`
  (Prevents non-AWS SSO flows from reusing the assertion.)

**No additional principals**

* There is **no** `TrustedRoleArn`. CI/CD or automation hops aren’t permitted unless added later explicitly.

---

## 5) Permissions model (attach vs boundary)

* **Attached managed policy**: grants capabilities.
* **Permissions boundary**: ceiling. Effective permissions are
  **(attached policy ∩ boundary) − explicit denies (in boundary)**.
* The role itself **does not** include inline policies.

**Outbound STS**

* Not granted by default. If you later need `sts:AssumeRole`, add it to the managed policy **and** ensure the boundary doesn’t deny it.

---

## 6) Conditions in template

```yaml
Conditions:
  PathIsRoot: !Equals [ !Ref CustomPolicyPath, "/" ]
```

* Used to compute `PathPart` for policy/boundary ARNs:

  * `PathPart: "/"` when root.
  * `PathPart: !Ref CustomPolicyPath` otherwise.
* This avoids malformed ARNs and duplicated slashes.

---

## 7) Outputs (what you can wire to other stacks/tools)

* `RoleName` — CFN `Ref` ⇒ the IAM role **name**.
* `RoleArn` — full role ARN.
* `ManagedPolicyArn` — resolved from inputs (`AccountNumber + path + name`).
* `PermissionsBoundaryArn` — same resolution logic as above.
* `SamlProviderArn` — federated principal ARN used in trust.

---

## 8) Deploy model (change sets + JSON params)

**`network-architecture-role.params.json`**

```json
[
  { "ParameterKey": "AccountNumber",                 "ParameterValue": "" },
  { "ParameterKey": "SamlProviderName",              "ParameterValue": "" },
  { "ParameterKey": "CustomPolicyPath",              "ParameterValue": "/managed/network/" },
  { "ParameterKey": "PrefixProfileName",             "ParameterValue": "aws" },
  { "ParameterKey": "EnvironmentName",               "ParameterValue": "prod" },
  { "ParameterKey": "CustomRoleName",                "ParameterValue": "network-architecture" },
  { "ParameterKey": "ManagedPolicyName",             "ParameterValue": "network-architecture" },
  { "ParameterKey": "PermissionsBoundaryPolicyName", "ParameterValue": "network-architecture-permissions-boundary" },
  { "ParameterKey": "CustomRolePath",                "ParameterValue": "/role/network/" }
]
```

* Leave `AccountNumber` and `SamlProviderName` empty in Git; fill at runtime (two string replacements) before creating the change set.
* Omit `MaxSessionDuration` to use the default.

> You said we’ll cover CLI details in a separate chat—this is just the **file shape** you asked for.

---

## 9) Validation checklist (fast, practical)

* **Lint**: `aws cloudformation validate-template --template-body file://network-architecture-role.yaml`
* **Existence** (before deploy):

  * Managed policy **name** exists at `CustomPolicyPath` in `AccountNumber`.
  * Permissions boundary **name** exists at `CustomPolicyPath` in `AccountNumber`.
  * SAML provider exists:
    `arn:aws:iam::${AccountNumber}:saml-provider/${SamlProviderName}`
* **After deploy**:

  * `aws iam get-role --role-name "<resolved name>"` → check `Arn`, `MaxSessionDuration`, `AssumeRolePolicyDocument`.
  * Verify attached managed policy and permissions boundary on the role.

---

## 10) Common pitfalls & how this template avoids them

* **Broken ARNs due to path slashes** → `PathIsRoot` resolves `/` vs nested paths safely.
* **Mixing names/ARNs** → parameters are **names**; ARNs built consistently in the template.
* **Trust drift** → single SAML principal + audience lock; no optional trusted role knob.
* **Overlong sessions** → upper cap 6h (`21600`).
* **Hidden env suffixes** → environment lives **in the role name**, not in policy names.

---

## 11) Reuse guidance (turn this into other roles)

* Keep the **structure identical**; only change:

  * `CustomRoleName` (and possibly `PrefixProfileName` convention).
  * The **managed policy** (name) attached.
  * The **permissions boundary** (name) enforced.
* Preserve `AssumeRoleWithSAML` + audience lock unless your SSO model demands otherwise.
* Keep `CustomPolicyPath` consistent across the org; use `/` if paths are not used.

---

## 12) Future extensions (when there’s a concrete need)

* **Add a controlled `TrustedRoleArn`** + explicit `sts:AssumeRole` in policy (and boundary allow) for CI/CD hops.
* **Tighten TTL** by reducing default or max `MaxSessionDuration`.
* **Export outputs** (CloudFormation `Export`) if nested or cross-stack references are required.

---
