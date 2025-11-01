## IAM

When you assign an IAM role to an EC2 instance, you are actually associating an instance profile with the instance. An instance profile is a container for an IAM role that allows the EC2 instance to assume that role.

- Assigning the Role: When you launch an EC2 instance, you can select an IAM role to attach. This action creates an instance profile behind the scenes (or uses an existing one) and associates it with the instance. This instance profile has a trust policy that permits the EC2 service to assume the attached IAM role.

- Requesting Credentials: An application running on the EC2 instance can make an HTTP request to the IMDS endpoint, which is a special link-local IPv4 address: http://169.254.169.254. This address is not routable outside of the instance's network. The application can query a specific path on this endpoint to retrieve the temporary security credentials.

        For example, using the AWS CLI or an AWS SDK, the library automatically knows to check the IMDS for credentials if no other credentials are provided.

- IMDSv2 and Security: AWS introduced IMDSv2 to address security vulnerabilities like Server-Side Request Forgery (SSRF). With IMDSv2, a simple GET request isn't enough. The application must first make a PUT request to get a session token, and then use that token in all subsequent GET requests to the metadata service. This makes it significantly harder for an attacker to steal credentials if they can trick your application into making a request on their behalf.

- Using the Credentials: Once the application receives the temporary credentials (an access key ID, a secret access key, and a session token) from the IMDS, it uses them to sign requests to other AWS services, such as S3, DynamoDB, or Secrets Manager. These temporary credentials expire after a certain time (usually a few hours) and are automatically rotated by the IMDS, meaning the application can simply make another request to the endpoint to get fresh credentials.


AWS IAM Roles is the aws equivalence of azure managed identity

In AWS, instead of manually managing credentials (like access keys), you attach IAM roles to AWS resources. These roles provide temporary security credentials that applications can use to call AWS services securely


- EC2 Instance Role
You assign an IAM role to an EC2 instance.
Applications on that instance automatically get temporary credentials through the Instance Metadata Service (IMDS).
This is very similar to Azure’s system-assigned managed identity.
- Lambda Execution Role
Each AWS Lambda function can have an execution role.
When the function runs, AWS automatically provides temporary credentials for that role.
This is analogous to Azure Functions with managed identity.
- ECS/EKS Task Roles
For ECS tasks or Kubernetes pods on EKS, you can assign task roles / pod roles.
They receive short-lived credentials via the AWS SDK.

### How It Works (in AWS)
- The application (running on EC2, Lambda, ECS, etc.) requests credentials from AWS STS (Security Token Service) via the metadata endpoint.
- AWS automatically rotates these credentials.
- The application uses the AWS SDK, which automatically retrieves and refreshes credentials when an IAM role is attached.

Let’s say your Lambda/EC2 has a role that allows access to AWS Secrets Manager:

```py
import boto3

# Boto3 will automatically use IAM role credentials provided by AWS
client = boto3.client("secretsmanager")

response = client.get_secret_value(SecretId="MySecret")

print("Secret Value:", response["SecretString"])
```

nstance Metadata Service (IMDS), which under the hood calls AWS STS (Security Token Service) to get temporary credentials.


## How the Flow Works
1. IAM Role Attached to Resource
When you launch an EC2 instance (or Lambda/ECS), you attach an IAM Role.
That role has a trust policy that says, for example: “EC2 instances can assume me.”
2. Application Requests Credentials
Inside the EC2 instance:
The AWS SDK (e.g., boto3, AWS CLI, Java SDK) queries the IMDS endpoint:
`http://169.254.169.254/latest/meta-data/iam/security-credentials/`
This endpoint is only accessible from inside the instance

3. Metadata Service Contacts STS
When IMDS receives the request, the AWS infrastructure behind the scenes calls STS AssumeRole on your behalf.
STS issues a short-lived credential set:
- AccessKeyId
- SecretAccessKey
- SessionToken
- Expiration time (usually ~6 hours for EC2 roles, shorter for Lambda)

4. Metadata Service Returns Credentials
If you query the endpoint directly:
`curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>`
You’ll get a JSON response like:
```json
{
  "Code" : "Success",
  "LastUpdated" : "2025-09-19T12:00:00Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIAxxxxxxxxxxxx",
  "SecretAccessKey" : "wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY",
  "Token" : "FQoGZXIvYXdzEK7//////////wEaDK...",
  "Expiration" : "2025-09-19T18:00:00Z"
}
```

5. Application Uses Credentials
The AWS SDK automatically picks these up, signs requests with SigV4 signing, and talks to S3, DynamoDB, etc.
When credentials expire, the SDK transparently refreshes them by hitting IMDS again


AWS Identity and Access Management (IAM) is the security backbone of AWS. It controls:
- Who (users, apps, services) can access AWS.
- What they can do (permissions, actions).
- How they authenticate (passwords, keys, roles, federation)

## Users
Represent people or applications that need long-term credentials.
Can log in to AWS Management Console or use AWS CLI/SDK with access keys.
Best practice: Avoid IAM users for apps → use roles instead.

## Groups
Collections of IAM users.
You attach policies to groups → all users inherit permissions.
Example: Developers group with read-only S3 access

## Roles
Identities that AWS services or users assume temporarily.
No permanent credentials — only temporary STS tokens.
Used for EC2, Lambda, ECS, EKS pods, cross-account access, and federation

## Policies
JSON documents that define permissions.
Example:
```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```
Can be attached to users, groups, or roles.

## Types of Policies
- Identity-based (attached to user, group, or role).
- Resource-based (attached to the resource, e.g., S3 bucket policy).
- Permission boundaries → limit what a user/role can do, even if their policy is broader.
- Service control policies (SCPs) → organization-wide restrictions.
- Session policies → applied when assuming roles, temporary.


### EC2 instance accessing DynamoDB
- EC2 has an instance role.
- App on EC2 queries IMDS → STS → temporary credentials.
- App calls DynamoDB API with those credentials.

## Permission Policies (a.k.a. “what you can do”)
Attached to users, groups, or roles.
Define what actions the identity is allowed (or denied) to perform on which resources.
Example:
```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```
This means: “If you are this user/role, you’re allowed to get objects from S3.”

## Trust Policies (a.k.a. “who can assume me”)
Found only on IAM roles (not users or groups).
Define who is allowed to assume this role.
Example:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
This means: “EC2 instances are allowed to assume this role.”

a trust policy answers “Who can wear this badge?”, while a permission policy answers “What can you do once you’re wearing it?”


When a request is made with a role:
Trust policy is checked first: Is this identity allowed to assume the role?
If no → blocked.
If yes, AWS STS issues temporary credentials.
Those credentials are evaluated against the permission policy of the role.

### EC2 instance reading S3
EC2 wants to assume role EC2ReadS3Role.
Trust policy:
```json
{
  "Principal": {"Service": "ec2.amazonaws.com"},
  "Action": "sts:AssumeRole"
}
```
Allowed → EC2 can assume role.

Once assumed, role’s permission policy:
{
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Effect": "Allow"
}
Allows reading S3 objects.

```sh
Yes, every IAM role must have a trust policy.
Without it, nobody (no service, no user, no account) could assume the role
```
Even if you attach permissions, the role is useless if nobody is trusted to use it.
- Permission policies, however, are optional.
A role can exist with no identity policies attached.
In that case, the role can be assumed (if trust allows), but the temporary credentials won’t grant access to do anything.
`Effectively, the role has no power until you give it permission policies`

IAM Roles and Their Policies
An IAM Role can have two different types of policies tied to it:

- Trust Policy (always present)
Special JSON document attached directly to the role.
Defines who is allowed to assume the role.
Found under the "Trust relationships" tab in the AWS Console.
Example:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

```

- Identity-based Policies (optional, but usually present)
Normal IAM policies you attach to a role.
Define what the role can do once it has been assumed.
Can be:
 - AWS managed policies (AmazonS3ReadOnlyAccess, etc.)
 - Customer managed policies
 - Inline policies

 ```json
 Example:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```
This says: “Anyone who assumes this role can read from my-bucket.”

IAM roles always have a trust policy (mandatory) and can also have identity-based policies (permissions).


### AWS Principal (IAM Users, Roles, Accounts)
Represents AWS identities: users, roles, or entire accounts.
Used for trusting a specific account, role, or user.

```json
"Principal": {
  "AWS": "arn:aws:iam::123456789012:role/SomeRole"
}
```

```json
"Principal": {
  "AWS": "arn:aws:iam::123456789012:user/Alice"
}
```

```json
"Principal": {
  "AWS": "arn:aws:iam::123456789012:root"
}
```
Trusts the entire account 123456789012

## Service Principal
Represents an AWS service that can assume the role.
Common for EC2, Lambda, ECS, etc.

```json
"Principal": {
  "Service": "ec2.amazonaws.com"
}
```
Allows EC2 instances to assume the role.

```json
"Principal": {
  "Service": "lambda.amazonaws.com"
}
```

## Federated Principal

Used for federated identity providers (IdPs).
Examples: SAML, OIDC (e.g., Cognito, Google, Okta, GitHub Actions).

```json
"Principal": {
  "Federated": "arn:aws:iam::123456789012:saml-provider/MyCompanyIdP"
}
```

Trusts a SAML provider.

```json
"Principal": {
  "Federated": "arn:aws:iam::123456789012:oidc-provider/accounts.google.com"
}

```

Trusts an OIDC provider (like Google or GitHub)


## Combination of Principals
You can specify multiple principals at once.

```json
"Principal": {
  "AWS": [
    "arn:aws:iam::123456789012:role/SomeRole",
    "arn:aws:iam::111111111111:user/Bob"
  ],
  "Service": "ec2.amazonaws.com"
}

```
Trusts both a role, a user, and EC2 service.


## SAML 2.0 Providers
Used for enterprise Single Sign-On (SSO) (corporate Active Directory, Okta, Ping, ADFS, etc.).
You create an IAM SAML provider in AWS → get an ARN like:
`arn:aws:iam::<account-id>:saml-provider/MyCompanyADFS`
In the role trust policy:
```json
"Principal": {
  "Federated": "arn:aws:iam::123456789012:saml-provider/MyCompanyADFS"
}
```

- User authenticates with corporate IdP.
- IdP issues a signed SAML assertion.
- AWS STS issues temporary role credentials.


## OIDC Providers (OpenID Connect)
- Used for modern web/mobile apps and workload identity federation.
- AWS supports external OIDC providers like:
- Google (accounts.google.com)
- GitHub (token.actions.githubusercontent.com)
- Okta OIDC
- Auth0
- Amazon Cognito (as IdP itself)
In the role trust policy:

```json
"Principal": {
  "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
}

```

- App authenticates with IdP → gets an OIDC token (JWT).
- Token is exchanged for AWS STS credentials.

Best for: CI/CD (GitHub Actions, GitLab CI), Kubernetes pods (EKS IRSA), mobile apps.

## Amazon Cognito Federated Identities
Cognito lets you federate social IdPs (Google, Facebook, Apple, Amazon) or enterprise IdPs via SAML/OIDC.
Cognito issues temporary AWS credentials by assuming a role on behalf of the federated user.
Example role trust policy:
```json
"Principal": {
  "Federated": "cognito-identity.amazonaws.com"
}
```
With conditions, you map users to specific IAM roles.

Actions Used with Federated Principals
- `sts:AssumeRoleWithSAML` → for SAML providers.
- `sts:AssumeRoleWithWebIdentity` → for OIDC and Cognito.
- `sts:AssumeRole` → for cross-account AWS principals (not external IdPs).


## GitHub Actions → AWS (CI/CD Pipelines)
Problem: Traditionally, CI/CD pipelines used long-lived IAM access keys stored as GitHub secrets → risky if leaked.

### Solution: Use OIDC federation with GitHub as IdP:
- GitHub Actions → authenticates to token.actions.githubusercontent.com.
- AWS IAM role trusts this OIDC provider with conditions (specific repo/branch/workflow).
- Pipeline assumes role via sts:AssumeRoleWithWebIdentity.
Trust policy snippet:
```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
    }
  }
}
```

### Use Cases:
- Deploy CloudFormation / Terraform from GitHub CI.
- Push Docker images to Amazon ECR.
- Deploy serverless apps to AWS Lambda.

This is now the best practice for GitHub → AWS

GitHub Actions → AWS OIDC flow diagram:
- GitHub Actions workflow requests an OIDC token from - GitHub’s OIDC provider.
- AWS STS verifies and exchanges the token.
- STS allows the workflow to assume an IAM role (per trust policy + conditions).
- The workflow uses the temporary credentials to access AWS services (S3, ECR, Lambda, etc.).

Prereq (one-time)
Create the OIDC provider in IAM (if you haven’t):
Provider URL: https://token.actions.githubusercontent.com
Audience: sts.amazonaws.com
No thumbprint needed today; AWS manages GitHub’s root CA for you.

1. Lock to a single repo & branch (most common)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:ref:refs/heads/<BRANCH>"
      }
    }
  }]
}
```

Example sub value: `repo:my-org/payments-api:ref:refs/heads/main`


## Deploy a Static Website to S3 (+ CloudFront Invalidation)
Trust policy (lock to main branch)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:ref:refs/heads/main" }
    }
  }]
}
```

Minimal permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:PutObject","s3:DeleteObject","s3:ListBucket"], "Resource": [
      "arn:aws:s3:::<BUCKET>",
      "arn:aws:s3:::<BUCKET>/*"
    ]},
    { "Effect": "Allow", "Action": ["cloudfront:CreateInvalidation"], "Resource": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>" }
  ]
}
```

GitHub workflow

```yml
name: deploy-site
on: { push: { branches: [ main ] } }
permissions: { id-token: write, contents: read }

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
          aws-region: <REGION>

      - run: aws s3 sync ./dist s3://<BUCKET> --delete
      - run: aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/*"
```


## Push Docker Image to ECR and Update ECS Service
Trust policy (lock to release tags v*)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:ref:refs/tags/v*" }
    }
  }]
}

```

Minimal permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["ecr:GetAuthorizationToken"], "Resource": "*" },
    { "Effect": "Allow", "Action": [
        "ecr:BatchCheckLayerAvailability","ecr:CompleteLayerUpload",
        "ecr:UploadLayerPart","ecr:InitiateLayerUpload","ecr:PutImage"
      ], "Resource": "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/<ECR_REPO>" },
    { "Effect": "Allow", "Action": [
        "ecs:UpdateService","ecs:DescribeServices","ecs:RegisterTaskDefinition","iam:PassRole"
      ], "Resource": "*" }  // scope to specific ARNs if you can
  ]
}

```

GitHub workflow

```yml
name: build-and-deploy
on: { push: { tags: [ "v*" ] } }
permissions: { id-token: write, contents: read }

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
          aws-region: <REGION>

      - id: login
        uses: aws-actions/amazon-ecr-login@v2

      - run: |
          IMAGE="${{ steps.login.outputs.registry }}/<ECR_REPO>:${GITHUB_REF_NAME}"
          docker build -t "$IMAGE" .
          docker push "$IMAGE"

      - name: Update ECS service
        run: |
          # Example using new task def JSON
          aws ecs register-task-definition --cli-input-json file://taskdef.json
          aws ecs update-service --cluster <CLUSTER> --service <SERVICE> --force-new-deployment

```

The JWT from token.actions.githubusercontent.com contains useful claims, for example:
```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "aud": "sts.amazonaws.com",
  "sub": "repo:myorg/myrepo:ref:refs/heads/main",
  "repository": "myorg/myrepo",
  "ref": "refs/heads/main",
  "workflow": "deploy.yml",
  "job_workflow_ref": "myorg/myrepo/.github/workflows/deploy.yml@refs/heads/main"
}
```