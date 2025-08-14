# AWS IAM Role: Network Architecture

## What this is

* CloudFormation template that creates **one** IAM role with **SAML federation**.
* Attaches **one** required managed policy and enforces **one** required permissions boundary.
* Effective permissions = **(attached managed policy ∩ permissions boundary) − explicit denies**.

## What it creates

* `AWS::IAM::Role` with:

  * **Trust:** SAML principal in this account only
    `arn:aws:iam::<AccountNumber>:saml-provider/<SamlProviderName>`
    and audience lock: `SAML:aud = https://signin.aws.amazon.com/saml`
  * **Name:** `"<PrefixProfileName>-<EnvironmentName>-<CustomRoleName>-role"`
  * **Path:** `CustomRolePath`
  * **MaxSessionDuration:** 1–6 hours (default **3600s**; cap **21600s**)
  * **ManagedPolicyArns:** `arn:aws:iam::<AccountNumber>:policy<CustomPolicyPath><ManagedPolicyName>`
  * **PermissionsBoundary:** `arn:aws:iam::<AccountNumber>:policy<CustomPolicyPath><PermissionsBoundaryPolicyName>`

## Conventions & constraints

* No extra trusted principals (**SAML only**). No tags. No outbound `sts:AssumeRole` granted here.
* Policy/boundary ARNs are assembled from **AccountNumber + CustomPolicyPath + names**.
* `PathIsRoot` condition resolves `/` vs. nested policy paths.
* Environment is encoded in the **role name**, not in policy/boundary names.

## Parameters (summary)

* **AccountNumber:** Owner account for policies and SAML provider.
* **CustomPolicyPath:** Policy path prefix (`/` to use root).
* **PrefixProfileName / EnvironmentName / CustomRoleName:** tokens for `RoleName`.
* **ManagedPolicyName / PermissionsBoundaryPolicyName:** required **policy names** (not ARNs).
* **SamlProviderName:** SAML provider name in this account.
* **CustomRolePath:** IAM role path.
* **MaxSessionDuration:** STS session TTL (seconds, **900–21600**).

## Outputs

* `RoleName`, `RoleArn`, `ManagedPolicyArn`, `PermissionsBoundaryArn`, `SamlProviderArn`.

## Usage notes

* Deploy with `CAPABILITY_NAMED_IAM`.
* Recommend **change sets**. Inject `AccountNumber` and `SamlProviderName` at deploy time (keep them out of VCS).
* Leave `MaxSessionDuration` omitted in params to use default.

## Validation tips

* Verify referenced **policies/boundary/SAML provider** exist in `AccountNumber`.
* Run **cfn-lint** for template structure; `aws iam get-role` to confirm final `RoleName/ARN`.
