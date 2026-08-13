# AWS CLOUDFORMATION INFRASTRUCTURE AS CODE

## Project Overview

This project demonstrates using AWS CloudFormation to define and provision AWS infrastructure using templates. CloudFormation allows you to treat your infrastructure as code, making it versionable, repeatable, and manageable.

### Objectives
1. Understand CloudFormation template structure
2. Create stacks with various AWS resources
3. Implement nested stacks and stack sets
4. Use CloudFormation macros and transformations
5. Implement drift detection
6. Use CloudFormation Guard for policy validation
7. Create CI/CD pipelines for CloudFormation

## Prerequisites
- AWS Account
- AWS CLI installed and configured
- Basic understanding of AWS services

## Step 1: Create Basic CloudFormation Stack

### Simple S3 Bucket Template

```
yaml
# s3-bucket.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple S3 Bucket CloudFormation Template

Resources:
  # S3 Bucket
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'my-unique-bucket-${AWS::AccountId}'
      AccessControl: Private
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      VersioningConfiguration:
        Status: Enabled
      Tags:
        - Key: Environment
          Value: Production
        - Key: Project
          Value: CloudFormation

Outputs:
  BucketName:
    Description: Name of the S3 bucket
    Value: !Ref MyS3Bucket
    Export:
      Name: !Sub '${AWS::StackName}-BucketName'
  
  BucketARN:
    Description: ARN of the S3 bucket
    Value: !GetAtt MyS3Bucket.Arn
    Export:
      Name: !Sub '${AWS::StackName}-BucketARN'
```

### Deploy Stack

```
bash
# Create stack
aws cloudformation create-stack \
    --stack-name my-s3-stack \
    --template-body file://s3-bucket.yaml \
    --capabilities CAPABILITY_IAM

# Update stack
aws cloudformation update-stack \
    --stack-name my-s3-stack \
    --template-body file://s3-bucket.yaml \
    --capabilities CAPABILITY_IAM

# Describe stack
aws cloudformation describe-stacks \
    --stack-name my-s3-stack

# Delete stack
aws cloudformation delete-stack --stack-name my-s3-stack
```

## Step 2: VPC with Multiple Resources

### Complete VPC Template

```
yaml
# vpc.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC with Public and Private Subnets

Parameters:
  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
    Description: CIDR block for VPC
  
  PublicSubnet1CIDR:
    Type: String
    Default: 10.0.1.0/24
    Description: CIDR for Public Subnet 1
  
  PublicSubnet2CIDR:
    Type: String
    Default: 10.0.2.0/24
    Description: CIDR for Public Subnet 2
  
  PrivateSubnet1CIDR:
    Type: String
    Default: 10.0.10.0/24
    Description: CIDR for Private Subnet 1
  
  PrivateSubnet2CIDR:
    Type: String
    Default: 10.0.11.0/24
    Description: CIDR for Private Subnet 2

Resources:
  # VPC
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-VPC'

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-IGW'

  # Attach IGW to VPC
  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  # Public Subnet 1
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PublicSubnet1CIDR
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-PublicSubnet1'
        - Key: Type
          Value: Public

  # Public Subnet 2
  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PublicSubnet2CIDR
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-PublicSubnet2'
        - Key: Type
          Value: Public

  # Private Subnet 1
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PrivateSubnet1CIDR
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-PrivateSubnet1'
        - Key: Type
          Value: Private

  # Private Subnet 2
  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PrivateSubnet2CIDR
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-PrivateSubnet2'
        - Key: Type
          Value: Private

  # NAT Gateway Elastic IP
  NatEIP:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway
    Properties:
      Domain: vpc

  # NAT Gateway
  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-NAT'

  # Public Route Table
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-PublicRT'

  # Public Route
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  # Private Route Table
  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-PrivateRT'

  # Private Route
  PrivateRoute:
    Type: AWS::EC2::Route
    DependsOn: NatGateway
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway

  # Route Table Associations
  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable

Outputs:
  VPCId:
    Description: ID of the VPC
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCID'

  PublicSubnet1ID:
    Description: ID of Public Subnet 1
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub '${AWS::StackName}-PublicSubnet1'

  PublicSubnet2ID:
    Description: ID of Public Subnet 2
    Value: !Ref PublicSubnet2
    Export:
      Name: !Sub '${AWS::StackName}-PublicSubnet2'

  PrivateSubnet1ID:
    Description: ID of Private Subnet 1
    Value: !Ref PrivateSubnet1
    Export:
      Name: !Sub '${AWS::StackName}-PrivateSubnet1'

  PrivateSubnet2ID:
    Description: ID of Private Subnet 2
    Value: !Ref PrivateSubnet2
    Export:
      Name: !Sub '${AWS::StackName}-PrivateSubnet2'
```

## Step 3: EC2 Instance with Security Group

```
yaml
# ec2-instance.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: EC2 Instance with Security Group

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
    Description: EC2 instance type
  
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Name of an existing EC2 KeyPair
  
  SSHLocation:
    Type: String
    Default: 0.0.0.0/0
    Description: IP range for SSH access

Mappings:
  RegionAMI:
    us-east-1:
      AMI: ami-0c55b159cbfafe1f0
    us-west-2:
      AMI: ami-0892d3c7ee96c0bf7
    eu-west-1:
      AMI: ami-0d71ea30463e0ff8d

Resources:
  # Security Group
  WebServerSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable HTTP and SSH access
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: !Ref SSHLocation
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-SG'

  # EC2 Instance
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !FindInMap [RegionAMI, !Ref 'AWS::Region', AMI]
      SecurityGroups:
        - !Ref WebServerSG
      SubnetId: !ImportValue MyVPC-PublicSubnet1
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "<h1>Deployed via CloudFormation</h1>" > /var/www/html/index.html

Outputs:
  InstanceId:
    Description: Instance ID
    Value: !Ref WebServer
  
  PublicIP:
    Description: Public IP Address
    Value: !GetAtt WebServer.PublicIp
  
  PublicDNS:
    Description: Public DNS Name
    Value: !GetAtt WebServer.PublicDnsName
```

## Step 4: RDS Database Instance

```
yaml
# rds-instance.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: RDS MySQL Instance

Parameters:
  DBInstanceClass:
    Type: String
    Default: db.t3.micro
    AllowedValues:
      - db.t3.micro
      - db.t3.small
      - db.t3.medium
    Description: Database instance class
  
  DBName:
    Type: String
    Default: mydatabase
    Description: Database name
  
  MasterUsername:
    Type: String
    Default: admin
    Description: Master username
  
  MasterUserPassword:
    Type: String
    NoEcho: true
    Description: Master password
    MinLength: 8
    MaxLength: 41
    AllowedPattern: '[a-zA-Z0-9]+'
    ConstraintDescription: Must contain only alphanumeric characters

Resources:
  # Subnet Group
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Subnet group for RDS
      SubnetIds:
        - !ImportValue MyVPC-PrivateSubnet1
        - !ImportValue MyVPC-PrivateSubnet2
  
  # Security Group
  RDSSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: RDS Security Group
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          CidrIp: 10.0.0.0/16
  
  # DB Instance
  MyDB:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: !Sub '${AWS::StackName}-db'
      DBInstanceClass: !Ref DBInstanceClass
      Engine: mysql
      EngineVersion: 8.0
      MasterUsername: !Ref MasterUsername
      MasterUserPassword: !Ref MasterUserPassword
      AllocatedStorage: 20
      StorageType: gp3
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref RDSSecurityGroup
      BackupRetentionPeriod: 7
      PreferredBackupWindow: "03:00-04:00"
      PreferredMaintenanceWindow: "mon:04:00-mon:05:00"
      MultiAZ: false
      PubliclyAccessible: false
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-RDS'

Outputs:
  DBEndpoint:
    Description: Database Endpoint
    Value: !GetAtt MyDB.Endpoint.Address
  
  DBPort:
    Description: Database Port
    Value: !GetAtt MyDB.Endpoint.Port
```

## Step 5: Auto Scaling Group

```
yaml
# asg.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Auto Scaling Group with Load Balancer

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
  
  MinSize:
    Type: Number
    Default: 2
  
  MaxSize:
    Type: Number
    Default: 4
  
  TargetCapacity:
    Type: Number
    Default: 2

Resources:
  # Launch Template
  WebServerLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub '${AWS::StackName}-LT'
      LaunchTemplateData:
        ImageId: ami-0c55b159cbfafe1f0
        InstanceType: !Ref InstanceType
        KeyName: my-key-pair
        SecurityGroupIds:
          - !Ref InstanceSecurityGroup
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd
            systemctl start httpd
            systemctl enable httpd

  # Security Group
  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Instance Security Group
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  # Application Load Balancer
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub '${AWS::StackName}-ALB'
      Scheme: internet-facing
      Type: application
      Subnets:
        - !ImportValue MyVPC-PublicSubnet1
        - !ImportValue MyVPC-PublicSubnet2
      SecurityGroups:
        - !Ref ALBSecurityGroup

  # ALB Security Group
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB Security Group
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  # Target Group
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub '${AWS::StackName}-TG'
      Port: 80
      Protocol: HTTP
      VpcId: !ImportValue MyVPC-VPCID
      HealthCheckPath: /
      HealthCheckIntervalSeconds: 30
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 5

  # ALB Listener
  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup

  # Auto Scaling Group
  WebServerASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: !Sub '${AWS::StackName}-ASG'
      LaunchTemplate:
        LaunchTemplateId: !Ref WebServerLaunchTemplate
        Version: !GetAtt WebServerLaunchTemplate.LatestVersionNumber
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      DesiredCapacity: !Ref TargetCapacity
      TargetGroupARNs:
        - !Ref TargetGroup
      VPCZoneIdentifier:
        - !ImportValue MyVPC-PublicSubnet1
        - !ImportValue MyVPC-PublicSubnet2

  # Scaling Policies
  ScaleUpPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref WebServerASG
      PolicyType: SimpleScaling
      AdjustmentType: ChangeInCapacity
      ScalingAdjustment: 1
      Cooldown: 300

  ScaleDownPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref WebServerASG
      PolicyType: SimpleScaling
      AdjustmentType: ChangeInCapacity
      ScalingAdjustment: -1
      Cooldown: 300

Outputs:
  ALBDNSName:
    Description: ALB DNS Name
    Value: !GetAtt ApplicationLoadBalancer.DNSName
```

## Step 6: Nested Stacks

### Root Stack

```
yaml
# root-stack.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Root Stack with Nested Stacks

Resources:
  # VPC Stack
  VPCStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/templates/vpc.yaml
      TimeoutInMinutes: 30
      Parameters:
        VpcCIDR: 10.0.0.0/16

  # Security Groups Stack
  SecurityGroupsStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: VPCStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/templates/security-groups.yaml
      TimeoutInMinutes: 30
      Parameters:
        VPCID: !GetAtt VPCStack.Outputs.VPCID

  # EC2 Stack
  EC2Stack:
    Type: AWS::CloudFormation::Stack
    DependsOn: VPCStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/templates/ec2.yaml
      TimeoutInMinutes: 30
      Parameters:
        VPCID: !GetAtt VPCStack.Outputs.VPCID
        SubnetID: !GetAtt VPCStack.Outputs.PublicSubnet1ID

Outputs:
  VPCID:
    Description: VPC ID from nested stack
    Value: !GetAtt VPCStack.Outputs.VPCID
```

## Step 7: CloudFormation Stack Sets

```
bash
# Create stack set
aws cloudformation create-stack-set \
    --stack-set-name cross-account-vpc \
    --template-body file://vpc.yaml \
    --description "VPC Stack Set"

# Add stacks to stack set (across accounts)
aws cloudformation create-stack-instances \
    --stack-set-name cross-account-vpc \
    --deployment-targets accounts='["123456789012","987654321098"]' \
    --regions '["us-east-1", "us-west-2"]' \
    --operation-preferences FailureToleranceCount=1,MaxConcurrentCount=2

# List stack instances
aws cloudformation list-stack-instances \
    --stack-set-name cross-account-vpc

# Update stack set
aws cloudformation update-stack-set \
    --stack-set-name cross-account-vpc \
    --template-body file://vpc-updated.yaml

# Delete stack instances
aws cloudformation delete-stack-instances \
    --stack-set-name cross-account-vpc \
    --accounts '["123456789012"]' \
    --regions '["us-west-2"]' \
    --retain-stacks
```

## Step 8: CloudFormation Macros

### Create Lambda Macro

```
bash
# Create Lambda function for macro
cat > macro-function.py <<EOF
import json

def handler(event, context):
    # Transform the template fragment
    fragment = event['fragment']
    
    # Example: Add tags to all resources
    if 'Resources' in fragment:
        for resource_id, resource in fragment['Resources'].items():
            if 'Tags' not in resource.get('Properties', {}):
                resource.setdefault('Properties', {}).setdefault('Tags', [])
            resource['Properties']['Tags'].append(
                {'Key': 'ManagedBy', 'Value': 'CloudFormation'}
            )
    
    return {
        'requestId': event['requestId'],
        'status': 'success',
        'fragment': fragment
    }
EOF

# Package Lambda function
zip macro-function.zip macro-function.py

# Create S3 bucket for Lambda code
aws s3 mb s3://cfn-macro-bucket

# Upload Lambda function
aws s3 cp macro-function.zip s3://cfn-macro-bucket/

# Create Lambda function
aws lambda create-function \
    --function-name CFNMacro \
    --runtime python3.9 \
    --role arn:aws:iam::123456789012:role/lambda-execution-role \
    --handler macro-function.handler \
    --code S3Bucket=cfn-macro-bucket,S3Key=macro-function.zip

# Create macro
aws cloudformation create-stack-set \
    --stack-set-name macro-stack \
    --template-body file://macro.yaml
```

### Macro Template

```
yaml
# macro.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CloudFormation Macro

Resources:
  CFNMacro:
    Type: AWS::CloudFormation::Macro
    Properties:
      Name: !Sub '${AWS::StackName}-AddTags'
      Description: Adds tags to all resources
      FunctionName: !GetAtt LambdaFunction.Arn

  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.9
      Handler: index.handler
      Code:
        S3Bucket: cfn-macro-bucket
        S3Key: macro-function.zip
      Role: !GetAtt LambdaExecutionRole.Arn

  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole

  LambdaExecutionPolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: lambda-execution
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Action:
              - logs:CreateLogGroup
              - logs:CreateLogStream
              - logs:PutLogEvents
            Resource: arn:aws:logs:*:*:*
      Roles:
        - !Ref LambdaExecutionRole
```

## Step 9: Drift Detection

```
bash
# Detect drift on a stack
aws cloudformation detect-stack-drift \
    --stack-name my-vpc-stack

# Describe stack drift detection status
aws cloudformation describe-stack-drift-detection-status \
    --stack-name my-vpc-stack

# Get stack resource drifts
aws cloudformation describe-stack-resource-drifts \
    --stack-name my-vpc-stack \
    --stack-resource-drift-status-filters MODIFIED DELETED

# List stacks with drift
aws cloudformation list-stacks \
    --stack-status-filter UPDATE_COMPLETE \
    --query 'StackSummaries[?DriftInformation.StackDriftStatus==`DRIFTED`]'

# Get template from stack
aws cloudformation get-template \
    --stack-name my-vpc-stack \
    --query 'TemplateBody'
```

## Step 10: CloudFormation Guard

### Install CFN-Guard

```
bash
# Install CFN-Guard
wget https://github.com/aws-cloudformation/cloudformation-guard/releases/download/3.0.0/cfn-guard-linux-x86_64.zip
unzip cfn-guard-linux-x86_64.zip
sudo mv cfn-guard /usr/local/bin/

# Verify installation
cfn-guard --version
```

### Create Guard Rules

```
yaml
# security-rules.guard
# Require encryption on S3 buckets
AWS::S3::Bucket == * {
    let encryption = Properties.BucketEncryption
    when %encryption != <empty> {
        %encryption.ServerSideEncryptionConfiguration[*] != <empty>
    }
}

# Require VPC flow logs
AWS::EC2::VPC == * {
    let flow_logs = Resources.*[ Type == "AWS::EC2::FlowLog" ]
    %flow_logs != <empty>
}

# Require tags on all resources
AWS::EC2::Instance == * {
    Properties.Tags != <empty>
}

# Require specific instance type
AWS::RDS::DBInstance == * {
    Properties.DBInstanceClass == /db\.[a-z]+\.micro/
}

# Deny public access
AWS::S3::Bucket == * {
    Properties.PublicAccessBlockConfiguration.BlockPublicAcls == true
    Properties.PublicAccessBlockConfiguration.BlockPublicPolicy == true
}
```

### Validate Template

```
bash
# Validate against rules
cfn-guard validate \
    --data s3-bucket.yaml \
    --rules security-rules.guard

# Check multiple files
cfn-guard validate \
    --data-dir ./templates \
    --rules security-rules.guard

# Generate test results
cfn-guard validate \
    --data s3-bucket.yaml \
    --rules security-rules.guard \
    --output-format json
```

## Step 11: CI/CD for CloudFormation

### GitHub Actions

```
yaml
# .github/workflows/cloudformation.yml
name: CloudFormation CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Install CFN-Nag
        run: |
          gem install cfn-nag
      
      - name: Validate CloudFormation templates
        run: |
          for template in templates/*.yaml; do
            aws cloudformation validate-template \
              --template-body file://$template
          done
      
      - name: Run CFN-Nag
        run: |
          for template in templates/*.yaml; do
            cfn-nag_scan \
              --input-path $template \
              --output-format txt
          done
      
      - name: Run CFN-Guard
        run: |
          cfn-guard validate \
            --data-dir templates \
            --rules rules/security.guard

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Create Change Set
        run: |
          aws cloudformation create-change-set \
            --stack-name my-stack \
            --template-body file://templates/main.yaml \
            --change-set-type UPDATE \
            --change-set-name ${{ github.sha }}
      
      - name: Describe Change Set
        run: |
          aws cloudformation describe-change-set \
            --stack-name my-stack \
            --change-set-name ${{ github.sha }}
      
      - name: Execute Change Set
        run: |
          aws cloudformation execute-change-set \
            --stack-name my-stack \
            --change-set-name ${{ github.sha }}
      
      - name: Wait for stack completion
        run: |
          aws cloudformation wait stack-update-complete \
            --stack-name my-stack
```

## Step 12: Complete Application Stack

### Full Stack Template

```
yaml
# full-stack.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Complete Application Stack

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod
  
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
  
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName

Conditions:
  IsProduction: !Equals [!Ref Environment, prod]

Resources:
  # VPC
  VPCStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${AWS::Region}-assets/${AWS::AccountId}/vpc.yaml'
      Parameters:
        VpcCIDR: !If [IsProduction, 10.0.0.0/16, 192.168.0.0/16]

  # RDS
  RDSStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: VPCStack
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${AWS::Region}-assets/${AWS::AccountId}/rds.yaml'
      Parameters:
        VPCID: !GetAtt VPCStack.Outputs.VPCID
        SubnetIDs: !Join [',', [!GetAtt VPCStack.Outputs.PrivateSubnet1ID, !GetAtt VPCStack.Outputs.PrivateSubnet2ID]]

  # ECS Cluster
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub 'app-${Environment}'
      ClusterSettings:
        - Name: containerInsights
          Value: enabled

  # Application Load Balancer
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub 'app-${Environment}-alb'
      Scheme: internet-facing
      Type: application
      Subnets: !Split [',', !Join [',', [!GetAtt VPCStack.Outputs.PublicSubnet1ID, !GetAtt VPCStack.Outputs.PublicSubnet2ID]]]
      SecurityGroups:
        - !Ref ALBSecurityGroup

  # ECS Service (defined in nested template or inline)
  ECSService:
    Type: AWS::ECS::Service
    DependsOn: ALBListener
    Properties:
      Cluster: !Ref ECSCluster
      ServiceName: !Sub 'app-${Environment}'
      TaskDefinition: !Ref TaskDefinition
      DesiredCount: !If [IsProduction, 3, 1]
      LaunchType: FARGATE
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: DISABLED
          Subnets: !Split [',', !Join [',', [!GetAtt VPCStack.Outputs.PrivateSubnet1ID, !GetAtt VPCStack.Outputs.PrivateSubnet2ID]]]
          SecurityGroups:
            - !Ref ECSSecurityGroup
      LoadBalancers:
        - ContainerName: app
          ContainerPort: 80
          TargetGroupArn: !Ref TargetGroup

  # Task Definition
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub 'app-${Environment}'
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      Cpu: 256
      Memory: 512
      ExecutionRoleArn: !Ref ECSTaskExecutionRole
      ContainerDefinitions:
        - Name: app
          Image: nginx:latest
          Essential: true
          PortMappings:
            - ContainerPort: 80

Outputs:
  ALBDNSName:
    Description: ALB DNS Name
    Value: !GetAtt ALB.DNSName
```

## Conclusion

This project demonstrates:
- Creating basic CloudFormation templates
- Working with VPC and networking resources
- Deploying EC2 instances with security groups
- Setting up RDS databases
- Creating Auto Scaling Groups with Load Balancers
- Implementing nested stacks
- Using CloudFormation Stack Sets
- Creating CloudFormation macros
- Implementing drift detection
- Using CloudFormation Guard for validation
- CI/CD pipelines for CloudFormation
- Building complete application stacks

## Additional Resources
- [CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)
- [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [AWS Quick Start Templates](https://aws.amazon.com/quickstart/)
- [cfn-nag](https://github.com/stelligent/cfn-nag)
- [cfn-guard](https://github.com/aws-cloudformation/cloudformation-guard)
