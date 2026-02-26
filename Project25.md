# SECURITY SCANNING IN CI/CD PIPELINE

## Project Overview

This project demonstrates implementing security scanning in CI/CD pipelines to identify vulnerabilities early in the development lifecycle. We will integrate SAST (Static Application Security Testing), DAST (Dynamic Application Security Testing), and container security scanning into Jenkins pipelines.

### Objectives
1. Integrate SAST tools (SonarQube, Semgrep, Bandit)
2. Implement DAST tools (OWASP ZAP, Nikto)
3. Add container security scanning (Trivy, Clair)
4. Create a comprehensive security pipeline
5. Generate security reports and metrics

## Prerequisites
- Jenkins installed and configured
- Docker installed
- AWS Account (for deployment)
- GitHub repository with application code

## Step 1: Set Up SonarQube for SAST

### Install SonarQube using Docker

```
bash
# Create Docker network
docker network create sonarnet

# Start SonarQube container
docker run -d --name sonarqube \
  --network sonarnet \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:latest

# Wait for SonarQube to start (check logs)
docker logs -f sonarqube

# Access SonarQube at http://localhost:9000
# Default credentials: admin/admin
```

### Generate SonarQube Token

1. Log in to SonarQube
2. Go to My Account → Security
3. Generate a new token: "jenkins-token"
4. Copy and save the token

### Install SonarQube Scanner Plugin in Jenkins

```
bash
# In Jenkins dashboard:
# 1. Manage Jenkins → Manage Plugins
# 2. Go to Available tab
# 3. Search for "SonarQube Scanner for Jenkins"
# 4. Install without restart
```

### Configure SonarQube in Jenkins

```
bash
# In Jenkins dashboard:
# 1. Manage Jenkins → Configure System
# 2. Scroll to SonarQube servers section
# 3. Click "Add SonarQube"
# 4. Configure:
#    - Name: SonarQube
#    - Server URL: http://sonarqube:9000
#    - Server authentication token: (create in Jenkins credentials)
```

## Step 2: Create Jenkins Pipeline with SAST

```
groovy
// Jenkinsfile with SAST integration
pipeline {
    agent any
    
    environment {
        SONAR_HOST = 'http://sonarqube:9000'
        SONAR_TOKEN = credentials('sonarqube-token')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=myapp \
                        -Dsonar.host.url=${SONAR_HOST} \
                        -Dsonar.login=${SONAR_TOKEN} \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.java.source=11
                    '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed'
        }
        failure {
            emailext body: 'Build Failed', subject: 'Jenkins Build Failed', to: 'admin@example.com'
        }
    }
}
```

## Step 3: Add SAST Tools

### Add Bandit (Python Security Scanner)

```
groovy
// Add Bandit scanning stage
stage('Bandit Security Scan') {
    steps {
        sh '''
            # Install bandit
            pip install bandit
            
            # Run bandit scan
            bandit -r . -f json -o bandit-report.json || true
            
            # Generate HTML report
            bandit -r . -f html -o bandit-report.html || true
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'bandit-report.*', allowEmptyArchive: true
            recordIssues(
                enabledForFailure: true,
                tools: [bandit()],
                qualityGates: [[threshold: 10, type: 'TOTAL', unstable: true]]
            )
        }
    }
}
```

### Add Semgrep (Multi-language Security Scanner)

```
groovy
// Add Semgrep scanning stage
stage('Semgrep Security Scan') {
    steps {
        sh '''
            # Install semgrep
            pip install semgrep
            
            # Run semgrep scan
            semgrep --config=auto --json --output=semgrep-report.json . || true
            
            # Run with security rules
            semgrep --config=security --json --output=semgrep-security.json . || true
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'semgrep-*.json', allowEmptyArchive: true
            recordIssues(
                enabledForFailure: true,
                tools: [semgrep()],
                qualityGates: [[threshold: 5, type: 'HIGH', unstable: true]]
            )
        }
    }
}
```

### Add OWASP Dependency Check

```
groovy
// Add dependency vulnerability scanning
stage('OWASP Dependency Check') {
    steps {
        sh '''
            # Install dependency-check
            wget https://github.com/jeremylong/DependencyCheck/releases/download/v7.4.1/dependency-check-7.4.1-release.zip
            unzip dependency-check-7.4.1-release.zip
            
            # Run dependency check
            ./dependency-check/bin/dependency-check.sh \
                --project "MyApp" \
                --scan ./target \
                --format XML \
                --output dependency-check-report.xml
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'dependency-check-report.*', allowEmptyArchive: true
            recordIssues(
                enabledForFailure: true,
                tools: [dependencyCheck(pattern: 'dependency-check-report.xml')],
                qualityGates: [[threshold: 10, type: 'HIGH', unstable: true]]
            )
        }
    }
}
```

## Step 4: Implement DAST (Dynamic Analysis)

### Add OWASP ZAP Scanning

```
groovy
// Add OWASP ZAP DAST scanning
stage('OWASP ZAP Scan') {
    steps {
        sh '''
            # Pull OWASP ZAP Docker image
            docker pull owasp/zap2docker-stable
            
            # Start ZAP in daemon mode
            docker run -d --name zap \
                -p 8080:8080 \
                owasp/zap2docker-stable zap.sh -daemon -port 8080 -config api.key=zapapikey
            
            # Wait for ZAP to start
            sleep 10
            
            # Spider the application
            curl http://localhost:8080/JSON/spider/action/scan/?url=http://target-app.example.com
            
            # Wait for spider to complete
            sleep 30
            
            # Run active scan
            curl http://localhost:8080/JSON/ascan/action/scan/?url=http://target-app.example.com
            
            # Wait for scan to complete
            sleep 60
            
            # Generate JSON report
            curl http://localhost:8080/JSON/report/authenticted/report/ -o zap-report.json || true
            
            # Stop ZAP container
            docker stop zap
            docker rm zap
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'zap-report.*', allowEmptyArchive: true
            recordIssues(
                enabledForFailure: true,
                tools: [zap(pattern: 'zap-report.json')],
                qualityGates: [[threshold: 3, type: 'HIGH', unstable: true]]
            )
        }
    }
}
```

### Add Nikto Scanning

```
groovy
// Add Nikto web server scanner
stage('Nikto Scan') {
    steps {
        sh '''
            # Pull Nikto Docker image
            docker pull sickhub/nikto
            
            # Run Nikto scan
            docker run --rm sickhub/nikto \
                -h http://target-app.example.com \
                -o nikto-report.xml \
                -Format xml
            
            # Copy report from container
            docker cp nikto:/nikto-report.xml . || true
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'nikto-report.*', allowEmptyArchive: true
        }
    }
}
```

## Step 5: Container Security Scanning

### Add Trivy Container Scanner

```
groovy
// Add Trivy container security scanning
stage('Container Security Scan') {
    steps {
        sh '''
            # Install Trivy
            wget https://github.com/aquasecurity/trivy/releases/download/v0.44.0/trivy_0.44.0_Linux-64bit.tar.gz
            tar -xzf trivy_0.44.0_Linux-64bit.tar.gz
            mv trivy /usr/local/bin/
            
            # Build Docker image
            docker build -t myapp:latest .
            
            # Scan container image
            trivy image --format json --output trivy-image-report.json myapp:latest || true
            
            # Scan for vulnerabilities
            trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest || true
            
            # Scan filesystem
            trivy fs --security-checks vuln,config trivy-fs-report.json . || true
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'trivy-*.json', allowEmptyArchive: true
        }
        failure {
            emailext body: 'Container has critical vulnerabilities', 
                        subject: 'Container Security Alert', 
                        to: 'security@example.com'
        }
    }
}
```

### Add Dockerfile Security Analysis

```
groovy
// Add Dockerfile security check
stage('Dockerfile Security') {
    steps {
        sh '''
            # Install hadolint
            wget https://github.com/hadolint/hadolint/releases/download/v2.12.0/hadolint-Linux-x86_64
            mv hadolint-Linux-x86_64 /usr/local/bin/hadolint
            chmod +x /usr/local/bin/hadolint
            
            # Lint Dockerfile
            hadolint Dockerfile > hadolint-report.txt || true
            
            # Use Docker Bench Security
            docker pull docker/docker-bench-security
            docker run -it --net host --pid host --userns host \
                -v /var/run/docker.sock:/var/run/docker.sock \
                -v /usr/lib/systemd:/usr/lib/systemd \
                -v /etc:/etc \
                --label docker-bench-security \
                docker/docker-bench-security > docker-bench-report.txt || true
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: '*-report.*', allowEmptyArchive: true
            recordIssues(
                enabledForFailure: true,
                tools: [hadolint(pattern: 'hadolint-report.txt')]
            )
        }
    }
}
```

## Step 6: Complete Security Pipeline

```
groovy
// Complete Jenkinsfile with all security scans
pipeline {
    agent any
    
    environment {
        SONAR_HOST = 'http://sonarqube:9000'
        SONAR_TOKEN = credentials('sonarqube-token')
        DOCKER_REGISTRY = 'docker.io'
        IMAGE_NAME = 'myapp'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }
    
    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Application') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Static Code Analysis') {
            parallel {
                stage('SonarQube') {
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh '''
                                mvn sonar:sonar \
                                -Dsonar.projectKey=myapp \
                                -Dsonar.host.url=${SONAR_HOST} \
                                -Dsonar.login=${SONAR_TOKEN}
                            '''
                        }
                    }
                }
                
                stage('Bandit') {
                    steps {
                        sh 'pip install bandit && bandit -r . -f json -o bandit-report.json || true'
                    }
                }
                
                stage('Semgrep') {
                    steps {
                        sh 'pip install semgrep && semgrep --config=auto --json . > semgrep-report.json || true'
                    }
                }
            }
        }
        
        stage('Dependency Check') {
            steps {
                sh '''
                    # Download and run OWASP Dependency Check
                    wget -q https://github.com/jeremylong/DependencyCheck/releases/download/v7.4.1/dependency-check-7.4.1-release.zip
                    unzip -q dependency-check-7.4.1-release.zip
                    ./dependency-check/bin/dependency-check.sh \
                        --project "MyApp" \
                        --scan ./target \
                        --format XML \
                        --output dependency-check-report.xml
                '''
            }
        }
        
        stage('Quality Gates') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                """
            }
        }
        
        stage('Container Security Scan') {
            steps {
                sh """
                    # Install and run Trivy
                    wget -q https://github.com/aquasecurity/trivy/releases/download/v0.44.0/trivy_0.44.0_Linux-64bit.tar.gz
                    tar -xzf trivy_0.44.0_Linux-64bit.tar.gz
                    mv trivy /usr/local/bin/
                    
                    # Scan image
                    trivy image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE_NAME}:${IMAGE_TAG} || true
                    trivy image --format json --output trivy-report.json ${IMAGE_NAME}:${IMAGE_TAG} || true
                """
            }
        }
        
        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    """
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh '''
                    # Deploy to staging environment
                    kubectl set image deployment/myapp myapp=${IMAGE_NAME}:${IMAGE_TAG} -n staging
                    
                    # Wait for rollout
                    kubectl rollout status deployment/myapp -n staging
                '''
            }
        }
        
        stage('Dynamic Security Scan') {
            steps {
                sh '''
                    # Start application for testing
                    docker run -d --name test-app -p 8080:8080 ${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 30
                    
                    # Run OWASP ZAP
                    docker run -d --name zap owasp/zap2docker-stable zap.sh -daemon -port 8081
                    sleep 10
                    
                    # Spider scan
                    curl http://localhost:8081/JSON/spider/action/scan/?url=http://host.docker.internal:8080
                    sleep 60
                    
                    # Active scan
                    curl http://localhost:8081/JSON/ascan/action/scan/?url=http://host.docker.internal:8080
                    sleep 90
                    
                    # Generate report
                    curl http://localhost:8081/JSON/report/standard/report/ -o zap-report.json || true
                    
                    # Cleanup
                    docker stop test-app zap
                    docker rm test-app zap
                '''
            }
        }
        
        stage('Security Approval') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    input message: 'Approve production deployment?', ok: 'Deploy'
                }
            }
        }
        
        stage('Deploy to Production') {
            steps {
                sh '''
                    kubectl set image deployment/myapp myapp=${IMAGE_NAME}:${IMAGE_TAG} -n production
                    kubectl rollout status deployment/myapp -n production
                '''
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up...'
            sh 'docker system prune -f'
        }
        success {
            emailext body: 'Security pipeline completed successfully', 
                        subject: 'Security Pipeline Success', 
                        to: 'team@example.com'
        }
        failure {
            emailext body: 'Security pipeline failed. Check reports for details.', 
                        subject: 'Security Pipeline Failed', 
                        to: 'security@example.com'
        }
    }
}
```

## Step 7: Security Metrics Dashboard

```
groovy
// Create security metrics stage
stage('Security Metrics') {
    steps {
        sh '''
            # Create security dashboard data
            cat > security-metrics.json <<EOF
            {
                "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
                "sonarqube": {
                    "bugs": 0,
                    "vulnerabilities": 0,
                    "code_smells": 0,
                    "security_hotspots": 0
                },
                "trivy": {
                    "critical": 0,
                    "high": 0,
                    "medium": 0,
                    "low": 0
                },
                "dependency_check": {
                    "critical": 0,
                    "high": 0
                }
            }
            EOF
        '''
        
        // Publish metrics to monitoring system
        withCredentials([string(credentialsId: 'prometheus-token', variable: 'PROM_TOKEN')]) {
            sh '''
                curl -X POST http://prometheus:9090/api/v1/push \
                    -H "Authorization: Bearer ${PROM_TOKEN}" \
                    -d @security-metrics.json
            '''
        }
    }
}
```

## Step 8: Security Policies with OPA

### Create OPA Policy

```
rego
# OPA policy for container security
package container.security

deny[msg] {
    input.image.config.config.ExposedPorts != {}
    msg = "Container should not expose unnecessary ports"
}

deny[msg] {
    input.image.config.config.User == "root"
    msg = "Container should not run as root user"
}

deny[msg] {
    input.image.config.config.Env[_] == "ADMIN_PASSWORD=secret"
    msg = "Hardcoded passwords are not allowed"
}
```

### Integrate OPA in Pipeline

```
groovy
stage('Policy Check') {
    steps {
        sh '''
            # Install OPA
            wget https://openpolicyagent.org/downloads/latest/opa_linux_amd64_static
            mv opa_linux_amd64_static /usr/local/bin/opa
            chmod +x /usr/local/bin/opa
            
            # Pull image and extract config
            docker pull ${IMAGE_NAME}:${IMAGE_TAG}
            docker save ${IMAGE_NAME}:${IMAGE_TAG} -o image.tar
            tar -xf image.tar -O */layer.tar | tar -xf - -C /tmp layer.json
            
            # Evaluate policy
            opa eval --input /tmp/layer.json --format pretty "data.container.security.deny"
        '''
    }
}
```

## Conclusion

This project demonstrates:
- Setting up SonarQube for static analysis
- Integrating multiple SAST tools (SonarQube, Bandit, Semgrep)
- Adding OWASP Dependency Check for vulnerability scanning
- Implementing DAST with OWASP ZAP and Nikto
- Container security scanning with Trivy
- Dockerfile security analysis with Hadolint
- Complete security pipeline in Jenkins
- Security metrics and monitoring
- Policy enforcement with OPA

## Additional Resources
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [SonarQube Documentation](https://docs.sonarqube.org/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [OWASP ZAP Documentation](https://www.zaproxy.org/docs/)
