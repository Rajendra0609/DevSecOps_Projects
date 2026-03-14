# AWS ECS & EKS FARGATE CONTAINER ORCHESTRATION

## Project Overview

This project demonstrates deploying and managing containerized applications using AWS Elastic Container Service (ECS) and Elastic Kubernetes Service (EKS) with Fargate launch type. Both services enable serverless containers, removing the need to manage underlying infrastructure.

### Objectives
1. Understand ECS architecture and task definitions
2. Deploy containers with ECS Fargate
3. Set up EKS cluster with Fargate
4. Implement service mesh and ingress
5. Configure auto-scaling
6. Set up CI/CD for containers
7. Implement monitoring and logging

## Prerequisites
- AWS Account
- AWS CLI installed and configured
- Docker installed
- kubectl installed
- eksctl installed

## Step 1: Set Up ECS with Fargate

### Create ECS Cluster

```
bash
# Create ECS cluster with Fargate
aws ecs create-cluster \
    --cluster-name my-fargate-cluster \
    --settings "name=containerInsights,value=enabled"

# List clusters
aws ecs list-clusters

# Describe cluster
aws ecs describe-cluster --cluster my-fargate-cluster
```

### Create Task Definition

```
bash
# Create task definition JSON
cat > nginx-task.json <<EOF
{
    "family": "nginx-web",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "nginx",
            "image": "nginx:latest",
            "essential": true,
            "portMappings": [
                {
                    "containerPort": 80,
                    "protocol": "tcp"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/nginx-web",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            },
            "environment": [
                {
                    "name": "NGINX_PORT",
                    "value": "80"
                }
            ]
        }
    ]
}
EOF

# Register task definition
aws ecs register-task-definition --cli-input-json file://nginx-task.json

# List task definitions
aws ecs list-task-definitions
```

### Create Service

```
bash
# Create security group for ECS service
aws ec2 create-security-group \
    --group-name ecs-service-sg \
    --description "Security group for ECS service"

# Get VPC and subnet IDs
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query 'Vpcs[0].VpcId' --output text)
SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[0:2].SubnetId' --output text | tr '\t' ',')

# Create ECS service
aws ecs create-service \
    --cluster my-fargate-cluster \
    --service-name nginx-service \
    --task-definition nginx-web:1 \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[sg-xxxxx],assignPublicIp=ENABLED}" \
    --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/nginx-tg/xxxxx,containerName=nginx,containerPort=80" \
    --deployment-configuration "minimumHealthyPercent=50,maximumPercent=200"

# Update service
aws ecs update-service \
    --cluster my-fargate-cluster \
    --service nginx-service \
    --desired-count 3

# List services
aws ecs list-services --cluster my-fargate-cluster
```

### Create Application Load Balancer

```
bash
# Create target group
aws elbv2 create-target-group \
    --name nginx-tg \
    --protocol HTTP \
    --port 80 \
    --vpc-id vpc-xxxxx \
    --target-type ip \
    --health-check-path /

# Create ALB
aws elbv2 create-load-balancer \
    --name nginx-alb \
    --scheme internet-facing \
    --type application \
    --subnets subnet-xxxxx subnet-yyyyy

# Create listener
aws elbv2 create-listener \
    --load-balancer-arn alb-arn \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=tg-arn

# Register targets
aws elbv2 register-targets \
    --target-group-arn tg-arn \
    --targets Id=10.0.1.100,Port=80 Id=10.0.2.100,Port=80
```

## Step 2: ECS with Docker Compose

### Install ECS CLI

```
bash
# Download ECS CLI
sudo curl -Lo /usr/local/bin/ecs-cli https://amazon-ecs-cli.s3.amazonaws.com/ecs-cli-linux-amd64-latest

# Make executable
sudo chmod +x /usr/local/bin/ecs-cli

# Verify installation
ecs-cli --version
```

### Create Docker Compose File

```
yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
    logging:
      driver: awslogs
      options:
        awslogs-group: ecs-compose-web
        awslogs-region: us-east-1
        awslogs-stream-prefix: web
    environment:
      - ENVIRONMENT=production

  api:
    image: myapi:latest
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db.example.com
      - DB_PORT=5432
    logging:
      driver: awslogs
      options:
        awslogs-group: ecs-compose-api
        awslogs-region: us-east-1
        awslogs-stream-prefix: api

  db:
    image: postgres:14
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=myapp
    volumes:
      - db-data:/var/lib/postgresql/data
    logging:
      driver: awslogs
      options:
        awslogs-group: ecs-compose-db
        awslogs-region: us-east-1
        awslogs-stream-prefix: db

volumes:
  db-data:
```

### ECS Compose Up

```
bash
# Configure ECS CLI
ecs-cli configure \
    --cluster my-fargate-cluster \
    --default-launch-type FARGATE \
    --region us-east-1

# Create ECS cluster
ecs-cli up --capability-iam

# Deploy compose file
ecs-cli compose up

# Scale service
ecs-cli compose scale 3

# View logs
ecs-cli logs

# Stop services
ecs-cli compose down
```

## Step 3: Set Up EKS with Fargate

### Install Required Tools

```
bash
# Install kubectl
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.27.1/2023-05-11/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/

# Verify installations
kubectl version --client
eksctl version
```

### Create EKS Cluster with Fargate

```
bash
# Create EKS cluster with Fargate
eksctl create cluster \
    --name my-fargate-cluster \
    --region us-east-1 \
    --version 1.27 \
    --fargate

# Or create with custom configuration
eksctl create cluster \
    --name production-cluster \
    --region us-east-1 \
    --version 1.27 \
    --nodegroup-name managed-nodes \
    --node-type t3.medium \
    --nodes 3 \
    --nodes-min 1 \
    --nodes-max 5 \
    --with-oidc \
    --alb-ingress-controller

# Get cluster info
kubectl cluster-info

# List nodes
kubectl get nodes
```

### Create Fargate Profile

```
bash
# Create Fargate profile
eksctl create fargateprofile \
    --cluster my-fargate-cluster \
    --region us-east-1 \
    --name default-fargate-profile \
    --namespace default \
    --labels environment=production

# Create profile for specific namespace
eksctl create fargateprofile \
    --cluster my-fargate-cluster \
    --region us-east-1 \
    --name game-2048 \
    --namespace game-2048
```

### Deploy Application to EKS

```
bash
# Create namespace
kubectl create namespace myapp

# Create deployment
cat > deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
EOF

kubectl apply -f deployment.yaml

# Create service
cat > service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f service.yaml

# Check deployment status
kubectl get pods -n myapp
kubectl get svc -n myapp
```

## Step 4: Implement Ingress with AWS ALB

### Install AWS Load Balancer Controller

```
bash
# Create IAM policy for ALB controller
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

# Create service account
kubectl create serviceaccount aws-load-balancer-controller -n kube-system

# Create IAM role
eksctl create iamserviceaccount \
    --cluster my-fargate-cluster \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --attach-policy-arn arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve

# Install ALB controller using Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller \
    eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=my-fargate-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller
```

### Create Ingress Resource

```
yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/health-check-path: /healthz
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80

kubectl apply -f ingress.yaml
```

## Step 5: Auto-Scaling Configuration

### Configure HPA for ECS

```
bash
# Create scaling policy JSON
cat > scaling-policy.json <<EOF
{
    "policyName": "myapp-scaling-policy",
    "serviceName": "nginx-service",
    "clusterName": "my-fargate-cluster",
    "stepScalingPolicyConfiguration": {
        "adjustmentType": "ChangeInCapacity",
        "stepAdjustments": [
            {
                "metricIntervalLowerBound": 0,
                "scalingAdjustment": 1
            }
        ],
        "cooldown": 60,
        "metricAggregationType": "Average"
    }
}
EOF

# Register the scaling target first
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/my-fargate-cluster/nginx-service \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 1 \
    --max-capacity 5

# Apply the scaling policy
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/my-fargate-cluster/nginx-service \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name myapp-scaling-policy \
    --policy-type StepScaling \
    --step-scaling-policy-configuration file://scaling-policy.json
```

### Configure HPA for EKS

```
bash
# Create HPA
cat > hpa.yaml <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
EOF

kubectl apply -f hpa.yaml

# Check HPA status
kubectl get hpa -n myapp
```

## Step 6: CI/CD for Containers

### GitHub Actions for ECS

```
yaml
# .github/workflows/deploy-ecs.yml
name: Deploy to ECS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build and push image
        env:
          ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/myapp:$IMAGE_TAG .
          docker push $ECR_REGISTRY/myapp:$IMAGE_TAG
      
      - name: Update task definition
        id: task-def
        uses: aws-actions/amazon-ecs-update-task-definition@v1
        with:
          task-definition: task-definition.json
          container-image: ${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
      
      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: myapp-service
          cluster: my-fargate-cluster
```

### GitHub Actions for EKS

```
yaml
# .github/workflows/deploy-eks.yml
name: Deploy to EKS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name my-fargate-cluster --region us-east-1
      
      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f k8s/
          kubectl rollout status deployment/myapp -n myapp
```

## Step 7: Monitoring and Logging

### CloudWatch Container Insights

```
bash
# Enable Container Insights for ECS
aws ecs put-cluster-settings \
    --cluster my-fargate-cluster \
    --settings name=containerInsights,value=enabled

# For EKS with CloudWatch
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment/deploymentbits.yaml
```

### Create CloudWatch Dashboard

```
bash
# Create dashboard for ECS
aws cloudwatch put-dashboard \
    --dashboard-name ECS-Fargate-Dashboard \
    --dashboard-body '{
        "widgets": [
            {
                "type": "metric",
                "properties": {
                    "title": "CPU Utilization",
                    "metrics": [
                        ["ECS/ContainerInsights", "CpuUtilized", "ClusterName", "my-fargate-cluster", "ServiceName", "nginx-service"],
                        [".", "CpuReserved", ".", ".", ".", "."]
                    ],
                    "period": 300,
                    "stat": "Average"
                }
            },
            {
                "type": "metric",
                "properties": {
                    "title": "Memory Utilization",
                    "metrics": [
                        ["ECS/ContainerInsights", "MemoryUtilized", "ClusterName", "my-fargate-cluster", "ServiceName", "nginx-service"],
                        [".", "MemoryReserved", ".", ".", ".", "."]
                    ],
                    "period": 300,
                    "stat": "Average"
                }
            }
        ]
    }'
```

### Prometheus and Grafana on EKS

```
bash
# Install Prometheus and Grafana using Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus \
    --namespace monitoring \
    --create-namespace

# Install Grafana
helm install grafana prometheus-community/grafana \
    --namespace monitoring \
    --set persistence.enabled=true \
    --set adminPassword=admin

# Port forward Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# Access Grafana at http://localhost:3000
```

## Step 8: Service Mesh with AWS App Mesh

```
bash
# Install App Mesh controller
helm repo add appmesh https://aws.github.io/appmesh-charts
helm install appmesh-controller appmesh/appmesh-controller \
    --namespace appmesh-system \
    --create-namespace

# Create mesh
cat > mesh.yaml <<EOF
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: myapp-mesh
spec:
  namespaceSelector:
    matchLabels:
      mesh: myapp-mesh
EOF

kubectl apply -f mesh.yaml

# Create virtual nodes
cat > virtual-node.yaml <<EOF
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: myapp-vn
  namespace: myapp
spec:
  meshRef:
    name: myapp-mesh
  listeners:
    - portMapping:
        port: 80
        protocol: http
  serviceDiscovery:
    dns:
      hostname: myapp-service.myapp.svc.cluster.local
EOF
```

## Step 9: Complete Deployment Example

### Complete ECS Stack

```
bash
#!/bin/bash
# deploy-ecs.sh

# Variables
CLUSTER_NAME="my-fargate-cluster"
SERVICE_NAME="myapp-service"
TASK_FAMILY="myapp"
ECR_REPO="123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp"
IMAGE_TAG="${1:-latest}"

# Create ECR repository
aws ecr create-repository --repository-name myapp

# Build and push Docker image
docker build -t myapp:latest .
docker tag myapp:latest ${ECR_REPO}:${IMAGE_TAG}
docker tag myapp:latest ${ECR_REPO}:latest

aws ecr get-login-password | docker login --username AWS --password-stdin ${ECR_REPO}
docker push ${ECR_REPO}:${IMAGE_TAG}
docker push ${ECR_REPO}:latest

# Create task definition
cat > task-definition.json <<EOF
{
    "family": "${TASK_FAMILY}",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512",
    "executionRoleArn": "arn:aws:iam::${AWS_ACCOUNT}:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "myapp",
            "image": "${ECR_REPO}:${IMAGE_TAG}",
            "essential": true,
            "portMappings": [
                {
                    "containerPort": 3000,
                    "protocol": "tcp"
                }
            ],
            "environment": [
                {
                    "name": "NODE_ENV",
                    "value": "production"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/${SERVICE_NAME}",
                    "awslogs-region": "${AWS_REGION}",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ]
}
EOF

# Register task definition
TASK_DEF_ARN=$(aws ecs register-task-definition \
    --cli-input-json file://task-definition.json \
    --query 'taskDefinition.taskDefinitionArn' \
    --output text)

# Create service
aws ecs create-service \
    --cluster ${CLUSTER_NAME} \
    --service-name ${SERVICE_NAME} \
    --task-definition ${TASK_DEF_ARN} \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=subnet-xxxxx,subnet-yyyyy,securityGroups=sg-zzzzz,assignPublicIp=ENABLED}"

echo "Deployment complete!"
```

### Complete EKS Deployment

```
bash
#!/bin/bash
# deploy-eks.sh

CLUSTER_NAME="my-fargate-cluster"
NAMESPACE="myapp"
IMAGE="123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest"

# Create namespace
kubectl create namespace ${NAMESPACE}

# Create deployment
cat > myapp-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: ${NAMESPACE}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: ${IMAGE}
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

# Create service
cat > myapp-service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: ${NAMESPACE}
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer
EOF

# Apply configurations
kubectl apply -f myapp-deployment.yaml
kubectl apply -f myapp-service.yaml

# Wait for deployment
kubectl rollout status deployment/myapp -n ${NAMESPACE}

# Show service
kubectl get svc -n ${NAMESPACE}

echo "Deployment complete!"
```

## Conclusion

This project demonstrates:
- Creating ECS clusters with Fargate
- Writing task definitions and container configurations
- Deploying services with load balancers
- Using ECS CLI with Docker Compose
- Setting up EKS clusters with Fargate
- Deploying applications to EKS
- Implementing ingress with AWS Load Balancer Controller
- Configuring auto-scaling (HPA)
- CI/CD pipelines for ECS and EKS
- Monitoring with CloudWatch Container Insights
- Setting up Prometheus and Grafana
- Service mesh with AWS App Mesh

## Additional Resources
- [ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [EKS Documentation](https://docs.aws.amazon.com/eks/)
- [AWS Fargate Documentation](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html)
- [eksctl Documentation](https://eksctl.io/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
