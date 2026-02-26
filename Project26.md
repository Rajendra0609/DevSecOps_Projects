# SECRETS MANAGEMENT WITH HASHICORP VAULT

## Project Overview

This project demonstrates using HashiCorp Vault for secrets management, which is a critical component of DevSecOps practices. Vault provides secure secrets storage, encryption as a service, and identity-based access to secrets.

### Objectives
1. Install and configure HashiCorp Vault
2. Implement various secrets engines (KV, Database, AWS)
3. Set up authentication methods (Kubernetes, AppRole)
4. Implement dynamic secrets
5. Create policies and ACLs
6. Integrate Vault with applications
7. Use Vault in CI/CD pipelines

## Prerequisites
- Kubernetes cluster (Minikube, EKS, or Kind)
- Docker installed
- kubectl configured
- AWS Account (for AWS secrets engine)

## Step 1: Install HashiCorp Vault

### Install Vault on Linux

```
bash
# Download Vault
wget https://releases.hashicorp.com/vault/1.13.0/vault_1.13.0_linux_amd64.zip

# Unzip Vault
unzip vault_1.13.0_linux_amd64.zip

# Move to bin directory
sudo mv vault /usr/local/bin/

# Verify installation
vault version
```

### Install Vault with Docker

```
bash
# Pull Vault Docker image
docker pull hashicorp/vault:1.13.0

# Start Vault in dev mode (for development only)
docker run -d --name vault \
  --cap-add IPC_LOCK \
  -p 8200:8200 \
  -e VAULT_ADDR=http://localhost:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=myroot \
  hashicorp/vault:1.13.0

# Check Vault is running
docker exec vault vault status

# Initialize Vault
export VAULT_ADDR='http://localhost:8200'
export VAULT_TOKEN='myroot'
vault status
```

### Install Vault on Kubernetes

```
bash
# Add HashiCorp Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Create namespace
kubectl create namespace vault

# Install Vault in HA mode
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "ui.enabled=true" \
  --set "injector.enabled=true"

# Check Vault status
kubectl get pods -n vault

# Initialize Vault (for dev mode)
kubectl exec -n vault vault-0 -- vault operator init

# Unseal Vault (for production mode)
kubectl exec -n vault vault-0 -- vault operator unseal
```

## Step 2: Configure Vault

### Initialize Vault

```
bash
# Set Vault address
export VAULT_ADDR='http://localhost:8200'

# Initialize Vault
vault operator init -key-shares=5 -key-threshold=3

# Example output:
# Unseal Key 1: xxxxxxxx
# Unseal Key 2: xxxxxxxx
# Unseal Key 3: xxxxxxxx
# Initial Root Token: xxxxxxxx

# Unseal Vault (run 3 times with different keys)
vault operator unseal

# Login with root token
vault login

# Check Vault status
vault status
```

### Enable Secrets Engines

```
bash
# Enable KV v2 secrets engine
vault secrets enable -path=secret kv-v2

# Enable AWS secrets engine
vault secrets enable -path=aws aws

# Enable Database secrets engine
vault secrets enable -path=database database

# Enable PKI secrets engine
vault secrets enable -path=pki pki

# List enabled secrets engines
vault secrets list
```

## Step 3: Work with KV Secrets Engine

### Store Static Secrets

```
bash
# Store a secret
vault kv put secret/myapp config.username="admin" config.password="secretpassword"

# Read a secret
vault kv get secret/myapp

# Read specific field
vault kv get -field=config.password secret/myapp

# List secrets
vault kv list secret/

# Delete a secret
vault kv delete secret/myapp

# Update existing secret
vault kv patch secret/myapp config.newfield="newvalue"

# Version history
vault kv versions secret/myapp
```

### Create Versioned Secrets

```
bash
# Create secret with multiple versions
vault kv put secret/database username="dbadmin" password="v1password"
vault kv put secret/database username="dbadmin" password="v2password"
vault kv put secret/database username="dbadmin" password="v3password"

# Get specific version
vault kv get -version=1 secret/database

# Rollback to previous version
vault kv rollback -version=1 secret/database

# Delete specific version
vault kv delete -versions=1 secret/database
```

## Step 4: Dynamic Secrets

### AWS Dynamic Secrets

```
bash
# Configure AWS secrets engine
vault write aws/config/root \
    access_key=AKIAIOSFODNN7EXAMPLE \
    secret_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
    region=us-east-1

# Create a role for dynamic credentials
vault write aws/roles/myapp-role \
    credential_type=iam_user \
    policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::myapp-bucket/*"
    }
  ]
}
EOF

# Generate dynamic credentials
vault read aws/creds/myapp-role

# Example output:
# lease_id       = aws/creds/myapp-role/xxxxx
# lease_duration = 3600s
# access_key     = AKIAXXXXXXXXXXXXX
# secret_key     = xxxxxxxxxxxxxxxxxxxxxx

# Revoke credentials
vault lease revoke aws/creds/myapp-role/xxxxx
```

### Database Dynamic Secrets

```
bash
# Configure database secrets engine
vault secrets enable database

# Configure MySQL connection
vault write database/config/mysql \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(localhost:3306)/" \
    allowed_roles="app-role" \
    username="root" \
    password="rootpassword"

# Create a role
vault write database/roles/app-role \
    db_name=mysql \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl=1h \
    max_ttl=24h

# Generate dynamic credentials
vault read database/creds/app-role

# Revoke credentials
vault lease revoke database/creds/app-role/xxxxx
```

## Step 5: Authentication Methods

### Kubernetes Authentication

```
bash
# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
    kubernetes_host=https://kubernetes.default.svc \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create a policy for the application
vault policy write app-policy -<<EOF
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

path "database/creds/app-role" {
  capabilities = ["read"]
}
EOF

# Create Kubernetes auth role
vault write auth/kubernetes/role/myapp \
    bound_service_account_names=myapp-sa \
    bound_service_account_namespaces=default \
    policies=app-policy \
    ttl=1h
```

### AppRole Authentication

```
bash
# Enable AppRole auth method
vault auth enable approle

# Create a policy
vault policy write myapp-policy -<<EOF
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

# Create AppRole role
vault write auth/approle/role/myapp \
    policies="myapp-policy" \
    token_ttl=1h \
    token_max_ttl=4h

# Get role ID
vault read auth/approle/role/myapp/role-id

# Generate secret ID
vault write -f auth/approle/role/myapp/secret-id

# Login with AppRole
vault write auth/approle/login \
    role_id=<role-id> \
    secret_id=<secret-id>
```

## Step 6: Vault Agent Injector

### Configure Vault Agent Injector

```
bash
# Add HashiCorp repo and update
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Install with agent injector
helm install vault hashicorp/vault \
  --namespace vault \
  --set "injector.enabled=true" \
  --set "server.dev.enabled=true"

# Create a ServiceAccount for the app
kubectl create serviceaccount myapp-sa

# Create a Kubernetes auth role (run inside Vault)
kubectl exec -it vault-0 -n vault -- vault write auth/kubernetes/role/myapp \
    bound_service_account_names=myapp-sa \
    bound_service_account_namespaces=default \
    policies=default \
    ttl=20m
```

### Annotations for Vault Injection

```
yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/myapp/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/myapp/config" -}}
          export DB_USERNAME="{{ .Data.data.username }}"
          export DB_PASSWORD="{{ .Data.data.password }}"
          {{- end }}
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: myapp
        image: myapp:latest
        env:
        - name: DB_HOST
          value: "postgres.example.com"
```

## Step 7: Vault in CI/CD Pipeline

### Jenkins Pipeline with Vault

```
groovy
// Jenkinsfile with Vault integration
pipeline {
    agent {
        docker {
            image 'hashicorp/vault:1.13.0'
            args '-u root:root'
        }
    }
    
    environment {
        VAULT_ADDR = 'http://vault:8200'
        VAULT_TOKEN = credentials('vault-token')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Fetch Secrets') {
            steps {
                sh '''
                    # Enable secrets engine
                    vault secrets enable -path=secret kv-v2 || true
                    
                    # Read secrets
                    export DB_PASSWORD=$(vault kv get -field=password secret/database)
                    export API_KEY=$(vault kv get -field=api_key secret/myapp)
                    
                    # Write to file for build
                    echo "DB_PASSWORD=$DB_PASSWORD" > .env
                    echo "API_KEY=$API_KEY" >> .env
                '''
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    source .env
                    ./gradlew build
                '''
            }
        }
        
        stage('Deploy Credentials') {
            steps {
                withCredentials([string(credentialsId: 'vault-token', variable: 'VAULT_TOKEN')]) {
                    sh '''
                        # Store deployment credentials
                        vault kv put secret/deployment/${BUILD_NUMBER} \
                            image_tag=${IMAGE_TAG} \
                            deploy_time=${BUILD_TIMESTAMP}
                        
                        # Read previous deployment info
                        vault kv get secret/deployment/${BUILD_NUMBER}
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh 'rm -f .env'
        }
    }
}
```

### GitHub Actions with Vault

```
yaml
# .github/workflows/deploy.yml
name: Deploy with Vault

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Vault
        uses: hashicorp/vault-action@v2.6.0
        with:
          url: ${{ secrets.VAULT_ADDR }}
          token: ${{ secrets.VAULT_TOKEN }}
          secrets: |
            secret/data/myapp config | jq -r '.data.data'
      
      - name: Deploy application
        run: |
          echo "Deploying with database password: ${{ steps.setup-vault.outputs.db_password }}"
          # Deployment commands here
```

## Step 8: Encryption as a Service

### Configure Transit Secrets Engine

```
bash
# Enable transit secrets engine
vault secrets enable -path=transit transit

# Create encryption key
vault write -f transit/keys/myapp-key

# Encrypt data
vault write transit/encrypt/myapp-key plaintext=$(echo "sensitive data" | base64)

# Decrypt data
vault write transit/decrypt/myapp-key ciphertext="vault:v1:xxxxx"

# Rotate key
vault write -f transit/keys/myapp-key/rotate

# Rewrap data (with new key version)
vault write transit/rewrap/myapp-key ciphertext="vault:v1:xxxxx"
```

### Use Transit for Application Encryption

```
bash
# Store encryption key reference
vault write transit/keys/app-key \
    type=aes256-gcm

# Encrypt a password
vault write transit/encrypt/app-key \
    plaintext=$(echo "MySecretPassword123" | base64)

# Decrypt
vault write transit/decrypt/app-key \
    ciphertext="vault:v1:I48DqXO3ZJbi..."
```

## Step 9: Policies

### Create Fine-Grained Policies

```
bash
# Create admin policy
vault policy write admin -<<EOF
# Manage secret engines
path "sys/mounts/*" {
  capabilities = ["create", "read", "update", "delete"]
}

# Manage auth methods
path "sys/auth/*" {
  capabilities = ["create", "read", "update", "delete"]
}

# Full access to all secrets
path "*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF

# Create developer policy
vault policy write developer -<<EOF
# Read all secrets
path "secret/data/*" {
  capabilities = ["read", "list"]
}

# Read AWS secrets
path "aws/creds/*" {
  capabilities = ["read"]
}

# Read database credentials
path "database/creds/*" {
  capabilities = ["read"]
}
EOF

# Create readonly policy
vault policy write readonly -<<EOF
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
EOF

# List policies
vault policy list

# Read a policy
vault policy read developer
```

## Step 10: Monitoring and Auditing

### Enable Audit Logging

```
bash
# Enable file audit device
vault audit enable file file_path=/var/log/vault/audit.log

# Enable multiple audit devices
vault audit enable -path=syslog syslog

# List audit devices
vault audit list

# Disable audit device
vault audit disable file/
```

### Monitor Vault Metrics

```
bash
# Enable Prometheus metrics
vault write sys/config/state sm_enable_metrics=true

# Check metrics endpoint
curl -H "X-Vault-Token: $VAULT_TOKEN" $VAULT_ADDR/v1/sys/metrics?format=prometheus

# Use Vault audit for compliance
# Audit logs show:
# - Who accessed what secrets
# - When secrets were accessed
# - What operations were performed
```

## Step 11: Complete Application Integration Example

### Python Application with Vault

```
python
# app.py
import hvac
import os

class VaultClient:
    def __init__(self, vault_addr=None, token=None):
        self.client = hvac.Client(
            url=vault_addr or os.environ.get('VAULT_ADDR', 'http://localhost:8200'),
            token=token or os.environ.get('VAULT_TOKEN')
        )
    
    def get_secret(self, path, key):
        """Get a specific key from a secret path"""
        secret = self.client.secrets.kv.v2.read_secret_version(path=path)
        return secret['data']['data'][key]
    
    def get_database_credentials(self, role):
        """Get dynamic database credentials"""
        return self.client.secrets.database.generate_credentials(role)

# Usage example
if __name__ == "__main__":
    vault = VaultClient()
    
    # Get static secret
    db_password = vault.get_secret('myapp/config', 'password')
    print(f"Database password: {db_password}")
    
    # Get dynamic credentials
    creds = vault.get_database_credentials('app-role')
    print(f"Database user: {creds['username']}")
```

### Environment Variables from Vault

```
bash
#!/bin/bash
# get-secrets.sh

export VAULT_ADDR="${VAULT_ADDR:-http://localhost:8200}"
export VAULT_TOKEN="${VAULT_TOKEN:-myroot}"

# Fetch secrets and export as environment variables
export DB_USERNAME=$(vault kv get -field=username secret/myapp/database)
export DB_PASSWORD=$(vault kv get -field=password secret/myapp/database)
export API_KEY=$(vault kv get -field=api_key secret/myapp/api)

# Run the application
exec "$@"
```

## Conclusion

This project demonstrates:
- Installing and configuring HashiCorp Vault
- Working with KV secrets engine (static secrets)
- Generating dynamic secrets for AWS and databases
- Setting up Kubernetes and AppRole authentication
- Using Vault Agent Injector for Kubernetes
- Integrating Vault in CI/CD pipelines (Jenkins, GitHub Actions)
- Implementing encryption as a service
- Creating fine-grained policies
- Enabling audit logging and monitoring

## Additional Resources
- [HashiCorp Vault Documentation](https://www.vaultproject.io/docs)
- [Vault with Kubernetes](https://www.vaultproject.io/docs/platform/k8s)
- [Vault Agent Injector](https://www.vaultproject.io/docs/agent-injector)
- [HVAC Library](https://hvac.readthedocs.io/)
