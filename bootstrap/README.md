# Bootstrap Setup Guide

This directory contains the initial setup configuration for the AWS Multi-Tier Infrastructure Platform, focusing on secure GitHub Actions integration using OIDC.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [OIDC Setup Process](#oidc-setup-process)
3. [EC2 Key Pair Setup](#ec2-key-pair-setup)
4. [Security Considerations](#security-considerations)
5. [Validation & Troubleshooting](#validation--troubleshooting)
6. [Next Steps](#next-steps)

## Prerequisites

### AWS Account Setup

1. **AWS CLI Configuration**
   ```bash
   aws configure
   ```

2. **Required AWS Accounts**
   - Development account
   - Staging account  
   - Production account

### GitHub Repository Setup

1. **Fork or Clone Repository**
2. **Create GitHub Environments** (optional for enhanced security):
   - `development`
   - `staging` 
   - `production`

## OIDC Setup Process

The `gha-oidc.yaml` template creates both the OIDC provider and IAM role needed for GitHub Actions to securely authenticate with AWS.

### Deployment Options

**Option 1: AWS Console (Recommended)**

1. Navigate to CloudFormation in each AWS account (dev, staging, prod)
2. Choose "Create stack"
3. Upload the `bootstrap/gha-oidc.yaml` template
4. Provide the required parameters:
   - `GitHubOrgUser`: Your GitHub organization or username
   - `RepositoryName`: Your repository name
5. Acknowledge IAM capabilities and create stack

**Option 2: AWS CLI**

```bash
aws cloudformation create-stack \
  --stack-name oidc-github-role-dev \
  --template-body file://bootstrap/gha-oidc.yaml \
  --parameters \
    ParameterKey=GitHubOrgUser,ParameterValue=YOUR-GITHUB-ORG \
    ParameterKey=RepositoryName,ParameterValue=YOUR-REPO-NAME \
  --capabilities CAPABILITY_NAMED_IAM
```

### Required Parameters

| Parameter | Description | Example | Required |
|-----------|-------------|---------|----------|
| `GitHubOrgUser` | GitHub organization or username (case-sensitive) | `mycompany` | Yes |
| `RepositoryName` | Repository name (case-sensitive) | `aws-infrastructure` | Yes |
| `RoleName` | IAM role name for GitHub Actions | `GitHubActionsRole` | No (default) |
| `OIDCAudience` | OIDC audience for AWS STS | `sts.amazonaws.com` | No (default) |

**Example Configuration:**
For repository `https://github.com/mycompany/aws-infrastructure`:
- `GitHubOrgUser`: `mycompany`
- `RepositoryName`: `aws-infrastructure`

## EC2 Key Pair Setup

The application servers require an EC2 key pair for deployment and potential troubleshooting access. This key pair name is referenced in the app-parameters.json files.

Create a key pair named `aws-demo` in each account:

```bash
aws ec2 create-key-pair \
  --key-name aws-demo \
  --query 'KeyMaterial' \
  --output text > aws-demo.pem

chmod 400 aws-demo.pem
```

**Note**: The key pair name `aws-demo` is configured in the `parameters/*/app-parameters.json` files. If you use a different name, update the `KeyName` parameter accordingly.

## Security Considerations

### Why OIDC Instead of IAM Keys?

- **No Long-lived Credentials**: OIDC tokens are temporary (1 hour max)
- **Least Privilege**: Roles can be scoped to specific repositories and branches
- **Auditable**: CloudTrail logs show exactly which GitHub workflow assumed which role
- **Revocable**: Disable the OIDC provider to immediately revoke all access

### IAM Role Permissions

The `gha-oidc.yaml` template creates a role with permissions for:
- CloudFormation stack operations
- EC2 resource management
- RDS operations
- VPC and networking
- IAM role management (limited scope)
- CloudWatch and logging

### Trust Relationship

Roles trust your specific GitHub organization/repository with branch-level access controls.

## Validation & Troubleshooting

**Test Setup:** Push a change to trigger workflow and verify no authentication errors in GitHub Actions logs.

**Common Issues:**
- **Trust Relationship Errors**: Verify GitHub org/user and repository name in parameters
- **Permission Denied**: Review IAM policies and CloudTrail logs
- **Role ARN Not Found**: Confirm stacks deployed successfully

## Next Steps

After completing the bootstrap setup:

### 1. Configure GitHub Secrets

Add these repository secrets in GitHub:

**AWS Role ARNs:**
- `AWS_ROLE_ARN_DEV`: ARN from development account OIDC stack
- `AWS_ROLE_ARN_STAGING`: ARN from staging account OIDC stack  
- `AWS_ROLE_ARN_PROD`: ARN from production account OIDC stack
- `AWS_REGION`: AWS region for deployments (optional, defaults to us-east-1)

**Database Passwords:**
- `DB_PASSWORD_DEV`: Secure password for development RDS instance
- `DB_PASSWORD_STAGING`: Secure password for staging RDS instance
- `DB_PASSWORD_PROD`: Secure password for production RDS instance

**To get the role ARNs:**
1. Go to AWS CloudFormation Console
2. Select the `oidc-github-role-dev` stack (or corresponding stack name for each environment)
3. Click on the "Outputs" tab
4. Copy the `RoleArn` value

<img src="../assets/OIDC%20Role%20Arn.png" alt="OIDC Role ARN from CloudFormation Outputs" width="800" />

**GitHub Secrets Configuration:**

Configure the repository secrets in GitHub as shown below:

<img src="../assets/GH%20secrets.png" alt="GitHub Secrets Configuration" width="800" />

### 2. Additional Setup Steps

1. **Update Parameter Files**: Modify files in `parameters/` directories for your specific requirements
2. **Test Deployment**: Start with development environment to validate configuration
3. **View Results**: After successful deployment, refer to the main README's [Demo section](../README.md#demo) to see how the pipeline and stacks look