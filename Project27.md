# AWS IAM SECURITY BEST PRACTICES

## Project Overview

This project demonstrates implementing AWS Identity and Access Management (IAM) security best practices. IAM is foundational to AWS security, and proper configuration is critical for securing cloud resources.

### Objectives
1. Understand IAM users, groups, roles, and policies
2. Implement least privilege access
3. Enable MFA for all users
4. Configure password policies
5. Set up AWS Organizations and Service Control Policies (SCPs)
6. Implement IAM Access Analyzer
7. Use IAM Identity Center for SSO
8. Monitor IAM with CloudTrail

## Prerequisites
- AWS Account with admin access
- AWS CLI installed and configured
- IAM users to manage

## Step 1: Set Up IAM Users and Groups

### Create IAM Users

```
bash
# Create IAM users
aws iam create-user --user-name developer
aws iam create-user --user-name devops-engineer
aws iam create-user --user-name security-auditor
aws iam create-user --user-name readonly-user

# Create login profile with MFA
aws iam create-login-profile \
    --user-name developer \
    --password "ComplexPassword123!" \
    --password-reset-required

# Create access keys
aws iam create-access-key --user-name developer
```

### Create IAM Groups

```
bash
# Create groups
aws iam create-group --group-name Developers
aws iam create-group --group-name DevOps
aws iam create-group --group-name SecurityTeam
aws iam create-group --group-name Auditors
aws iam create-group --group-name ReadOnly

# Add users to groups
aws iam add-user-to-group --user-name developer --group-name Developers
aws iam add-user-to-group --user-name devops-engineer --group-name DevOps
aws iam add-user-to-group --user-name security-auditor --group-name SecurityTeam
aws iam add-user-to-group --user-name readonly-user --group-name ReadOnly
```

### Create IAM Policies

```
bash
# Create policy for Developers
cat > developer-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:Describe*",
                "s3:GetObject",
                "s3:ListBucket",
                "logs:Describe*",
                "cloudwatch:Describe*",
                "cloudwatch:GetMetricStatistics"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::dev-bucket/*"
        }
    ]
}
EOF

aws iam create-policy \
    --policy-name DeveloperPolicy \
    --policy-document file://developer-policy.json

# Attach policy to group
aws iam attach-group-policy \
    --group-name Developers \
    --policy-arn arn:aws:iam::123456789012:policy/DeveloperPolicy
```

### Create Inline Policy for Specific Access

```
bash
# Create inline policy for S3 bucket access
cat > s3-access-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::my-app-bucket",
                "arn:aws:s3:::my-app-bucket/*"
            ]
        }
    ]
}
EOF

aws iam put-user-policy \
    --user-name developer \
    --policy-name S3ReadAccess \
    --policy-document file://s3-access-policy.json
```

## Step 2: Implement Least Privilege

### Create Role-Based Access

```
bash
# Create EC2 instance role
aws iam create-role \
    --role-name EC2WebServerRole \
    --assume-role-policy-document <<EOF
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
EOF

# Attach policies to role
aws iam attach-role-policy \
    --role-name EC2WebServerRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam attach-role-policy \
    --role-name EC2WebServerRole \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Create instance profile
aws iam create-instance-profile --instance-profile-name WebServerProfile
aws iam add-role-to-instance-profile \
    --instance-profile-name WebServerProfile \
    --role-name EC2WebServerRole
```

### Create Lambda Execution Role

```
bash
# Create Lambda execution role
aws iam create-role \
    --role-name LambdaExecutionRole \
    --assume-role-policy-document <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

# Attach Lambda basic execution policy
aws iam attach-role-policy \
    --role-name LambdaExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create custom policy for DynamoDB access
cat > lambda-dynamodb-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:Query",
                "dynamodb:Scan"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/UsersTable"
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name LambdaExecutionRole \
    --policy-name LambdaDynamoDBAccess \
    --policy-document file://lambda-dynamodb-policy.json
```

## Step 3: Enable MFA

### Enable MFA for Root Account

```
bash
# Generate virtual MFA device
aws iam create-virtual-mfa-device \
    --virtual-mfa-device-name root-mfa \
    --outfile ./mfaQRCode.png \
    --bootstrap-method QRCodePNG

# Enable MFA for root account
aws iam enable-mfa-device \
    --user-name root \
    --serial-number "arn:aws:iam::123456789012:mfa/root-mfa" \
    --authentication-code1 <code1> \
    --authentication-code2 <code2>
```

### Enforce MFA for IAM Users

```
bash
# Create MFA policy
cat > enforce-mfa-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyAllExceptListedIfNoMFA",
            "Effect": "Deny",
            "NotAction": [
                "iam:CreateVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:GetUser",
                "iam:ListMFADevices",
                "iam:ListUsers",
                "sts:GetSessionToken"
            ],
            "Resource": "*",
            "Condition": {
                "BoolIfExists": {
                    "aws:MultiFactorAuthPresent": "false"
                }
            }
        }
    ]
}
EOF

# Attach to group
aws iam create-policy \
    --policy-name EnforceMFAPolicy \
    --policy-document file://enforce-mfa-policy.json

aws iam attach-group-policy \
    --group-name Developers \
    --policy-arn arn:aws:iam::123456789012:policy/EnforceMFAPolicy
```

## Step 4: Password Policy

```
bash
# Create strong password policy
aws iam update-account-password-policy \
    --minimum-password-length 16 \
    --require-uppercase-characters \
    --require-lowercase-characters \
    --require-numbers \
    --require-symbols \
    --allow-users-to-change-password \
    --max-password-age 90 \
    --password-reuse-prevention 5

# View current password policy
aws iam get-account-password-policy
```

## Step 5: AWS Organizations and SCPs

### Create AWS Organization

```
bash
# Create organization
aws organizations create-organization

# Create organizational units (OUs)
aws organizations create-organizational-unit \
    --parent-id r-xxxx \
    --name Production

aws organizations create-organizational-unit \
    --parent-id r-xxxx \
    --name Development

aws organizations create-organizational-unit \
    --parent-id r-xxxx \
    --name Security

# Move accounts between OUs
aws organizations move-account \
    --account-id 123456789012 \
    --source-parent-id r-xxxx \
    --target-parent-id ou-xxxx-xxxxxxxx
```

### Create Service Control Policies

```
bash
# Deny access to specific services
cat > deny-s3-public-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyPublicS3Buckets",
            "Effect": "Deny",
            "Action": [
                "s3:PutBucketPublicAccessBlock",
                "s3:DeleteBucketPolicy"
            ],
            "Resource": "arn:aws:s3:::*/*",
            "Condition": {
                "ArnNotEquals": {
                    "aws:PrincipalARN": "arn:aws:iam::*:role/OrganizationAccountAccessRole"
                }
            }
        }
    ]
}
EOF

aws organizations create-policy \
    --content file://deny-s3-public-policy.json \
    --description "Deny modification of S3 public access" \
    --name DenyS3PublicAccess \
    --type SERVICE_CONTROL_POLICY

# Deny deletion of security resources
cat > deny-security-deletion-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenySecurityResourceDeletion",
            "Effect": "Deny",
            "Action": [
                "iam:DeleteUser",
                "iam:DeleteRole",
                "iam:DeletePolicy",
                "cloudtrail:DeleteTrail",
                "guardduty:DeleteDetector"
            ],
            "Resource": "*"
        }
    ]
}
EOF

# Attach policy to root
aws organizations attach-policy \
    --policy-id p-xxxx \
    --target-id r-xxxx
```

## Step 6: IAM Access Analyzer

### Enable Access Analyzer

```
bash
# Create analyzer (account-level)
aws accessanalyzer create-analyzer \
    --analyzer-name organization-analyzer \
    --type ORGANIZATION

# Create analyzer (account-level)
aws accessanalyzer create-analyzer \
    --analyzer-name account-analyzer \
    --type ACCOUNT

# List findings
aws accessanalyzer list-findings \
    --analyzer-arn <analyzer-arn>

# Get specific finding
aws accessanalyzer get-finding \
    --analyzer-arn <analyzer-arn> \
    --finding-arn <finding-arn>
```

### Archive Findings

```
bash
# Archive a finding
aws accessanalyzer archive-finding \
    --analyzer-arn <analyzer-arn> \
    --finding-arn <finding-arn>

# Create custom archive rule
cat > archive-rule.json <<EOF
{
    "ruleName": "InternalAccessRule",
    "filter": {
        "principal": {
            "eq": ["arn:aws:iam::123456789012:root"]
        },
        "resource": {
            "eq": ["arn:aws:s3:::internal-bucket/*"]
        }
    }
}
EOF

aws accessanalyzer create-archive-rule \
    --analyzer-arn <analyzer-arn> \
    --rule file://archive-rule.json
```

## Step 7: AWS IAM Identity Center (SSO)

### Enable IAM Identity Center

```
bash
# Enable IAM Identity Center
aws sso-admin create-permission-set \
    --instance-arn <instance-arn> \
    --name AdministratorAccess \
    --session-duration 43200

# Create permission set
aws sso-admin create-permission-set \
    --instance-arn <instance-arn> \
    --name DeveloperAccess \
    --session-duration 28800 \
    --relay-state "urn:amzn:homeRegion:us-east-1"

# Get managed policy ARN
aws sso-admin list-managed-policies-in-permission-set \
    --instance-arn <instance-arn> \
    --permission-set-arn <permission-set-arn>

# Attach managed policy
aws sso-admin attach-managed-policy-to-permission-set \
    --instance-arn <instance-arn> \
    --permission-set-arn <permission-set-arn> \
    --managed-policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Assign user to account
aws sso-admin create-account-assignment \
    --instance-arn <instance-arn> \
    --permission-set-arn <permission-set-arn> \
    --principal-id <user-id> \
    --principal-type USER \
    --target-account-id 123456789012
```

## Step 8: CloudTrail Integration

### Set Up CloudTrail

```
bash
# Create S3 bucket for CloudTrail logs
aws s3 mb s3://my-cloudtrail-logs-bucket

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket my-cloudtrail-logs-bucket \
    --versioning-configuration Status=Enabled

# Create trail
aws cloudtrail create-trail \
    --name my-trail \
    --s3-bucket-name my-cloudtrail-logs-bucket \
    --is-multi-region-trail \
    --enable-log-file-validation

# Start logging
aws cloudtrail start-logging \
    --name my-trail

# Look up IAM events
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventSource,AttributeValue=iam.amazonaws.com
```

### Analyze IAM Events with CloudWatch

```
bash
# Create CloudWatch Logs group
aws logs create-log-group \
    --log-group-name /aws/cloudtrail/my-trail

# Configure CloudTrail to send to CloudWatch
aws cloudtrail update-trail \
    --name my-trail \
    --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:/aws/cloudtrail/my-trail \
    --cloud-watch-logs-role-arn arn:aws:iam::123456789012:role/CloudTrail-CloudWatch

# Create metric filter
aws logs put-metric-filter \
    --log-group-name /aws/cloudtrail/my-trail \
    --filter-name IAMRootLogin \
    --metric-transformations metricName=RootLogin,metricNamespace=CloudTrailMetrics,metricValue=1 \
    --pattern '{ $.userIdentity.type = "Root" && $.eventType = "AwsConsoleSignIn" }'

# Create alarm
aws cloudwatch put-metric-alarm \
    --alarm-name RootLoginAlarm \
    --metric-name RootLogin \
    --namespace CloudTrailMetrics \
    --statistic Sum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1 \
    --alarm-actions <sns-topic-arn>
```

## Step 9: IAM Best Practices Automation

### Use AWS Config Rules

```
bash
# Enable AWS Config
aws configservice put-configuration-recorder \
    --configuration-recorder name=default \
    --recording-group '{"allSupported":true,"includeGlobalResourceTypes":true}'

# Create conformance pack
cat > iam-conformance-pack.yaml <<EOF
Resources:
  MfaEnabledForIamUsers:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: iam-user-mfa-enabled
      Source:
        Owner: AWS
        SourceIdentifier: IAM_USER_MFA_ENABLED
      MaximumExecutionFrequency: TwentyFour_Hours
  
  RootAccountMfaEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: root-account-mfa-enabled
      Source:
        Owner: AWS
        SourceIdentifier: ROOT_ACCOUNT_MFA_ENABLED
  
  IamPasswordPolicy:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: iam-password-policy
      Source:
        Owner: AWS
        SourceIdentifier: IAM_PASSWORD_POLICY
  
  AccessKeysRotated:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: access-keys-rotated
      Source:
        Owner: AWS
        SourceIdentifier: ACCESS_KEYS_ROTATED
      InputParameters:
        maxCredentialUsageAge: 90
EOF

aws configservice put-conformance-pack \
    --conformance-pack-name iam-security \
    --template-body file://iam-conformance-pack.yaml
```

## Step 10: Security Summary Script

```
bash
#!/bin/bash
# iam-security-check.sh

echo "=== IAM Security Check Report ==="
echo ""

echo "1. MFA Status for Users:"
aws iam list-users --query 'Users[].{Name:UserName,MFAAccess:Tags[?Key==`MFA`].Value}'
echo ""

echo "2. Users with Access Keys:"
aws iam list-users --query 'Users[*].UserName'
echo ""

echo "3. Password Last Used:"
aws iam get-credential-report --output text --query 'Content' | cut -d',' -f1,5,11,13
echo ""

echo "4. Security Policies:"
aws iam list-policies --scope Local --query 'Policies[].PolicyName'
echo ""

echo "5. Roles:"
aws iam list-roles --query 'Roles[].RoleName'
echo ""

echo "6. Groups:"
aws iam list-groups --query 'Groups[].GroupName'
echo ""

echo "7. Password Policy:"
aws iam get-account-password-policy --query 'PasswordPolicy'
echo ""

echo "8. MFA Devices:"
aws iam list-mfa-devices --query 'MFADevices[].{UserName:UserName,SerialNumber:SerialNumber}'
echo ""

echo "9. Attached Policies:"
aws iam list-attached-user-policies --user-name <username> --query 'AttachedPolicies[].PolicyName'
echo ""

echo "=== Report Complete ==="
```

## Conclusion

This project demonstrates:
- Creating and managing IAM users and groups
- Implementing least privilege access with policies
- Creating IAM roles for different services
- Enforcing MFA for all users
- Setting up strong password policies
- Using AWS Organizations and Service Control Policies
- Implementing IAM Access Analyzer
- Setting up IAM Identity Center (SSO)
- CloudTrail integration for monitoring
- AWS Config rules for compliance
- Security automation scripts

## Additional Resources
- [AWS IAM Documentation](https://docs.aws.amazon.com/iam/)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS Security Hub](https://aws.amazon.com/security-hub/)
- [AWS Config](https://aws.amazon.com/config/)
