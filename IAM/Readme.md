## AWS IAM Policy Type

### Core IAM Policy Types

| Policy Type              | Description                                                                                  | Use Case Example                                  |
|--------------------------|----------------------------------------------------------------------------------------------|---------------------------------------------------|
| **Identity-based**       | Attached to IAM users, groups, or roles. Grants permissions to those identities.             | Allow a user to access S3 buckets.                |
| **Resource-based**       | Attached directly to AWS resources (e.g., S3 buckets, Lambda functions).                     | Allow cross-account access to an S3 bucket.       |
| **Managed Policies**     | Standalone policies managed by AWS or the user. Can be reused across identities.             | AWS-managed: `AmazonEC2ReadOnlyAccess`<br>Customer-managed: Custom EC2 access policy |
| **Inline Policies**      | Embedded directly into a single IAM identity. Cannot be reused.                              | One-off permissions for a specific user.          |

###  Advanced Policy Types

| Policy Type                   | Description                                                                                      | Use Case Example                                  |
|-------------------------------|--------------------------------------------------------------------------------------------------|---------------------------------------------------|
| **Permissions Boundaries**    | Defines the maximum permissions an identity-based policy can grant.                             | Limit a developer role to read-only access.       |
| **Service Control Policies (SCPs)** | Used in AWS Organizations to define max permissions for accounts or OUs.                       | Prevent accounts from deleting CloudTrail logs.   |
| **Session Policies**          | Passed during role assumption to limit permissions for the session duration.                    | Temporary access with restricted actions.         |
| **Access Control Lists (ACLs)** | Legacy mechanism for controlling access to resources like S3 and Amazon VPC.                    | Grant public read access to an S3 object.         |
## AWS IAM Roles

An IAM role is an identity you create in AWS with a defined set of permissions to perform actions on AWS resources. Unlike an IAM user, a role does not have long-term credentials such as passwords or access keys. Instead, when an entity assumes a role, AWS Security Token Service (STS) issues temporary security credentials for that session.

### Core Components

- Trust policy  
  Defines which principals (users, services, or accounts) are allowed to assume the role.  

- Permissions policy  
  Specifies the actions and resources the role can perform or access once assumed.  

- Session duration  
  Determines how long the temporary credentials remain valid (up to 12 hours by default).

### Common Use Cases

- EC2 instance profiles granting instances access to S3 buckets without embedding credentials.  
- Lambda execution roles enabling functions to read/write DynamoDB or publish to SNS.  
- Cross-account access for auditing or data sharing between separate AWS accounts.  
- Federation for corporate users via SAML or third-party OIDC providers.  
