## Table of Contents

1. [What is IAM?](#1-what-is-iam)
2. [IAM Users](#2-iam-users)
3. [IAM Groups](#3-iam-groups)
4. [IAM Roles](#4-iam-roles)
5. [AWS STS and Temporary Credentials](#5-aws-sts-and-temporary-credentials)
6. [IAM Policy Types](#6-iam-policy-types)
7. [Anatomy of a Policy Document](#7-anatomy-of-a-policy-document)
8. [Policy Evaluation Logic](#8-policy-evaluation-logic)
9. [Condition Keys](#9-condition-keys)
10. [Tag-Based Access Control (ABAC)](#10-tag-based-access-control-abac)
11. [Permissions Boundaries](#11-permissions-boundaries)
12. [Service Control Policies (SCPs)](#12-service-control-policies-scps)
13. [Cross-Account Access](#13-cross-account-access)
14. [Identity Federation](#14-identity-federation)
15. [AWS IAM Identity Center (SSO)](#15-aws-iam-identity-center-sso)
16. [Service-Linked Roles](#16-service-linked-roles)
17. [Multi-Factor Authentication (MFA)](#17-multi-factor-authentication-mfa)
18. [Root User Account](#18-root-user-account)
19. [Password Policy & Credential Management](#19-password-policy--credential-management)
20. [IAM Access Analyzer](#20-iam-access-analyzer)
21. [IAM Policy Simulator](#21-iam-policy-simulator)
22. [Credential Report & Last Accessed Data](#22-credential-report--last-accessed-data)
23. [IAM Best Practices](#23-iam-best-practices)
24. [Common Interview Questions](#24-common-interview-questions)

---

## 1. What is IAM?

**AWS Identity and Access Management (IAM)** is a global AWS service that lets you securely control **who** (authentication) can do **what** (authorization) on **which** AWS resources.

- **Global service** — IAM is not Region-scoped; users, groups, roles, and policies exist account-wide, across all Regions.
- **Free to use** — no additional charge for IAM itself.
- Core job: issue **identities** (users, roles) and attach **policies** (JSON documents) that grant or deny specific actions on specific resources, optionally under specific conditions.

---

## 2. IAM Users

An **IAM User** is a persistent identity representing a single person or application that needs long-term access to your AWS account.

### Key Characteristics
- Has a unique name within the account and can have:
  - **Console password** (for sign-in to the AWS Management Console).
  - **Access keys** (access key ID + secret access key) for programmatic access via CLI/SDK.
- Credentials are **long-term** by default — they don't expire automatically (unlike role-based STS credentials), which is why users are increasingly discouraged in favor of roles/federation.
- A user can belong to **up to 10 groups**, and can have both **managed** and **inline** policies attached directly.

### When to Use IAM Users
- Legacy or small-scale setups without a federation/SSO solution.
- Break-glass emergency access accounts (used rarely, tightly monitored).
- Service accounts for tools that genuinely cannot assume a role (rare in modern architectures — most AWS services and compute environments support roles instead).

> **Interview tip:** Modern AWS guidance favors **IAM Identity Center / federated roles over long-lived IAM users with access keys**, since long-term credentials are a major source of security incidents (leaked keys in code repos, etc.).

---

## 3. IAM Groups

An **IAM Group** is a collection of IAM users, used to apply the same set of policies to many users at once.

### Key Characteristics
- A group is **not** a true identity — it cannot be a principal in a trust policy, cannot be assumed, and cannot be referenced as the subject of a resource-based policy.
- Policies attached to a group apply to **every user in that group**.
- Groups **cannot be nested** (a group cannot contain another group).
- A user can be a member of **multiple groups**, and receives the union of all their permissions.

### Use Case
- Organize users by job function (e.g., `Developers`, `Admins`, `Billing`) and attach the appropriate managed policy to each group once, instead of to every individual user.

---

## 4. IAM Roles

An **IAM Role** is an identity you create in AWS with a defined set of permissions to perform actions on AWS resources. Unlike an IAM user, a role does **not** have long-term credentials such as passwords or access keys. Instead, when an entity assumes a role, **AWS Security Token Service (STS)** issues **temporary security credentials** for that session.

### Core Components

- **Trust policy**
  Defines which principals (users, services, or accounts) are allowed to assume the role. This is a resource-based policy attached to the role itself, separate from its permissions policy.

- **Permissions policy**
  Specifies the actions and resources the role can perform or access once assumed.

- **Session duration**
  Determines how long the temporary credentials remain valid — default maximum is **12 hours**, but can be configured as low as 1 hour or as high as 12 hours (36 hours for some federation scenarios via `AssumeRoleWithSAML`/custom broker setups).

### Common Use Cases
- **EC2 instance profiles** granting instances access to S3 buckets without embedding credentials on disk.
- **Lambda execution roles** enabling functions to read/write DynamoDB or publish to SNS.
- **Cross-account access** for auditing or data sharing between separate AWS accounts.
- **Federation** for corporate users via SAML or third-party OIDC providers.
- **Service-linked roles** that AWS services use to perform actions on your behalf (see [Section 16](#16-service-linked-roles)).

### IAM User vs IAM Role

| Aspect | IAM User | IAM Role |
|---|---|---|
| Credentials | Long-term (password, access keys) | Temporary (issued by STS, auto-expiring) |
| Identity | Fixed, tied to one person/app | Assumable by multiple trusted principals |
| Best for | Rare — legacy/break-glass cases | Default choice for apps, services, cross-account, federation |
| Rotation | Must be manually rotated | Automatically rotates every session |

---

## 5. AWS STS and Temporary Credentials

**AWS Security Token Service (STS)** issues short-lived, temporary security credentials (access key, secret key, and session token) for IAM roles and federated users.

### Key STS API Calls

| API Call | Purpose |
|---|---|
| `AssumeRole` | Assume a role within your own account or a trusted external account. |
| `AssumeRoleWithSAML` | Assume a role using a SAML 2.0 assertion from a corporate identity provider. |
| `AssumeRoleWithWebIdentity` | Assume a role using a token from a web identity provider (Google, Facebook, Amazon Cognito, or any OIDC-compatible provider). |
| `GetSessionToken` | Get temporary credentials for an IAM user (e.g., to enable MFA-protected API calls). |
| `GetFederationToken` | Get temporary credentials for a federated user, scoped by a policy you pass in. |

### Key Points
- Temporary credentials **automatically expire**, removing the need for manual rotation — a major security advantage over long-term access keys.
- The session's effective permissions are the **intersection** of the role's permissions policy and any **session policy** passed at assume-time (see [Section 6](#6-iam-policy-types)).
- `AssumeRole` calls can include an **external ID** condition to prevent the "confused deputy" problem in third-party cross-account access scenarios (see [Section 13](#13-cross-account-access)).

---

## 6. IAM Policy Types

### Core IAM Policy Types

| Policy Type | Description | Use Case Example |
|---|---|---|
| **Identity-based** | Attached to IAM users, groups, or roles. Grants permissions to those identities. | Allow a user to access S3 buckets. |
| **Resource-based** | Attached directly to AWS resources (e.g., S3 buckets, Lambda functions, KMS keys). | Allow cross-account access to an S3 bucket. |
| **Managed Policies** | Standalone policies managed by AWS or the user. Can be reused across identities. | AWS-managed: `AmazonEC2ReadOnlyAccess`. Customer-managed: custom EC2 access policy. |
| **Inline Policies** | Embedded directly into a single IAM identity. Cannot be reused. | One-off permissions for a specific user. |

### Advanced Policy Types

| Policy Type | Description | Use Case Example |
|---|---|---|
| **Permissions Boundaries** | Defines the maximum permissions an identity-based policy can grant. | Limit a developer role to read-only access. |
| **Service Control Policies (SCPs)** | Used in AWS Organizations to define max permissions for accounts or OUs. | Prevent accounts from deleting CloudTrail logs. |
| **Session Policies** | Passed during role assumption to limit permissions for the session duration. | Temporary access with restricted actions. |
| **Access Control Lists (ACLs)** | Legacy mechanism for controlling access to resources like S3 and Amazon VPC. | Grant public read access to an S3 object. |

### AWS Managed vs Customer Managed vs Inline

| Aspect | AWS Managed | Customer Managed | Inline |
|---|---|---|---|
| Created/maintained by | AWS | You | You |
| Reusable across identities | ✅ Yes | ✅ Yes | ❌ No (1:1 with identity) |
| Versioning | AWS updates automatically | You manage versions (up to 5 stored) | No versioning |
| Best for | Common, well-known permission sets | Org-specific reusable permission sets | Rare one-off, tightly scoped grants |

---

## 7. Anatomy of a Policy Document

Every IAM policy is a JSON document with a consistent structure:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnly",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": { "aws:RequestedRegion": "us-east-1" }
      }
    }
  ]
}
```

| Element | Purpose |
|---|---|
| `Version` | Policy language version — always `"2012-10-17"` for current policies. |
| `Sid` | Optional statement ID, for readability/reference. |
| `Effect` | `Allow` or `Deny`. |
| `Principal` | Required only on resource-based policies — who the statement applies to. |
| `Action` | The API action(s) the statement covers (supports wildcards, e.g. `s3:Get*`). |
| `Resource` | The ARN(s) the statement applies to (supports wildcards). |
| `Condition` | Optional constraints narrowing when the statement applies (see [Section 9](#9-condition-keys)). |

---

## 8. Policy Evaluation Logic

When a principal makes a request, AWS evaluates **every applicable policy** — identity-based, resource-based, permissions boundaries, SCPs, and session policies — to reach a final decision.

### The Evaluation Flow
1. **Default is implicit Deny** — if nothing explicitly allows the request, it's denied.
2. Check for an **explicit Deny** anywhere in any applicable policy (identity-based, resource-based, SCP, permissions boundary, session policy) — if found, the request is **immediately denied**, no matter what else allows it.
3. If no explicit Deny, check for an **explicit Allow** in at least one applicable policy — if found (and not blocked by a boundary/SCP), the request is allowed.
4. If neither an explicit Allow nor Deny is found, the implicit Deny from step 1 applies.

### Key Rule of Thumb
> **Explicit Deny always wins.** Among Allows, the request is granted only if it falls within the **intersection** of everything that must agree: the identity-based policy, any resource-based policy, the permissions boundary (if set), the SCP (if in an Organization), and the session policy (if assuming a role) — all must allow it; none may deny it.

### Same-Account vs Cross-Account Evaluation

| Scenario | Evaluation |
|---|---|
| Same account | Either the identity-based policy **or** the resource-based policy granting Allow is sufficient (they're OR'd for the Allow decision), subject to any boundary/SCP. |
| Cross-account | **Both** the identity-based policy (in the requester's account) **and** the resource-based policy (in the resource owner's account) must explicitly allow the action (they're AND'd). |

---

## 9. Condition Keys

**Condition keys** let a policy statement apply only when specific contextual conditions are true, adding fine-grained control beyond simple action/resource matching.

### Common Global Condition Keys

| Key | Example Use |
|---|---|
| `aws:SourceIp` | Restrict access to a specific IP range. |
| `aws:SecureTransport` | Require HTTPS (`"Bool": {"aws:SecureTransport": "true"}`). |
| `aws:MultiFactorAuthPresent` | Require the caller to have authenticated with MFA. |
| `aws:RequestedRegion` | Restrict which Region the action can target. |
| `aws:PrincipalOrgID` | Restrict access to principals within your AWS Organization. |
| `aws:SourceVpce` | Restrict access to requests coming through a specific VPC endpoint. |
| `aws:ResourceTag/key` | Restrict based on a tag on the resource being accessed (see ABAC, [Section 10](#10-tag-based-access-control-abac)). |
| `sts:ExternalId` | Used in cross-account role trust policies to prevent the confused deputy problem. |

### Condition Operators
- `StringEquals`, `StringLike` — exact or wildcard string match.
- `NumericLessThan`, `NumericGreaterThan` — numeric comparisons.
- `DateGreaterThan`, `DateLessThan` — time-based access windows.
- `Bool` — true/false conditions (e.g., MFA present, secure transport).
- `IpAddress`, `NotIpAddress` — CIDR-based matching.

---

## 10. Tag-Based Access Control (ABAC)

**Attribute-Based Access Control (ABAC)** grants permissions based on **tags** attached to IAM principals and AWS resources, instead of writing a separate policy per resource (Role-Based Access Control, or RBAC).

### How It Works
- Tag IAM users/roles with attributes, e.g., `Department=Engineering`, `Project=Phoenix`.
- Tag resources (EC2 instances, S3 buckets, etc.) the same way.
- Write a single policy that grants access only when the principal's tag matches the resource's tag:

```json
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*",
  "Condition": {
    "StringEquals": { "aws:ResourceTag/Project": "${aws:PrincipalTag/Project}" }
  }
}
```

### ABAC vs RBAC

| Aspect | RBAC (traditional, role-per-team) | ABAC (tag-based) |
|---|---|---|
| Policy count | Grows with number of teams/resources | A handful of reusable policies |
| Scaling | Requires new policies as org grows | Scales via tagging alone, no new policies needed |
| Onboarding new resources/teams | Requires policy updates | Just apply the right tags |
| Best for | Small, stable org structures | Large, fast-growing orgs with consistent tagging discipline |

---

## 11. Permissions Boundaries

A **Permissions Boundary** is an advanced feature that sets the **maximum permissions** an identity-based policy can grant to a user or role — it doesn't grant permissions itself, it only **caps** what other attached policies can grant.

### Key Points
- The **effective permissions** are the intersection of the identity-based policy and the permissions boundary — even if the identity-based policy allows an action, the boundary blocks it unless the boundary also allows it.
- Commonly used to let a **developer or automation pipeline create IAM roles/users**, while guaranteeing those newly created identities can never exceed a defined boundary (preventing privilege escalation).
- Set on a **per-user or per-role basis** (not account-wide — that's what SCPs are for).

### Example Use Case
A platform team lets application developers create their own IAM roles for their Lambda functions, but attaches a permissions boundary capping every role they create to read-only access on a specific set of services — developers get self-service IAM, without risking privilege escalation to admin.

---

## 12. Service Control Policies (SCPs)

**Service Control Policies** are used in **AWS Organizations** to set the maximum available permissions for **accounts or Organizational Units (OUs)** — they are account/org-wide guardrails, not grants.

### Key Points
- SCPs **never grant permissions** — they only define the outer boundary of what's *possible*, even for the account's root user. An identity still needs an actual identity-based or resource-based policy granting the action.
- Applied at the **AWS Organizations** level — attached to the Organization root, an OU, or an individual account; they cascade down to every account in that OU.
- Does **not** apply to the **management (payer) account** itself by default.
- Common pattern: **Deny lists** — explicitly deny dangerous actions account-wide (e.g., deny `cloudtrail:StopLogging`, deny leaving the Organization, deny disabling GuardDuty) rather than trying to allow-list everything.

### SCP vs Permissions Boundary

| Aspect | SCP | Permissions Boundary |
|---|---|---|
| Scope | AWS account or OU (org-wide) | Individual IAM user or role |
| Requires AWS Organizations | ✅ Yes | ❌ No |
| Affects root user | ✅ Yes | ❌ No (boundaries apply to IAM identities only) |
| Grants permissions | ❌ Never — caps only | ❌ Never — caps only |
| Typical use | Org-wide guardrails (e.g., block region use, block leaving org) | Per-role cap to prevent privilege escalation |

---

## 13. Cross-Account Access

Cross-account access lets a principal in **Account A** access resources in **Account B**, typically via an IAM role in Account B that Account A is trusted to assume.

### How It Works
1. In **Account B** (resource owner), create an IAM role with a **trust policy** naming Account A (or a specific principal in it) as a trusted principal.
2. Attach a **permissions policy** to that role defining what it can do in Account B.
3. A user/role in **Account A** calls `sts:AssumeRole` on the role's ARN in Account B.
4. STS returns temporary credentials scoped to that role's permissions, usable for the session duration.

### Preventing the "Confused Deputy" Problem
- When a **third party** (e.g., a SaaS vendor) is granted a role to access your account, always include a unique, secret **`sts:ExternalId`** condition in the trust policy.
- Without it, if the third party's own AWS account is compromised or reused across customers, an attacker could trick the third party into assuming your role on their behalf — the External ID prevents this by requiring a shared secret only you and the vendor know.

### Alternatives to Cross-Account Roles
- **Resource-based policies** (e.g., an S3 bucket policy or SQS queue policy) granting another account direct access, without that account needing to assume a role at all — simpler for read/write access to a single resource.
- **AWS RAM (Resource Access Manager)** for sharing entire resources (subnets, Transit Gateways, etc.) across accounts.

---

## 14. Identity Federation

**Federation** lets users authenticate with an **external identity provider (IdP)** and receive temporary AWS credentials, without needing an IAM user in AWS at all.

### Types of Federation

| Type | Protocol | Typical Use Case |
|---|---|---|
| **SAML 2.0 Federation** | SAML 2.0 assertions | Corporate workforce using Active Directory / Okta / Azure AD to access the AWS Console or API. |
| **Web Identity Federation (OIDC)** | OpenID Connect | Mobile/web apps letting users sign in with Google, Facebook, Amazon, or Apple. |
| **Amazon Cognito Identity Pools** | Wraps OIDC/SAML | Mobile/web apps needing direct, scalable temporary AWS credentials for end users, often paired with Cognito User Pools for sign-up/sign-in. |
| **AWS IAM Identity Center (SSO)** | SAML/OIDC, AWS-managed | Centralized workforce access across many AWS accounts (see [Section 15](#15-aws-iam-identity-center-sso)). |

### How It Works (SAML Example)
1. User authenticates with the corporate IdP (e.g., Okta).
2. The IdP issues a **SAML assertion** confirming the user's identity and group memberships.
3. The user (or a client app) calls `sts:AssumeRoleWithSAML`, passing the assertion.
4. STS validates the assertion against an **IAM SAML identity provider** resource, and returns temporary credentials scoped to the mapped IAM role.

### Why Federation Over IAM Users
- No long-term AWS credentials to manage, rotate, or leak.
- Centralizes identity lifecycle management (onboarding/offboarding) in the corporate IdP instead of duplicating it in IAM.
- Scales naturally to thousands of users without creating thousands of IAM users.

---

## 15. AWS IAM Identity Center (SSO)

**AWS IAM Identity Center** (formerly AWS Single Sign-On) is AWS's recommended way to manage workforce access **centrally across multiple AWS accounts** in an Organization.

### Key Features
- Provides a single sign-on **portal** where users log in once and see every AWS account/role they're permitted to access.
- Can use its own built-in identity store, or connect to an external IdP (Azure AD, Okta, Google Workspace, on-prem AD via AD Connector) via SAML/SCIM.
- Uses **Permission Sets** — reusable templates of IAM policies — that get provisioned as IAM roles automatically in each target account.
- Also supports SSO access to other AWS resources and some third-party, SAML-enabled applications.

### Identity Center vs Manually Managing Cross-Account Roles

| Aspect | IAM Identity Center | Manual per-account IAM roles/users |
|---|---|---|
| Central user management | ✅ Single place to manage users/groups | ❌ Must manage per account (or via federation setup yourself) |
| Multi-account at scale | ✅ Built for AWS Organizations | ❌ Becomes unmanageable beyond a few accounts |
| Setup complexity | Low (AWS managed service) | Higher (custom SAML/OIDC trust setup) |
| Recommended for | Most organizations' workforce access today | Narrow, single-purpose cross-account trust relationships |

---

## 16. Service-Linked Roles

A **Service-Linked Role** is a special type of IAM role that is **predefined and linked directly to a specific AWS service**, used by that service to perform actions on your behalf.

### Key Points
- Created (and often deleted) automatically by the service itself when needed, or manually from the IAM console in some cases.
- Has a **fixed permissions policy defined by AWS** — you cannot modify its permissions, since the service depends on exactly those permissions to function correctly.
- Appears in IAM with a recognizable naming pattern, e.g. `AWSServiceRoleForAutoScaling`, `AWSServiceRoleForRDS`.
- Cannot usually be deleted while the related resources that depend on it still exist (AWS blocks the deletion to avoid breaking the service).

### Example
When you enable **AWS Auto Scaling**, AWS automatically creates the `AWSServiceRoleForAutoScaling` service-linked role, which Auto Scaling uses to launch/terminate EC2 instances and attach them to load balancers on your behalf — without you needing to craft a custom IAM role for it.

---

## 17. Multi-Factor Authentication (MFA)

**MFA** requires a second authentication factor (beyond a password) to sign in or perform sensitive actions, significantly reducing the risk from compromised passwords.

### Supported MFA Types
- **Virtual MFA device** — authenticator app (e.g., Google Authenticator, Authy) generating time-based one-time passwords (TOTP).
- **Hardware MFA device** — physical token (e.g., YubiKey, Gemalto).
- **FIDO security key** — supports WebAuthn/U2F hardware keys.
- **Built-in authenticators** — fingerprint/face-based, on supported devices.

### Key Use Cases
- Required for the **root user** at minimum (AWS strongly recommends this as a day-one security step).
- Can be **required via policy condition** (`aws:MultiFactorAuthPresent`) for specific sensitive actions (e.g., deleting a bucket, modifying IAM policies) even for otherwise-authorized users.
- `sts:GetSessionToken` can be called with an MFA code to get MFA-authenticated temporary credentials for programmatic access.

> **Interview tip:** Enforcing MFA via an IAM condition is a very common "how would you secure this" interview answer — e.g., "deny all actions unless `aws:MultiFactorAuthPresent` is true."

---

## 18. Root User Account

The **root user** is the identity created when you first set up an AWS account, with **unrestricted access to everything** in the account — it cannot be restricted by any IAM policy, permissions boundary, or (by default) SCP.

### Best Practices for the Root User
- **Enable MFA on the root user immediately** — ideally hardware MFA for the highest assurance.
- **Do not create access keys** for the root user; delete them if they exist.
- **Do not use the root user for day-to-day work** — create an IAM admin user/role instead and lock the root credentials away.
- Set up **account recovery details** (phone, email) correctly in case you need to regain access.
- In an AWS Organizations setup, use **SCPs** to further restrict what member-account root users can do (though the management account's own root user cannot be restricted by its own org's SCPs).

---

## 19. Password Policy & Credential Management

AWS lets you define an account-wide **IAM password policy** enforcing rules for IAM user console passwords.

### Configurable Settings
- Minimum password length and complexity requirements (uppercase, lowercase, numbers, symbols).
- Password expiration period, and whether users can reuse recent passwords.
- Whether users can change their own password.
- Whether to require an administrator to reset an expired password.

### Access Key Best Practices
- Rotate access keys regularly if long-term credentials must be used at all.
- Never embed access keys in source code — use roles instead wherever possible.
- Use **AWS Secrets Manager** or environment-based role credentials instead of hardcoded keys for applications.

---

## 20. IAM Access Analyzer

**IAM Access Analyzer** continuously analyzes resource policies (S3 buckets, IAM roles, KMS keys, Lambda functions, SQS queues, and more) to identify resources that are shared with **external entities** — helping you find unintended public or cross-account access.

### Key Features
- Uses **automated reasoning** (formal logic/mathematical analysis) to determine all possible access paths a resource-based policy grants — not just pattern matching.
- Generates **findings** whenever a resource is accessible from outside a defined "zone of trust" (typically your AWS Organization or account).
- **Policy generation**: can analyze CloudTrail logs of actual API activity to generate a **least-privilege IAM policy** based on what a role/user actually used — great for tightening overly broad policies.
- **Custom policy checks**: validates policies against security best practices before deployment (e.g., checking for public access or unintended access grants), useful in CI/CD pipelines.

### Use Cases
- Regularly auditing for accidental public S3 buckets, publicly shared KMS keys, or overly broad IAM roles.
- Right-sizing IAM policies by generating a policy from actual usage instead of guessing permissions needed.
- Embedding policy validation checks into infrastructure-as-code pipelines before changes are deployed.

---

## 21. IAM Policy Simulator

The **IAM Policy Simulator** is a tool (console and API) that lets you **test and troubleshoot** identity-based and resource-based policies **without actually making the API call** — useful for verifying whether a user/role would be allowed or denied a specific action before granting real access.

### Key Points
- Lets you select an existing user, group, or role, and simulate specific actions against specific resources.
- Shows whether the request would be **Allowed** or **Denied**, and — critically — **which specific statement** in which policy caused that decision.
- Useful for debugging "why is this user still able to do X" or "why can't this role do Y" without trial-and-error in production.
- Does not account for all runtime context (e.g., some condition keys that are only present on a genuine request) — so it's a strong first-pass debugging tool, not a 100% perfect substitute for a live test.

---

## 22. Credential Report & Last Accessed Data

Two built-in IAM auditing features for identifying stale or risky credentials:

### Credential Report
- A downloadable CSV, generated on demand (updated at most once every 4 hours), listing **every IAM user** in the account and the status of their passwords and access keys: whether MFA is enabled, password/key age, last used date, and more.
- Used for account-wide security audits — e.g., finding users who still have active access keys they haven't used in 90+ days.

### Service Last Accessed Data
- Shown on a user/role/group's **Access Advisor** tab — lists which AWS **services** that identity has actually used, and when it last used each one.
- The primary tool for identifying permissions that are granted but never used, feeding into least-privilege clean-up (and pairs well with Access Analyzer's policy generation).

---

## 23. IAM Best Practices

A consolidated checklist, commonly asked as an open-ended interview question ("how would you secure IAM in a new AWS account?"):

1. **Lock down the root user**: enable MFA, remove access keys, don't use it day-to-day ([Section 18](#18-root-user-account)).
2. **Enforce least privilege**: start with minimal permissions and grant more as needed, rather than starting broad and trying to restrict later.
3. **Prefer roles over long-term IAM users**, especially for applications/services ([Section 4](#4-iam-roles)).
4. **Use federation / IAM Identity Center** for human users instead of creating individual IAM users per person ([Sections 14–15](#14-identity-federation)).
5. **Require MFA**, at least for the root user and privileged roles/actions ([Section 17](#17-multi-factor-authentication-mfa)).
6. **Use groups** to assign permissions to users, not per-user inline policies ([Section 3](#3-iam-groups)).
7. **Use permissions boundaries and SCPs** to cap what self-service/automated role creation can grant ([Sections 11–12](#11-permissions-boundaries)).
8. **Regularly audit** with the Credential Report, Access Analyzer, and Access Advisor to find unused permissions and stale credentials ([Sections 20, 22](#20-iam-access-analyzer)).
9. **Rotate and never hardcode** credentials — prefer roles; if keys are unavoidable, use Secrets Manager and rotate regularly ([Section 19](#19-password-policy--credential-management)).
10. **Use condition keys** (MFA required, source IP, VPC endpoint) to add defense-in-depth beyond basic action/resource grants ([Section 9](#9-condition-keys)).
11. **Consider ABAC with tags** once the organization grows large enough that per-resource RBAC policies become unmanageable ([Section 10](#10-tag-based-access-control-abac)).
12. **Use the policy simulator** before granting broad or unusual permissions in production ([Section 21](#21-iam-policy-simulator)).

---

## 24. Common Interview Questions

1. **What's the difference between an IAM user and an IAM role?** → [Section 4](#4-iam-roles) — long-term credentials vs. temporary, assumable credentials via STS.
2. **How does policy evaluation work when there are multiple policies?** → Explicit Deny always wins; otherwise need at least one applicable Allow within the intersection of all boundaries ([Section 8](#8-policy-evaluation-logic)).
3. **What's the difference between a Permissions Boundary and an SCP?** → [Section 12](#12-service-control-policies-scps) — per-identity cap vs. account/OU-wide cap; SCP requires AWS Organizations and affects the account's root user, a boundary does not.
4. **How do you grant a third-party SaaS vendor access to your account securely?** → Cross-account role with a unique `sts:ExternalId` condition ([Section 13](#13-cross-account-access)).
5. **How do you let corporate employees access AWS without creating IAM users for each one?** → SAML/OIDC federation, ideally via **IAM Identity Center** ([Sections 14–15](#14-identity-federation)).
6. **How do you enforce MFA for sensitive actions?** → A policy `Deny` statement conditioned on `aws:MultiFactorAuthPresent` being false ([Section 17](#17-multi-factor-authentication-mfa)).
7. **How do you find overly permissive or publicly accessible resources?** → **IAM Access Analyzer** ([Section 20](#20-iam-access-analyzer)).
8. **How do you right-size an overly broad IAM role?** → Review Access Advisor / Access Analyzer's policy generation based on actual CloudTrail activity ([Sections 20, 22](#20-iam-access-analyzer)).
9. **Same-account vs cross-account resource access — how does evaluation differ?** → Same-account: identity-based OR resource-based Allow is enough; cross-account: both must explicitly Allow ([Section 8](#8-policy-evaluation-logic)).
10. **What's ABAC and when would you use it over RBAC?** → Tag-based access control; use it when the number of teams/resources makes per-resource RBAC policies unmanageable ([Section 10](#10-tag-based-access-control-abac)).
11. **Can an SCP grant permissions?** → No — SCPs (and permissions boundaries) only ever restrict the maximum possible permissions; an identity-based or resource-based policy must still grant the actual permission ([Sections 11–12](#11-permissions-boundaries)).
12. **What happens to the root user under an SCP?** → SCPs do affect member account root users, but not the management account's own root user ([Section 18](#18-root-user-account)).
