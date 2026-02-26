# JENKINS ADMINISTRATION - COMPLETE GUIDE

## Project Overview

This project covers comprehensive Jenkins administration including installation, configuration, security, maintenance, troubleshooting, and enterprise-level deployments.

### Objectives
1. Install and configure Jenkins
2. Manage users and permissions
3. Configure security settings
4. Maintain and backup Jenkins
5. Troubleshoot common issues
6. Optimize performance

## Prerequisites
- Root/sudo access to Jenkins server
- Basic understanding of CI/CD
- Access to Jenkins web interface

## Step 1: Installation

### Install on Ubuntu/Debian

```
bash
# Update package index
sudo apt update

# Install Java (required)
sudo apt install -y openjdk-11-jdk

# Add Jenkins repository
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

# Install Jenkins
sudo apt update
sudo apt install -y jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check status
sudo systemctl status jenkins
```

### Install on CentOS/RHEL

```
bash
# Install Java
sudo yum install -y java-11-openjdk

# Add Jenkins repository
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

# Install Jenkins
sudo yum install -y jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

### Install with Docker

```
bash
# Pull Jenkins image
docker pull jenkins/jenkins:lts

# Run Jenkins container
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Install with Docker Compose

```
yaml
# docker-compose.yml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    restart: unless-stopped

volumes:
  jenkins_home:
```

## Step 2: Initial Configuration

### Setup Wizard

```
bash
# Access Jenkins at http://localhost:8080

# Step 1: Unlock Jenkins
# Enter initial admin password from:
# /var/jenkins_home/secrets/initialAdminPassword

# Step 2: Install suggested plugins
# Click "Install suggested plugins"

# Step 3: Create first admin user
# Username: admin
# Password: <strong-password>
# Full name: Jenkins Admin
# Email: admin@example.com

# Step 4: Configure Jenkins URL
# Jenkins URL: http://jenkins.example.com:8080/
```

### Manual Configuration

```
bash
# Disable setup wizard
mkdir -p /var/lib/jenkins/init.groovy.d

cat > /var/lib/jenkins/init.groovy.d/disable-setup-wizard.groovy <<EOF
import jenkins.install.InstallState;
import jenkins.install.InstallStateProperty;

if (!InstallState.INITIALIZED.equals(jenkins.model.Jenkins.instance.installState)) {
    jenkins.model.Jenkins.instance.installState = new InstallStateProperty(InstallState.INITIALIZED);
}
EOF

# Restart Jenkins
sudo systemctl restart jenkins
```

### Configure System Settings

```
groovy
// Configure in Manage Jenkins → Configure System

// General Settings
// Jenkins URL: http://jenkins.example.com
// System Admin e-mail address: admin@example.com

// Global Properties
// Environment variables:
// MAVEN_HOME=/opt/maven
// JAVA_HOME=/usr/lib/jvm/java-11

// Shell:
// Shell: /bin/bash

// Enable security
// Security: Enable
```

## Step 3: User Management

### Create Users

```
bash
# Create user via Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 \
  create-user admin admin admin@example.com

# Create user via Jenkins Script Console
import jenkins.model.Jenkins
import hudson.security.FullControlOnceLoggedInAuthorizationStrategy

def user = hudson.model.User.get('newuser')
user.setFullName('New User')
user.addProperty(hudson.security.HudsonPrivateSecurityRealmDetails.class)
password = hudson.util.Secret.fromString('password123')
user.getProperty(hudson.security.HudsonPrivateSecurityRealmDetails.class).setPassword(password)
user.save()
```

### LDAP Integration

```
groovy
// Configure LDAP in Jenkins System
// Go to: Manage Jenkins → Configure Global Security
// Security Realm: LDAP
// Server: ldap://ldap.example.com
// Base DN: dc=example,dc=com
// User search Base: ou=users
// User search Filter: uid={0}
// Group search Base: ou=groups
// Group search Filter: cn={0}

// Configure using Jenkins Configuration as Code
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: "ldap://ldap.example.com"
          rootDN: "dc=example,dc=com"
          userSearchBase: "ou=users"
          userSearchFilter: "uid={0}"
          groupSearchBase: "ou=groups"
```

### SSO with OAuth

```
groovy
// Install OAuth plugin
// Configure GitHub OAuth:
// 1. Create GitHub OAuth App
// 2. Configure in Jenkins

// Jenkins Configuration as Code
jenkins:
  securityRealm:
    github:
      clientID: "your-client-id"
      clientSecret: "your-client-secret"
      githubWebUri: "https://github.com"
      apiUri: "https://api.github.com"
      oauthScopes: "read:user,user:email"
```

## Step 4: Security Configuration

### Configure Security

```
groovy
// Configure in: Manage Jenkins → Configure Global Security

// Authorization Matrix
// Matrix-based security:
//   Admin: Overall/Administer
//   Developer: Overall/Read, Job/Build, Job/Create
//   Viewer: Overall/Read

// Pipeline Policies
// Approve script signatures:
// Manage Jenkins → In-process Script Approval
```

### CSRF Protection

```
groovy
// Enable CSRF Protection
// Configure in: Manage Jenkins → Configure Global Security
// Prevent Cross Site Request Forgery exploits: Enabled
// Crumb Algorithm: DefaultCrumbIssuer
```

### API Token

```
bash
# Create API token
# Go to: User → Configure → API Token
# Click "Add new Token"
# Generate and copy token

# Use API token
curl -u "username:api-token" http://localhost:8080/job/myjob/build
```

## Step 5: Plugin Management

### Install Plugins

```
bash
# Install plugins via Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 \
  install-plugin docker-workflow

# Install plugins via Jenkins Script Console
import jenkins.model.Jenkins
import hudson.util.PluginServletPlugin

def pluginManager = Jenkins.instance.pluginManager
def pluginName = "pipeline-utility-steps"

pluginManager.installPlugin(pluginName, false)

// Restart not needed for most plugins
```

### Update Plugins

```
bash
# Check for plugin updates
# Go to: Manage Jenkins → Manage Plugins → Updates

# Update all plugins
java -jar jenkins-cli.jar -s http://localhost:8080 \
  install-plugin -restart

# Update specific plugin
java -jar jenkins-cli.jar -s http://localhost:8080 \
  install-plugin workflow-aggregator -restart
```

### Configure Plugin Repository

```
groovy
// Configure custom plugin repository
// Go to: Manage Jenkins → Manage Plugins → Advanced
// Update Site: https://updates.jenkins.io/update-center.json

// Custom plugin site
// URL: https://jenkins-updates.example.com/update-center.json
```

## Step 6: Backup and Restore

### Manual Backup

```
bash
#!/bin/bash
# backup_jenkins.sh

JENKINS_HOME="/var/lib/jenkins"
BACKUP_DIR="/var/backups/jenkins"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# Backup Jenkins home
tar -czf "$BACKUP_DIR/jenkins_home_$DATE.tar.gz" \
  -C "$(dirname $JENKINS_HOME)" jenkins

# Keep only last 7 backups
find "$BACKUP_DIR" -name "jenkins_home_*.tar.gz" -mtime +7 -delete

echo "Backup complete: jenkins_home_$DATE.tar.gz"
```

### Automated Backup with ThinBackup

```
groovy
// Install ThinBackup Plugin
// Configure:
// Go to: Manage Jenkins → ThinBackup

// Settings:
// Backup schedule: H 2 * * *  (Daily at 2 AM)
// Full backup schedule: H H 0 * *  (Weekly on Sunday)
// Backup directory: /var/backups/jenkins
// Max backup sets: 10
// Files to backup: Configuration + Build history
// Backup build results: Checked
// Backup 'userContent' folder: Checked
```

### Restore from Backup

```
bash
#!/bin/bash
# restore_jenkins.sh

JENKINS_HOME="/var/lib/jenkins"
BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file.tar.gz>"
    exit 1
fi

# Stop Jenkins
systemctl stop jenkins

# Backup current Jenkins home
mv "$JENKINS_HOME" "${JENKINS_HOME}.old"

# Extract backup
tar -xzf "$BACKUP_FILE" -C "$(dirname $JENKINS_HOME)"

# Fix permissions
chown -R jenkins:jenkins "$JENKINS_HOME"

# Start Jenkins
systemctl start jenkins

echo "Restore complete!"
```

## Step 7: Maintenance

### Clean Up Old Builds

```
groovy
// Clean up old builds
import hudson.model.Job
import hudson.tasks.LogRotator

// Configure for all jobs
Jenkins.instance.items.each { job ->
    if (job instanceof hudson.model.FreeStyleProject) {
        // Set log rotation (keep 10 builds)
        job.logRotator = new LogRotator(-1, 10, -1, 10)
        job.save()
        println "Configured: ${job.name}"
    }
}

// Delete old builds older than 30 days
def cutoff = System.currentTimeMillis() - (30L * 24 * 60 * 60 * 1000)

Jenkins.instance.items.each { job ->
    job.builds.each { build ->
        if (build.timestamp.timeInMillis < cutoff) {
            build.delete()
            println "Deleted: ${job.name} #${build.number}"
        }
    }
}
```

### Clean Up Workspace

```
groovy
// Delete workspaces not in use
import hudson.model.Node

for (node in Jenkins.instance.nodes) {
    def workspacePath = node.getWorkspaceFor(new hudson.model.Job(hudson.model.Hudson.instance, "myjob"))
    if (workspacePath != null) {
        println "Workspace: ${workspacePath}"
    }
}

// Delete workspaces older than 7 days
def daysOld = 7
def cutoff = System.currentTimeMillis() - (daysOld * 24 * 60 * 60 * 1000)

Jenkins.instance.items.each { job ->
    job.builds.each { build ->
        def ws = build.getWorkspace()
        if (ws != null) {
            def lastBuild = build.timestamp.timeInMillis
            if (lastBuild < cutoff) {
                ws.deleteRecursive()
                println "Deleted workspace for ${job.name} #${build.number}"
            }
        }
    }
}
```

### Clean Up Logs

```
bash
# Clean old logs
find /var/log/jenkins -name "*.log.*" -mtime +30 -delete

# Compress old logs
find /var/log/jenkins -name "*.log" -mtime +7 -exec gzip {} \;

# Clear build logs older than 30 days
find /var/lib/jenkins/jobs -name "log" -mtime +30 -delete
```

## Step 8: Performance Tuning

### JVM Options

```
bash
# Configure JVM options
# Edit: /etc/default/jenkins

# Add/modify:
JENKINS_ARGS="--prefix=/jenkins"

# JVM options for production
JAVA_ARGS="-Xms1024m -Xmx2048m -XX:+UseG1GC -XX:+UseStringDeduplication -XX:+AlwaysPreTouch"

# Apply changes
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

### Optimize Executors

```
groovy
// Optimize executor count
// For master-only builds
Jenkins.instance.numExecutors = 2

// For distributed builds (master should not build)
Jenkins.instance.numExecutors = 0

// Set for each agent
Jenkins.instance.slaves.each { agent ->
    // Set based on CPU cores
    def cpuCount = Runtime.runtime.availableProcessors()
    agent.numExecutors = Math.max(1, cpuCount - 1)
    agent.save()
}
```

### Cache Configuration

```
groovy
// Configure caches
System.setProperty("hudson.model.LoadStatistics.decay", "0.5")
System.setProperty("hudson.model.Job.consoleLogDataThreshold", "500KB")

// Disable unnecessary features
System.setProperty("jenkins.model.Jenkins.minimumKillInterval", "30000")
System.setProperty("hudson.util.AtmosphereResourceCache.maxSizePerContext", "100")
```

## Step 9: Monitoring

### Prometheus Monitoring

```
groovy
// Install Prometheus Plugin
// Configure: Manage Jenkins → System → Prometheus

// Expose metrics endpoint
// http://jenkins:8080/prometheus/

// Use in Prometheus
scrape_configs:
  - job_name: 'jenkins'
    metrics_path: '/prometheus/'
    static_configs:
      - targets: ['jenkins:8080']
```

### Health Check

```
groovy
// Create health check endpoint
import javax.servlet.http.HttpServletResponse

def healthCheck() {
    def checks = [:]
    
    // Check disk space
    def rootFile = new File("/")
    def freeSpace = rootFile.freeSpace
    checks.disk = [status: freeSpace > 1e9 ? "OK" : "LOW", free: freeSpace]
    
    // Check memory
    def runtime = Runtime.runtime
    def freeMemory = runtime.freeMemory()
    checks.memory = [status: freeMemory > 1e8 ? "OK" : "LOW", free: freeMemory]
    
    // Check executors
    def executors = Jenkins.instance.executors
    def busy = executors.count { it.isBusy() }
    checks.executors = [busy: busy, total: executors.size()]
    
    return checks
}

healthCheck()
```

## Step 10: Troubleshooting

### Common Issues

```
bash
# Issue: Jenkins won't start
# Solution: Check logs
tail -f /var/log/jenkins/jenkins.log

# Check Java version
java -version

# Check port availability
netstat -tulpn | grep 8080

# Issue: Build stuck in queue
# Solution: Check if all executors are busy
# Increase executors or check for locked resources
```

### Memory Issues

```
bash
# Check memory usage
free -h

# Monitor Java memory
jmap -heap <jenkins-pid>

# Enable GC logging
JAVA_ARGS="-Xlog:gc*:file=/var/log/jenkins/gc.log:time,uptime,level,tags -Xms1024m -Xmx2048m"
```

### Plugin Issues

```
bash
# Disable plugin
# Rename plugin file
mv /var/lib/jenkins/plugins/docker-workflow.jpi /var/lib/jenkins/plugins/docker-workflow.jpi.disabled

# Restart Jenkins
systemctl restart jenkins

# Re-enable by renaming back
```

### Database/Build Issues

```
groovy
// Rebuild plugin index
Jenkins.instance.pluginManager.doCheckUpdatesServer()

// Reload configuration
Jenkins.instance.reload()

// Clear temporary files
new File(Jenkins.instance.rootDir, "tmp").deleteDir()
```

## Step 11: Script Console Examples

### User Management

```
groovy
// List all users
import hudson.model.User

User.getAll().each { user ->
    println "${user.id} - ${user.fullName} - ${user.email}"
}

// Create user programmatically
def user = User.get('newuser')
user.addProperty(hudson.security.HudsonPrivateSecurityRealmDetails.class)
user.getProperty(hudson.security.HudsonPrivateSecurityRealmDetails.class).setPassword(hudson.util.Secret.fromString('password'))
user.save()
```

### Job Management

```
groovy
// List all jobs
Jenkins.instance.items.each { job ->
    println "${job.name} - ${job.class.simpleName} - ${job.lastBuild?.number ?: 'No builds'}"
}

// Disable all jobs
Jenkins.instance.items.each { job ->
    job.disabled = true
    job.save()
    println "Disabled: ${job.name}"
}

// Delete old builds
Jenkins.instance.items.each { job ->
    def builds = job.builds
    def keep = 10
    while (builds.size() > keep) {
        builds.last().delete()
    }
}
```

### System Information

```
groovy
// Get system information
println "Jenkins Version: ${Jenkins.instance.version}"
println "Java Version: ${System.getProperty('java.version')}"
println "OS: ${System.getProperty('os.name')}"
println "Available Processors: ${Runtime.runtime.availableProcessors()}"
println "Total Memory: ${Runtime.runtime.totalMemory() / 1024 / 1024} MB"
println "Free Memory: ${Runtime.runtime.freeMemory() / 1024 / 1024} MB"
println "Plugins Count: ${Jenkins.instance.pluginManager.plugins.size()}"
println "Jobs Count: ${Jenkins.instance.items.size()}"
println "Executors: ${Jenkins.instance.numExecutors}"
```

## Step 12: Jenkins Configuration as Code (JCasC)

### Basic Configuration

```
yaml
# jenkins.yaml
jenkins:
  systemMessage: "Jenkins Managed by JCasC"
  numExecutors: 4
  mode: NORMAL
  scmCheckoutRetryCount: 2
  
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${JENKINS_ADMIN_PASSWORD}"
          
  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false

  remoting:
    enabled: true
    
  globalLibraries:
    libraries:
      - name: "shared-lib"
        defaultVersion: "main"
        retriever:
          modernSCM:
            scm:
              filesystem:
                path: "/var/jenkins_libraries"

  credentials:
    system:
      domainCredentials:
        - credentials:
            - string:
                id: "github-token"
                secret: "${GITHUB_TOKEN}"
                description: "GitHub API Token"
            - usernamePassword:
                id: "docker-hub"
                username: "${DOCKER_USER}"
                password: "${DOCKER_PASS}"
                description: "Docker Hub Credentials"
```

### Configure Tools

```
yaml
# tools.yaml
unclassified:
  location:
    url: "http://jenkins.example.com"
    adminAddress: "admin@example.com"
    
  maven:
    installations:
      - name: "maven-3.8"
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.8.6"
                    
  jdk:
    installations:
      - name: "jdk-11"
        properties:
          - installSource:
              installers:
                - jdk:
                    id: "11.0.14+9"
                    
  git:
    installations:
      - name: "git"
        properties:
          - installSource:
              installers:
                - git:
                    id: "Default"
                    
  docker:
    installations:
      - name: "docker"
        properties:
          - installSource:
              installers:
                - dockerTool:
                    id: "20.10"
                    url: "https://download.docker.com"
```

### Configure Jobs

```
yaml
# jobs.yaml
jobs:
  - script: >
      pipelineJob('example-pipeline') {
        definition {
          cpsScm {
            scm {
              git {
                remote('https://github.com/example/repo.git')
                branch('main')
              }
            }
            lightweight()
          }
        }
      }
```

## Conclusion

This project covers:
- Jenkins installation (various methods)
- Initial configuration and setup
- User management (local and LDAP)
- Security configuration
- Plugin management
- Backup and restore
- Maintenance tasks
- Performance tuning
- Monitoring
- Troubleshooting common issues
- Script console examples
- Configuration as Code (JCasC)

## Additional Resources
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Jenkins CLI](https://www.jenkins.io/doc/book/managing/cli/)
- [Jenkins Script Console](https://www.jenkins.io/doc/book/managing/script-console/)
- [Jenkins Plugins](https://plugins.jenkins.io/)
- [JCasC Documentation](https://jenkins.io/projects/jcasc/)
