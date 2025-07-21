Yes. Here’s the **Administrator** version of that doc, same structure/ordering, with only the necessary differences:

# AWS IAM Role: Network Administrator

## What this is

* CloudFormation template that creates **one** IAM role with **SAML federation**.
* Attaches **one** required managed policy and enforces **one** required permissions boundary.
* Effective permissions = **(attached managed policy ∩ permissions boundary) − explicit denies**.
* Optimized for **day-2 network operations** (not general prototyping).

## What it creates

* `AWS::IAM::Role` with:

  * **Trust:** SAML principal in this account only
    `arn:aws:iam::<AccountNumber>:saml-provider/<SamlProviderName>`
    audience lock: `SAML:aud = https://signin.aws.amazon.com/saml`
    **and** allows `sts:TagSession` so the IdP can pass `mfa=true` (required by the guardrails)
  * **Name:** `"<PrefixProfileName>-<EnvironmentName>-<CustomRoleName>-role"`
  * **Path:** `CustomRolePath`
  * **MaxSessionDuration:** 1–6 hours (default **3600s**; cap **21600s**)
  * **ManagedPolicyArns:** `arn:aws:iam::<AccountNumber>:policy<CustomPolicyPath><ManagedPolicyName>` (**administrator policy**)
  * **PermissionsBoundary:** `arn:aws:iam::<AccountNumber>:policy<CustomPolicyPath><PermissionsBoundaryPolicyName>` (**administrator boundary**)

## Conventions & constraints

* **SAML only** trusted principal. No outbound `sts:AssumeRole` granted here.
* Session must carry `aws:PrincipalTag/mfa = true` (enforced by boundary/policy); trust permits `sts:TagSession`.
* Policy/boundary ARNs are assembled from **AccountNumber + CustomPolicyPath + names**.
* `PathIsRoot` condition resolves `/` vs. nested policy paths.
* Environment is encoded in the **role name**, not in policy/boundary names.

## Parameters (summary)

* **AccountNumber:** Owner account for policies and SAML provider.
* **CustomPolicyPath:** Policy path prefix (`/` to use root).
* **PrefixProfileName / EnvironmentName / CustomRoleName:** tokens for `RoleName`.
* **ManagedPolicyName / PermissionsBoundaryPolicyName:** **administrator** policy names (not ARNs).
* **SamlProviderName:** SAML provider name in this account.
* **CustomRolePath:** IAM role path.
* **MaxSessionDuration:** STS session TTL (seconds, **900–21600**).

## Outputs

* `RoleName`, `RoleArn`, `ManagedPolicyArn`, `PermissionsBoundaryArn`, `SamlProviderArn`.

## Usage notes

* Deploy with `CAPABILITY_NAMED_IAM`.
* Recommend **change sets**. Inject `AccountNumber` and `SamlProviderName` at deploy time.
* Use the **administrator** managed policy/boundary pair (read-only for non-network platforms; ops/admin where required for network services).

## Validation tips

* Verify the **administrator** managed policy & boundary exist and match the finalized guardrails.
* Confirm trust policy includes:

  * `sts:AssumeRoleWithSAML` with `SAML:aud = https://signin.aws.amazon.com/saml`
  * `sts:TagSession` permitting `aws:RequestTag/mfa = "true"` and `aws:TagKeys = ["mfa"]`
* After deploy, **assume the role** and check the session has `mfa=true` and actions align with day-2 ops scope.
