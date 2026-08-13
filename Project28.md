# AWS LAMBDA & SERVERLESS ARCHITECTURE

## Project Overview

This project demonstrates building serverless applications using AWS Lambda, API Gateway, and other serverless services. Serverless computing allows you to build and run applications without provisioning or managing servers.

### Objectives
1. Create and configure AWS Lambda functions
2. Build REST APIs with API Gateway
3. Implement serverless data processing
4. Use DynamoDB for serverless databases
5. Implement serverless authentication with Cognito
6. Set up serverless CI/CD pipelines
7. Monitor serverless applications

## Prerequisites
- AWS Account
- AWS CLI installed and configured
- Node.js or Python installed
- SAM CLI (optional but recommended)

## Step 1: Set Up Development Environment

### Install AWS SAM CLI

```
bash
# Install AWS SAM CLI on Linux
# Download SAM CLI
wget https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
unzip aws-sam-cli-linux-x86_64.zip
sudo ./install

# Verify installation
sam --version

# Or install using pip
pip install aws-sam-cli
sam --version

# Initialize SAM project
sam init --runtime python3.9 --name my-serverless-app
```

### Configure AWS CLI

```
bash
# Configure AWS credentials
aws configure
# Enter your Access Key ID
# Enter your Secret Access Key
# Enter region (e.g., us-east-1)
# Enter output format (json)

# Verify configuration
aws sts get-caller-identity
```

## Step 2: Create Your First Lambda Function

### Create Lambda Function Using Console

1. Navigate to AWS Lambda Console
2. Click "Create function"
3. Select "Author from scratch"
4. Configure:
   - Function name: `HelloWorldFunction`
   - Runtime: Python 3.9
   - Architecture: x86_64
   - Permissions: Create a new role with basic Lambda permissions
5. Click "Create function"

### Write Lambda Function Code

```
python
# lambda_function.py
import json

def lambda_handler(event, context):
    """AWS Lambda handler function"""
    
    # Get name from query parameter or body
    name = event.get('queryStringParameters', {}).get('name', 'World')
    
    # Return response
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({
            'message': f'Hello, {name}!',
            'input': event,
            'context': {
                'function_name': context.function_name,
                'memory_limit': context.memory_limit_in_mb,
                'request_id': context.aws_request_id,
                'log_stream': context.log_stream_name
            }
        })
    }

# Alternative handler for testing
def test_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Test successful!'
    }
```

### Create Lambda Function Using CLI

```
bash
# Create Lambda function using AWS CLI
aws lambda create-function \
    --function-name HelloWorldFunction \
    --runtime python3.9 \
    --role arn:aws:iam::123456789012:role/lambda-execution-role \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip \
    --timeout 10 \
    --memory-size 128

# Update function code
aws lambda update-function-code \
    --function-name HelloWorldFunction \
    --zip-file fileb://function.zip

# Invoke function
aws lambda invoke \
    --function-name HelloWorldFunction \
    --payload '{"queryStringParameters":{"name":"DevOps"}}' \
    response.json

# View logs
aws logs tail /aws/lambda/HelloWorldFunction --follow
```

## Step 3: Build REST API with API Gateway

### Create API Gateway

```
bash
# Create REST API
aws apigateway create-rest-api \
    --name MyServerlessAPI \
    --description "Serverless REST API" \
    --endpoint-configuration types=REGIONAL

# Get API ID
API_ID=$(aws apigateway get-rest-apis --query 'items[0].id' --output text)

# Get root resource ID
ROOT_ID=$(aws apigateway get-resources \
    --rest-api-id $API_ID \
    --query 'items[0].id' --output text)

# Create resource
aws apigateway create-resource \
    --rest-api-id $API_ID \
    --parent-id $ROOT_ID \
    --path-part hello

# Get resource ID
RESOURCE_ID=$(aws apigateway get-resources \
    --rest-api-id $API_ID \
    --query 'items[?pathPart==`hello`].id' --output text)

# Create method
aws apigateway put-method \
    --rest-api-id $API_ID \
    --resource-id $RESOURCE_ID \
    --http-method GET \
    --authorization-type NONE

# Create Lambda integration
aws apigateway put-integration \
    --rest-api-id $API_ID \
    --resource-id $RESOURCE_ID \
    --http-method GET \
    --type AWS_PROXY \
    --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789012:function:HelloWorldFunction/invocations \
    --integration-httpMethod POST

# Deploy API
aws apigateway create-deployment \
    --rest-api-id $API_ID \
    --stage-name dev

# Get API URL
echo "API URL: https://${API_ID}.execute-api.us-east-1.amazonaws.com/dev/hello"
```

### SAM Template for API

```
yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Serverless API with Lambda

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.9
      Timeout: 10
      MemorySize: 128
      Events:
        HelloApi:
          Type: Api
          Properties:
            Path: /hello
            Method: get

  HelloFunctionPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref HelloWorldFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub arn:aws:execute-api:us-east-1:${AWS::AccountId}:*/*/*/*

Outputs:
  ApiUrl:
    Description: API endpoint URL
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello"
```

```
python
# hello_world/app.py
import json

def lambda_handler(event, context):
    """Lambda handler function"""
    
    # Get query parameters
    params = event.get('queryStringParameters') or {}
    name = params.get('name', 'World')
    
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Headers': 'Content-Type'
        },
        'body': json.dumps({
            'message': f'Hello, {name}!',
            'requestId': context.aws_request_id
        })
    }
```

### Deploy Using SAM

```
bash
# Build SAM application
sam build

# Deploy SAM application
sam deploy --guided

# Or deploy with specific options
sam deploy \
    --stack-name my-serverless-api \
    --s3-bucket my-deployment-bucket \
    --region us-east-1 \
    --capabilities CAPABILITY_IAM

# Test API
curl https://xxxx.execute-api.us-east-1.amazonaws.com/dev/hello?name=DevOps
```

## Step 4: Serverless Data Processing

### S3 Trigger Lambda

```
python
# s3_processor.py
import boto3
import json
import os

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    """Process S3 object created events"""
    
    # Get bucket name and key from event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    print(f"Processing file: {key} from bucket: {bucket}")
    
    # Get object metadata
    response = s3_client.head_object(Bucket=bucket, Key=key)
    size = response['ContentLength']
    content_type = response.get('ContentType', 'unknown')
    
    print(f"File size: {size} bytes")
    print(f"Content type: {content_type}")
    
    # Process based on file type
    if key.endswith('.csv'):
        process_csv(bucket, key)
    elif key.endswith('.json'):
        process_json(bucket, key)
    elif key.endswith('.txt'):
        process_text(bucket, key)
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'File processed successfully',
            'bucket': bucket,
            'key': key
        })
    }

def process_csv(bucket, key):
    """Process CSV files"""
    print(f"Processing CSV file: {key}")
    # Add your CSV processing logic here

def process_json(bucket, key):
    """Process JSON files"""
    print(f"Processing JSON file: {key}")
    # Add your JSON processing logic here

def process_text(bucket, key):
    """Process text files"""
    print(f"Processing text file: {key}")
    # Add your text processing logic here
```

### SAM Template for S3 Trigger

```
yaml
# template.yaml
Resources:
  S3ProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: s3_processor/
      Handler: app.lambda_handler
      Runtime: python3.9
      Timeout: 300
      MemorySize: 1024
      Events:
        S3Trigger:
          Type: S3
          Properties:
            Bucket: !Ref InputBucket
            Event: s3:ObjectCreated:*
            Filter:
              S3Key:
                Rules:
                  - Name: suffix
                    Value: .csv

  InputBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub input-bucket-${AWS::AccountId}
      CorsConfiguration:
        CorsRules:
          - AllowedHeaders: ['*']
            AllowedMethods: [GET, PUT, POST]
            AllowedOrigins: ['*']
            MaxAge: 3000
```

### DynamoDB Stream Processing

```
python
# dynamodb_processor.py
import json
from datetime import datetime

def lambda_handler(event, context):
    """Process DynamoDB stream events"""
    
    print(f"Received {len(event['Records'])} records")
    
    for record in event['Records']:
        # Handle insert events
        if record['eventName'] == 'INSERT':
            new_image = record['dynamodb']['NewImage']
            print(f"New item: {json.dumps(new_image)}")
            process_new_item(new_image)
        
        # Handle modify events
        elif record['eventName'] == 'MODIFY':
            old_image = record['dynamodb']['OldImage']
            new_image = record['dynamodb']['NewImage']
            print(f"Modified item: {json.dumps(new_image)}")
            process_modified_item(old_image, new_image)
        
        # Handle remove events
        elif record['eventName'] == 'REMOVE':
            old_image = record['dynamodb']['OldImage']
            print(f"Removed item: {json.dumps(old_image)}")
            process_removed_item(old_image)
    
    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Records processed successfully'})
    }

def process_new_item(item):
    """Process new DynamoDB item"""
    # Add your processing logic
    pass

def process_modified_item(old_item, new_item):
    """Process modified DynamoDB item"""
    # Add your processing logic
    pass

def process_removed_item(item):
    """Process removed DynamoDB item"""
    # Add your processing logic
    pass
```

## Step 5: Serverless Database with DynamoDB

### Create DynamoDB Table

```
bash
# Create DynamoDB table
aws dynamodb create-table \
    --table-name Products \
    --attribute-definitions \
        AttributeName=productId,AttributeType=S \
        AttributeName=category,AttributeType=S \
    --key-schema \
        AttributeName=productId,KeyType=HASH \
    --global-secondary-indexes \
        '[{"IndexName":"CategoryIndex","KeySchema":[{"AttributeName":"category","KeyType":"HASH"}],"Projection":{"ProjectionType":"ALL"}}]' \
    --billing-mode PAY_PER_REQUEST \
    --region us-east-1

# Enable TTL for automatic cleanup
aws dynamodb update-time-to-live \
    --table-name Products \
    --time-to-live-specification Enabled=true,AttributeName=expiryDate
```

### Lambda with DynamoDB

```
python
# dynamodb_operations.py
import boto3
import json
import os
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ.get('TABLE_NAME', 'Products'))

def lambda_handler(event, context):
    """Handle CRUD operations for DynamoDB"""
    
    http_method = event['httpMethod']
    path = event['path']
    
    if http_method == 'GET':
        if '/products' in path:
            if 'productId' in event.get('pathParameters', {}):
                return get_product(event['pathParameters']['productId'])
            else:
                return list_products(event.get('queryStringParameters', {}))
    
    elif http_method == 'POST':
        return create_product(json.loads(event['body']))
    
    elif http_method == 'PUT':
        return update_product(
            event['pathParameters']['productId'],
            json.loads(event['body'])
        )
    
    elif http_method == 'DELETE':
        return delete_product(event['pathParameters']['productId'])
    
    return {
        'statusCode': 400,
        'body': json.dumps({'error': 'Invalid request'})
    }

def get_product(product_id):
    """Get a single product"""
    response = table.get_item(Key={'productId': product_id})
    item = response.get('Item')
    
    if item:
        return {
            'statusCode': 200,
            'body': json.dumps(item)
        }
    return {
        'statusCode': 404,
        'body': json.dumps({'error': 'Product not found'})
    }

def list_products(params):
    """List products with optional filtering"""
    if params and 'category' in params:
        response = table.query(
            IndexName='CategoryIndex',
            KeyConditionExpression=Key('category').eq(params['category'])
        )
    else:
        response = table.scan()
    
    return {
        'statusCode': 200,
        'body': json.dumps(response.get('Items', []))
    }

def create_product(item):
    """Create a new product"""
    table.put_item(Item=item)
    return {
        'statusCode': 201,
        'body': json.dumps(item)
    }

def update_product(product_id, updates):
    """Update existing product"""
    update_expr = 'SET ' + ', '.join([f'{k} = :{k}' for k in updates.keys()])
    expr_values = {f':{k}': v for k, v in updates.items()}
    
    response = table.update_item(
        Key={'productId': product_id},
        UpdateExpression=update_expr,
        ExpressionAttributeValues=expr_values,
        ReturnValues='ALL_NEW'
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps(response.get('Attributes'))
    }

def delete_product(product_id):
    """Delete a product"""
    table.delete_item(Key={'productId': product_id})
    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Product deleted'})
    }
```

## Step 6: Authentication with Cognito

### Create Cognito User Pool

```
bash
# Create Cognito User Pool
aws cognito-idp create-user-pool \
    --pool-name MyUserPool \
    --username-attributes email \
    --auto-verified-attributes email \
    --policies '{"PasswordPolicy":{"MinimumLength":8,"RequireUppercase":true,"RequireLowercase":true,"RequireNumbers":true,"RequireSymbols":true}}' \
    --verification-message-template EmailSubject="Verify your email" EmailMessage="Your verification code is {####}"

# Create App Client
aws cognito-idp create-user-pool-client \
    --user-pool-id <pool-id> \
    --client-name web-app \
    --no-generate-secret \
    --allowed-o-auth-flows code \
    --allowed-o-auth-scopes email openid \
    --callback-urls https://example.com/callback \
    --logout-urls https://example.com/logout

# Create Identity Pool
aws cognito-identity create-identity-pool \
    --identity-pool-name MyIdentityPool \
    --allow-unauthenticated-identities \
    --cognito-identity-providers ProviderName=cognito-idp.us-east-1.amazonaws.com/<pool-id>,ClientId=<app-client-id>
```

### Lambda Authorizer with Cognito

```
python
# cognito_authorizer.py
import json
import boto3
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    """Validate JWT token from Cognito"""
    
    # Get token from authorization header
    token = event.get('authorizationToken')
    
    if not token:
        return generate_policy('deny', event['methodArn'])
    
    # Validate token (simplified - use cognito-jwt-validator in production)
    try:
        # In production, use amazon-cognito-identity-js or python-jose
        # to validate the JWT token
        
        # For now, assume valid if token exists
        # Replace with actual validation logic
        validate_token(token)
        
        return generate_policy('allow', event['methodArn'], {
            'sub': 'user-id',
            'email': 'user@example.com'
        })
    
    except Exception as e:
        print(f"Token validation error: {e}")
        return generate_policy('deny', event['methodArn'])

def validate_token(token):
    """Validate JWT token"""
    # Add your token validation logic here
    # - Verify signature with Cognito public keys
    # - Check token expiration
    # - Validate claims
    pass

def generate_policy(effect, resource, context=None):
    """Generate IAM policy"""
    policy = {
        'principalId': context.get('sub', 'anonymous') if context else 'anonymous',
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Action': 'execute-api:Invoke',
                    'Effect': effect,
                    'Resource': resource
                }
            ]
        }
    }
    
    if context:
        policy['context'] = context
    
    return policy
```

### SAM Template with Cognito Authorizer

```
yaml
# template.yaml
Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: CognitoAuthorizer
        Authorizers:
          CognitoAuthorizer:
            UserPoolArn: !GetAtt UserPool.Arn
            Identity:
              Header: Authorization
    
  UserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: MyUserPool
      UsernameAttributes:
        - email
      AutoVerifiedAttributes:
        - email
```

## Step 7: Serverless CI/CD Pipeline

### GitHub Actions Workflow

```
yaml
# .github/workflows/serverless.yml
name: Serverless Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          pip install pytest pytest-cov boto3 moto
      
      - name: Run tests
        run: |
          pytest tests/ --cov=. --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  deploy:
    needs: test
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
      
      - name: Setup SAM
        uses: aws-actions/setup-sam@v2
      
      - name: Build SAM
        run: sam build
      
      - name: Deploy SAM
        run: |
          sam deploy \
            --stack-name ${{ secrets.STACK_NAME }} \
            --s3-bucket ${{ secrets.S3_BUCKET }} \
            --capabilities CAPABILITY_IAM \
            --no-confirm-changeset
```

### Jenkins Pipeline

```
groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        AWS_REGION = 'us-east-1'
        S3_BUCKET = 'my-serverless-deploy-bucket'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install SAM CLI') {
            steps {
                sh 'pip install aws-sam-cli'
            }
        }
        
        stage('Build') {
            steps {
                sh 'sam build'
            }
        }
        
        stage('Test') {
            steps {
                sh 'pytest tests/ -v'
            }
        }
        
        stage('Package') {
            steps {
                sh """
                    sam package \
                    --s3-bucket ${S3_BUCKET} \
                    --output-template-file packaged.yaml
                """
            }
        }
        
        stage('Deploy') {
            steps {
                withCredentials([string(credentialsId: 'aws-creds', variable: 'AWS_ACCESS_KEY_ID')]) {
                    sh """
                        sam deploy \
                        --stack-name my-serverless-app \
                        --template-file packaged.yaml \
                        --capabilities CAPABILITY_IAM \
                        --parameter-overrides ParameterKey=Environment,ParameterValue=prod
                    """
                }
            }
        }
        
        stage('Verify') {
            steps {
                sh """
                    API_URL=\$(aws cloudformation describe-stacks \
                    --stack-name my-serverless-app \
                    --query 'Stacks[0].Outputs[?OutputKey==`ApiUrl`].OutputValue' \
                    --output text)
                    
                    curl -s \$API_URL/hello?name=Test
                """
            }
        }
    }
}
```

## Step 8: Monitoring and Observability

### CloudWatch Logs and Metrics

```
python
# monitored_function.py
import json
import logging
import time
from datetime import datetime

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Custom metrics
def put_metric(name, value, unit='Count'):
    """Send custom CloudWatch metric"""
    import boto3
    cloudwatch = boto3.client('cloudwatch')
    
    cloudwatch.put_metric_data(
        Namespace='ServerlessApp',
        MetricData=[
            {
                'MetricName': name,
                'Value': value,
                'Unit': unit,
                'Timestamp': datetime.utcnow()
            }
        ]
    )

def lambda_handler(event, context):
    """Lambda handler with monitoring"""
    
    start_time = time.time()
    
    try:
        logger.info(f"Request received: {json.dumps(event)}")
        
        # Your business logic here
        result = process_request(event)
        
        # Record success metrics
        put_metric('Requests', 1)
        put_metric('Latency', time.time() - start_time, 'Seconds')
        
        logger.info(f"Request processed: {result}")
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
    
    except Exception as e:
        # Record error metrics
        put_metric('Errors', 1)
        logger.error(f"Error processing request: {str(e)}")
        
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }

def process_request(event):
    """Process the request"""
    # Add your processing logic
    return {'status': 'success'}
```

### X-Ray Tracing

```
python
# xray_function.py
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask_lambda import patch_all
import json

# Patch all libraries
patch_all()

def lambda_handler(event, context):
    """Lambda handler with X-Ray tracing"""
    
    # Start subsegment
    with xray_recorder.capture('process_request') as subsegment:
        subsegment.put_annotation('request_id', context.aws_request_id)
        subsegment.put_metadata('event', event)
        
        # Your business logic
        result = process_data(event)
        
        subsegment.put_annotation('result', result)
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }

def process_data(event):
    """Process data with X-Ray"""
    with xray_recorder.capture('data_processing'):
        # Add your processing logic
        return {'processed': True}
```

### SAM Template with Monitoring

```
yaml
# template.yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: app.lambda_handler
      Runtime: python3.9
      Tracing: Active
      Environment:
        Variables:
          LOG_LEVEL: INFO
      Policies:
        - AWSXrayWriteOnlyAccess
        - CloudWatchLogsFullAccess

  # CloudWatch Dashboard
  Dashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: ServerlessAppDashboard
      DashboardBody:
        Fn::Sub: '{"widgets":[{"type":"Metric","properties":{"title":"Invocations","metrics":[["ServerlessApp","Errors"]],["."]]}}]}'
```

## Step 9: Complete Serverless Application

### Project Structure

```
my-serverless-app/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── tests/
│   ├── __init__.py
│   ├── test_function.py
│   └── conftest.py
├── hello_world/
│   ├── __init__.py
│   ├── app.py
│   └── requirements.txt
├── products/
│   ├── __init__.py
│   ├── app.py
│   └── requirements.txt
├── template.yaml
├── samconfig.toml
├── README.md
└── requirements.txt
```

### Complete SAM Template

```
yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Complete Serverless Application

Globals:
  Function:
    Timeout: 30
    MemorySize: 256
    Environment:
      Variables:
        LOG_LEVEL: INFO
        TABLE_NAME: !Ref ProductsTable

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod

Resources:
  # DynamoDB Table
  ProductsTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      TableName: !Sub products-${Environment}
      PrimaryKey:
        Name: productId
        Type: String

  # Hello World Function
  HelloFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        Api:
          Type: Api
          Properties:
            Path: /hello
            Method: GET

  # Products API Function
  ProductsFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: products/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        Api:
          Type: Api
          Properties:
            Path: /products
            Method: ANY
        ApiRoot:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: ANY

  # S3 Upload Function
  UploadFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: upload/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        S3Upload:
          Type: S3
          Properties:
            Bucket: !Ref UploadBucket
            Event: s3:ObjectCreated:*

  UploadBucket:
    Type: AWS::S3::Bucket

Outputs:
  ApiUrl:
    Description: API endpoint URL
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/${Stage}"
```

## Conclusion

This project demonstrates:
- Setting up development environment with SAM CLI
- Creating AWS Lambda functions
- Building REST APIs with API Gateway
- Serverless data processing with S3 triggers
- DynamoDB integration for serverless databases
- Cognito authentication
- CI/CD pipelines for serverless applications
- Monitoring with CloudWatch and X-Ray
- Building complete serverless applications

## Additional Resources
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [API Gateway Documentation](https://docs.aws.amazon.com/apigateway/)
- [DynamoDB Documentation](https://docs.aws.amazon.com/dynamodb/)
- [Cognito Documentation](https://docs.aws.amazon.com/cognito/)
