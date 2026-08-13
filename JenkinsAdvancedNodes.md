# JENKINS ADVANCED - NODES, AGENTS & DISTRIBUTED BUILD

## Project Overview

This project covers advanced Jenkins configuration including distributed builds with agents/nodes, managing Jenkins infrastructure, and scaling Jenkins for enterprise deployments.

### Objectives
1. Understand Jenkins architecture and distributed builds
2. Configure Jenkins agents (SSH, Docker, Kubernetes)
3. Implement agent pools and labels
4. Configure cloud-based agents
5. Set up high availability
6. Manage Jenkins security at scale

## Prerequisites
- Jenkins installed
- Multiple Linux machines or cloud instances
- SSH access between machines
- Docker (for Docker agents)

## Step 1: Jenkins Architecture Overview

### Master-Agent Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     JENKINS MASTER                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Job       │  │  Scheduler  │  │   Security Realm    │ │
│  │   Config    │  │             │  │                     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Build      │  │  Pipeline   │  │   Plugins           │ │
│  │  Queue      │  │  Executor   │  │   Manager           │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│                           │                                  │
│         ┌─────────────────┼─────────────────┐               │
│         │                 │                 │               │
│         ▼                 ▼                 ▼               │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐       │
│  │  Agent 1   │    │  Agent 2   │    │  Agent 3   │       │
│  │  (Linux)   │    │  (Linux)   │    │  (Windows) │       │
│  └────────────┘    └────────────┘    └────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Understanding Executors

```bash
# View executor status
# In Jenkins UI: Manage Jenkins → System Information

# Configure executors via Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 set-num-executors -n nodeName 4

# Configure via Jenkins Script Console
Jenkins.instance.setNumExecutors(4)
Jenkins.instance.getNodes().each { node ->
    node.setNumExecutors(2)
}
```

## Step 2: Configure SSH Agents

### Set Up SSH Agent

```bash
# On Agent Node: Install Java
sudo apt update
sudo apt install -y openjdk-11-jdk

# Create Jenkins user
sudo useradd -m -s /bin/bash jenkins
sudo mkdir -p /home/jenkins/.ssh
sudo chown -R jenkins:jenkins /home/jenkins

# Generate SSH key
sudo su - jenkins
ssh-keygen -t rsa -b 4096 -C "jenkins-agent" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Configure in Jenkins UI

```bash
# Steps to add SSH Agent:
# 1. Go to Manage Jenkins → Manage Nodes
# 2. Click "New Node"
# 3. Enter name: "linux-agent-1"
# 4. Select "Permanent Agent"
# 5. Configure:
#    - Remote root directory: /home/jenkins/workspace
#    - Labels: "linux docker build"
#    - Usage: "Use this node as much as possible"
#    - Launch method: "Launch agents via SSH"
#    - Host: <agent-ip>
#    - Credentials: Add SSH username with private key
#    - Host Key Verification Strategy: "Known hosts file"
#    - Availability: "Keep this agent online as much as possible"
```

### Configure via Jenkins Configuration as Code (JCasC)

```yaml
# jenkins.yaml
jenkins:
  numExecutors: 4
  mode: NORMAL
  scmCheckoutRetry: 2

  nodes:
    - permanent:
        name: "linux-agent-1"
        remoteFS: "/home/jenkins"
        numExecutors: 2
        labels: "linux docker build"
        launcher:
          ssh:
            host: "agent1.example.com"
            port: 22
            credentialsId: "ssh-agent-credentials"
            javaPath: "/usr/bin/java"
            jvmOptions: "-Xmx512m"
            prefixStartSlaveCmd: ""
            suffixStartSlaveCmd: ""

    - permanent:
        name: "linux-agent-2"
        remoteFS: "/home/jenkins"
        numExecutors: 2
        labels: "linux docker"
        launcher:
          ssh:
            host: "agent2.example.com"
            port: 22
            credentialsId: "ssh-agent-credentials"
```

## Step 3: Configure Docker Agents

### Docker Agent Configuration

```bash
# Install Docker on agent machine
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
sudo systemctl restart docker

# Create Docker network for Jenkins
docker network create jenkins-network
```

### Configure Docker Agent in Jenkins

```bash
# Install Docker Pipeline plugin
# Go to: Manage Jenkins → Manage Plugins → Available
# Search and install: "Docker Pipeline"

# Add Docker Hub credentials
# Go to: Manage Jenkins → Manage Credentials
# Add Username/Password credentials for Docker Hub
```

### Jenkinsfile with Docker Agent

```groovy
// Jenkinsfile with Docker Agent
pipeline {
    agent {
        docker {
            image 'maven:3.8-openjdk-11'
            label 'docker'
            args '-v /root/.m2:/root/.m2'
        }
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
}
```

### Using Custom Docker Image as Agent

```groovy
pipeline {
    agent {
        docker {
            image 'my-custom-jenkins-agent:latest'
            label 'docker'
            args '-u root:root'
        }
    }
    
    environment {
        MAVEN_OPTS = '-Xmx1024m'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
        
        stage('Test') {
            steps {
                sh 'make test'
            }
        }
    }
}
```

## Step 4: Kubernetes Agents

### Install Kubernetes Plugin

```bash
# Install Kubernetes plugin
# Go to: Manage Jenkins → Manage Plugins → Available
# Search and install: "Kubernetes"

# Configure Kubernetes cloud
# Go to: Manage Jenkins → Manage Nodes → Configure Clouds
# Add new cloud → Kubernetes
```

### Configure Kubernetes Cloud

```yaml
# Kubernetes Plugin Configuration
kubernetes:
  name: "kubernetes"
  serverUrl: "https://kubernetes.default.svc"
  namespace: "jenkins"
  jenkinsUrl: "http://jenkins:8080"
  jenkinsTunnel: "jenkins-agent:50000"
  connectTimeout: 5
  readTimeout: 15
  containerCapStr: "100"
  maxRequestsPerHostStr: "32"
  retentionTimeout: 5

  # Pod templates
  podTemplates:
    - name: "maven"
      label: "maven"
      containers:
        - name: "maven"
          image: "maven:3.8-openjdk-11"
          workingDir: "/workspace"
          command: ""
          ttyEnabled: true
          resourceRequestCpu: "500m"
          resourceRequestMemory: "512Mi"
          resourceLimitCpu: "1000m"
          resourceLimitMemory: "1Gi"

    - name: "nodejs"
      label: "nodejs"
      containers:
        - name: "nodejs"
          image: "node:18"
          workingDir: "/workspace"
          command: ""
          ttyEnabled: true

    - name: "python"
      label: "python"
      containers:
        - name: "python"
          image: "python:3.10"
          workingDir: "/workspace"
          command: ""
          ttyEnabled: true
```

### Jenkinsfile with Kubernetes Agent

```groovy
pipeline {
    agent {
        kubernetes {
            label 'maven'
            defaultContainer 'maven'
        }
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        
        stage('Test') {
            steps {
                container('maven') {
                    sh 'mvn test'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    container('maven') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }
    }
}
```

## Step 5: Agent Labels and Distribution

### Using Labels

```groovy
// Jenkinsfile with specific label
pipeline {
    agent {
        label 'linux && docker && maven'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}

// Using multiple agents
pipeline {
    stages {
        stage('Build on Linux') {
            agent {
                label 'linux'
            }
            steps {
                sh 'make build-linux'
            }
        }
        
        stage('Build on Windows') {
            agent {
                label 'windows'
            }
            steps {
                bat 'make build-windows'
            }
        }
        
        stage('Build on macOS') {
            agent {
                label 'macos'
            }
            steps {
                sh 'make build-macos'
            }
        }
    }
}
```

### Matrix Project with Labels

```groovy
// Matrix project configuration
matrix {
    axes {
        axis {
            name 'PLATFORM'
            values 'linux', 'windows', 'macos'
        }
        axis {
            name 'BROWSER'
            values 'chrome', 'firefox', 'safari'
        }
    }
    
    agent {
        label "PLATFORM:${PLATFORM}"
    }
    
    stages {
        stage('Test') {
            steps {
                sh "run-tests.sh --platform=${PLATFORM} --browser=${BROWSER}"
            }
        }
    }
}
```

## Step 6: Cloud-Based Agents

### Configure AWS EC2 Agents (EC2 Fleet)

```bash
# Install EC2 Fleet Plugin
# Go to: Manage Jenkins → Manage Plugins → Available
# Search and install: "EC2 Fleet"

# Configure EC2 Fleet
# Go to: Manage Jenkins → Manage Nodes → Configure Clouds
# Add: "Amazon EC2 Fleet"
```

### EC2 Fleet Configuration

```yaml
# EC2 Fleet Configuration
ec2Fleet:
  name: "ec2-fleet"
  cloudName: "aws"
  region: "us-east-1"
  fleetType: "spot"
  
  # Fleet configuration
  idleMinutesBeforeScalingDown: 3
  minSize: 0
  maxSize: 10
  desiredCapacity: 2
  
  # Instance configuration
  instanceType: "t3.medium"
  ami: "ami-0c55b159cbfafe1f0"
  sshPort: 22
  
  # IAM role
  iamInstanceProfile: "jenkins-agent-role"
  
  # Security groups
  securityGroups: "sg-jenkins-agents"
  
  # Key pair
  keyPairName: "jenkins-key"
  
  # Labels
  labels: "ec2 spot build"
  
  # Remote root directory
  remoteFS: "/home/jenkins"
  
  # Connection strategy
  connectStrategy: "PUBLIC_IP"
  
  # User data
  userData: "#!/bin/bash\nmkdir -p /home/jenkins && useradd -m -s /bin/bash jenkins"
```

### Configure Azure Agents

```bash
# Install Azure VM Agents Plugin
# Go to: Manage Jenkins → Manage Plugins → Available
# Search and install: "Azure VM Agents"
```

### Azure VM Configuration

```yaml
# Azure VM Agents Configuration
azure:
  name: "azure-agents"
  subscriptionId: "your-subscription-id"
  
  vmTemplates:
    - name: "linux-agent"
      location: "East US"
      vmSize: "Standard_D2s_v3"
      storageAccountType: "Standard_LRS"
      
      # Image
      imageReference:
        publisher: "Canonical"
        offer: "UbuntuServer"
        sku: "18.04-LTS"
        version: "latest"
      
      # Network
      resourceGroup: "jenkins-rg"
      subnetId: "/subscriptions/xxx/resourceGroups/jenkins-rg/providers/Microsoft.Network/virtualNetworks/jenkins-vnet/subnets/default"
      
      # Security
      sshPublicKey: "ssh-rsa AAAA..."
      initScript: |
        #!/bin/bash
        apt-get update
        apt-get install -y openjdk-11-jdk
        useradd -m -s /bin/bash jenkins
      
      # Configuration
      jenkinsController:
        url: "http://jenkins:8080"
        tunnel: "jenkins-agent:50000"
      
      labels: "azure linux"
      remoteFS: "/home/jenkins"
      usageMode: "NORMAL"
      executors: 2
```

## Step 7: High Availability

### Configure Jenkins for High Availability

```groovy
// Install HA plugin and configure
// Go to: Manage Jenkins → System
// Configure: "Kubernetes HA"

pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                echo "Building on ${env.JENKINS_URL}"
                sh 'mvn clean package'
            }
        }
    }
}
```

### Backup Configuration

```bash
# Install ThinBackup Plugin
# Configure backup schedule

# Configure in Jenkins System
# ThinBackup → Settings
# Backup schedule: H 2 * * *
# Full backup schedule: H H 0 * *
# Backup directory: /var/jenkins_backup
# Max backup sets: 10
```

### Restore Script

```bash
#!/bin/bash
# restore_jenkins.sh

JENKINS_HOME="/var/jenkins_home"
BACKUP_DIR="/var/jenkins_backup"
BACKUP_FILE="$1"

echo "Stopping Jenkins..."
systemctl stop jenkins

echo "Restoring from backup: $BACKUP_FILE"
tar -xzf "$BACKUP_FILE" -C "$JENKINS_HOME"

echo "Setting permissions..."
chown -R jenkins:jenkins "$JENKINS_HOME"

echo "Starting Jenkins..."
systemctl start jenkins

echo "Restore complete!"
```

## Step 8: Agent Monitoring

### Monitor Agents

```groovy
// Script to monitor agent status
import hudson.model.*

// Get all nodes
def nodes = Hudson.instance.nodes

nodes.each { node ->
    println "Node: ${node.displayName}"
    println "  Status: ${node.toComputer().online ? 'ONLINE' : 'OFFLINE'}"
    println "  Executors: ${node.numExecutors}"
    println "  Labels: ${node.labelString}"
    println "  Idle: ${node.toComputer().idle}"
    println ""
}

// Check disk space
def thread = Thread.start {
    while(true) {
        def disk = new File("/").totalSpace
        def free = new File("/").freeSpace
        println "Disk: ${(free/1024/1024/1024).round(2)} GB free of ${(disk/1024/1024/1024).round(2)} GB"
        sleep(60000)
    }
}
```

### Agent Health Check Script

```groovy
// Automated health check
import hudson.model.*

def checkAgents() {
    def nodes = Hudson.instance.nodes
    def offlineAgents = []
    
    nodes.each { node ->
        def computer = node.toComputer()
        if (!computer.online) {
            offlineAgents.add(node.displayName)
        }
    }
    
    if (offlineAgents.size() > 0) {
        // Send notification
        emailext (
            subject: "Jenkins Agents Offline",
            body: "The following agents are offline: ${offlineAgents.join(', ')}",
            to: "jenkins-admin@example.com"
        )
    }
}

checkAgents()
```

## Step 9: Security and Access Control

### Role-Based Access Control

```bash
# Install Role-based Authorization Strategy
# Go to: Manage Jenkins → Manage Plugins → Available
# Install: "Role-based Authorization Strategy"

# Configure in: Manage Jenkins → Configure Global Security
# Authorization: Role-Based Strategy
```

### Manage Roles

```groovy
// Create roles via Jenkins Script Console
import org.jenkinsci.plugins.rolestrategy.Role
import org.jenkinsci.plugins.rolestrategy.RoleType
import com.michelin.rolestrategy.RoleStrategy

def roleStrategy = RoleStrategy.getInstance()

// Admin role
def adminRole = new Role("admin", ".*", RoleType.Global)
adminRole.add("Overall/Administer")

// Developer role
def developerRole = new Role("developer", ".*", RoleType.Global)
developerRole.add("Overall/Read")
developerRole.add("Job/Build")
developerRole.add("Job/Create")
developerRole.add("Job/Delete")

// Assign roles
roleStrategy.addRole(RoleType.Global, adminRole)
roleStrategy.addRole(RoleType.Global, developerRole)

// Assign to user
roleStrategy.assignRole(RoleType.Global, "admin-user", "admin")
roleStrategy.assignRole(RoleType.Global, "dev-user", "developer")
```

### Matrix Authorization

```yaml
# Matrix Authorization Configuration
jenkins:
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"
        - "Job/Build:developer"
        - "Job/Create:developer"
        - "Job/Delete:admin"
        - "View/Read:authenticated"
```

## Step 10: Performance Optimization

### Configure Executors

```groovy
// Optimize executor count
// Master node should not run builds
Jenkins.instance.numExecutors = 0

// Set executors on agents
Jenkins.instance.slaves.each { agent ->
    agent.numExecutors = 2
    agent.save()
}

// Verify
Jenkins.instance.nodes.each { node ->
    println "${node.displayName}: ${node.numExecutors} executors"
}
```

### Resource Management

```groovy
// Monitor build resources
def getBuildMetrics() {
    def builds = Jenkins.instance.getAllItems(Job.class).collectMany { job ->
        job.builds
    }
    
    def avgDuration = builds.collect { it.duration }.average() ?: 0
    def totalBuilds = builds.size()
    def successBuilds = builds.count { it.result == Result.SUCCESS }
    
    println "Total Builds: ${totalBuilds}"
    println "Success Rate: ${(successBuilds/totalBuilds*100).round(2)}%"
    println "Average Duration: ${(avgDuration/1000/60).round(2)} minutes"
}

getBuildMetrics()
```

### Build Queue Management

```groovy
// Manage build queue
import hudson.model.Queue

def queue = Hudson.instance.queue

println "Queue items: ${queue.items.size()}"

queue.items.each { item ->
    println "  - ${item.task.name} (Waiting for ${item.inQueueSince/1000}s)"
}
```

## Conclusion

This project covers:
- Jenkins master-agent architecture
- SSH agent configuration
- Docker agents and containers
- Kubernetes agents
- Cloud-based agents (AWS, Azure)
- Agent labels and distribution
- High availability setup
- Backup and restore
- Agent monitoring and health checks
- Security and access control
- Performance optimization

## Additional Resources
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Jenkins Plugins](https://plugins.jenkins.io/)
- [Jenkins Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/)
- [Jenkins EC2 Fleet Plugin](https://plugins.jenkins.io/ec2-fleet/)
