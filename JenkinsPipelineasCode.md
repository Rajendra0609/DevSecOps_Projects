# Jenkins Pipeline as Code — Complete Guide

## Project Overview

Pipeline as Code is the practice of defining your entire CI/CD pipeline inside a `Jenkinsfile` that lives in your source repository alongside your application code. This approach brings version control, code review, and auditability to your pipeline definitions — a fundamental practice in any mature DevSecOps environment.

### Objectives
1. Understand Declarative vs Scripted pipeline syntax
2. Master pipeline directives: agent, stages, environment, parameters, triggers, post
3. Implement parallel stages and matrix builds
4. Build reusable Shared Libraries
5. Set up Multibranch Pipelines
6. Integrate security scanning stages (SAST, DAST, SCA) in the pipeline
7. Handle secrets safely using Jenkins Credentials and HashiCorp Vault
8. Apply real-world pipeline patterns used in enterprise DevSecOps

---

## Step 1: Declarative vs Scripted Pipelines

Jenkins supports two pipeline syntaxes. In modern practice, **Declarative** is the standard for most teams because it is structured, readable, and enforces best practices. **Scripted** is available when you need Groovy-level control.

### Declarative Pipeline — Skeleton

```groovy
pipeline {
    agent any

    environment {
        APP_NAME  = 'myapp'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit tests')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Deployment target')
    }

    triggers {
        // Poll SCM every minute
        pollSCM('* * * * *')
        // Or use webhook (preferred) — configure in GitHub/GitLab to push events
        // githubPush()
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        retry(2)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                // sh "git log --oneline -5"
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests'
            }
        }

        stage('Test') {
            when {
                expression { return !params.SKIP_TESTS }
            }
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Deploy') {
            when {
                expression { return params.ENVIRONMENT != 'prod' || env.BRANCH_NAME == 'main' }
            }
            steps {
                echo "Deploying to ${params.ENVIRONMENT}"
            }
        }
    }

    post {
        success {
            echo "Build ${env.BUILD_NUMBER} succeeded."
            // slackSend channel: '#ci', message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            echo "Build ${env.BUILD_NUMBER} failed."
            // emailext subject: "FAILURE: ${env.JOB_NAME}", body: "Check logs at ${env.BUILD_URL}", to: 'team@example.com'
        }
        always {
            cleanWs()
        }
    }
}
```

### Scripted Pipeline — When to Use It

Scripted pipelines are written in raw Groovy inside a `node` block. Use them when you need dynamic logic that Declarative cannot express — for example, generating stages programmatically based on a list.

```groovy
// Scripted pipeline example
node('linux') {
    def services = ['auth-service', 'order-service', 'payment-service']

    stage('Checkout') {
        checkout scm
    }

    services.each { svc ->
        stage("Build ${svc}") {
            dir(svc) {
                sh "mvn clean package -DskipTests"
            }
        }
    }

    stage('Notify') {
        if (currentBuild.result == 'FAILURE') {
            mail to: 'team@example.com', subject: "Pipeline failed", body: "Check ${env.BUILD_URL}"
        }
    }
}
```

---

## Step 2: Pipeline Directives In Depth

### agent

The `agent` directive tells Jenkins where to run the pipeline or a specific stage. Setting `agent none` at the top level and specifying agents per stage gives you the most control.

```groovy
pipeline {
    // No global agent — each stage declares its own
    agent none

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args  '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Code Analysis') {
            agent {
                label 'sonarqube'
            }
            steps {
                sh 'mvn sonar:sonar'
            }
        }

        stage('Deploy to K8s') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['cat']
    tty: true
"""
                }
            }
            steps {
                container('kubectl') {
                    sh 'kubectl apply -f k8s/'
                }
            }
        }
    }
}
```

### environment

Environment variables can be set globally or per-stage. Use `credentials()` to safely inject secrets without exposing them in logs.

```groovy
pipeline {
    agent any

    environment {
        // Plain variables
        APP_NAME    = 'myapp'
        REGISTRY    = 'registry.example.com'

        // Inject credentials — these are masked in logs
        DOCKER_CREDS    = credentials('docker-hub-creds')       // sets DOCKER_CREDS_USR and DOCKER_CREDS_PSW
        SONAR_TOKEN     = credentials('sonarqube-token')         // single secret string
        AWS_CREDS       = credentials('aws-access-key')          // AWS_CREDS_ACCESS_KEY_ID + AWS_CREDS_SECRET_ACCESS_KEY
    }

    stages {
        stage('Docker Push') {
            steps {
                sh """
                    echo "${DOCKER_CREDS_PSW}" | docker login ${REGISTRY} -u ${DOCKER_CREDS_USR} --password-stdin
                    docker push ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                """
            }
        }
    }
}
```

### parameters

Parameters allow humans or upstream jobs to pass inputs at build time.

```groovy
parameters {
    string(name: 'IMAGE_TAG',     defaultValue: 'latest',  description: 'Docker image tag to deploy')
    booleanParam(name: 'DRY_RUN', defaultValue: true,      description: 'Perform a dry run only')
    choice(name: 'REGION',        choices: ['us-east-1', 'eu-west-1', 'ap-south-1'], description: 'AWS Region')
    password(name: 'DB_PASSWORD', defaultValue: '',         description: 'Database password (use credentials instead in prod)')
    text(name: 'RELEASE_NOTES',   defaultValue: '',         description: 'Release notes for this deployment')
}
```

### triggers

```groovy
triggers {
    // GitHub/GitLab webhook push (most common in real environments)
    githubPush()

    // Poll SCM as fallback if webhooks aren't configured
    pollSCM('H/5 * * * *')

    // Scheduled build (e.g. nightly)
    cron('H 2 * * *')

    // Upstream job trigger
    upstream(upstreamProjects: 'build-job,test-job', threshold: hudson.model.Result.SUCCESS)
}
```

### options

```groovy
options {
    timestamps()                                     // Prefix every line with a timestamp
    timeout(time: 45, unit: 'MINUTES')               // Kill the build if it exceeds this limit
    buildDiscarder(logRotator(numToKeepStr: '30',    // Retain last 30 builds
                              artifactNumToKeepStr: '5'))  // Keep artifacts for last 5 only
    disableConcurrentBuilds(abortPrevious: true)     // Cancel old run when a new one starts
    skipDefaultCheckout(true)                        // Manage checkout manually
    ansiColor('xterm')                               // Colored output in logs
    quietPeriod(5)                                   // Wait 5s before starting (absorbs rapid commits)
    retry(2)                                         // Retry the whole pipeline up to 2 times on failure
}
```

---

## Step 3: Parallel Stages and Matrix Builds

### Parallel Stages

Running independent stages in parallel dramatically reduces pipeline duration — a key optimization in real-world CI.

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Parallel Quality Gates') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test'
                    }
                    post {
                        always {
                            junit '**/target/surefire-reports/*.xml'
                        }
                    }
                }

                stage('Integration Tests') {
                    steps {
                        sh 'mvn verify -Pintegration-tests'
                    }
                }

                stage('SAST — SonarQube') {
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh 'mvn sonar:sonar'
                        }
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }

                stage('SCA — Dependency Check') {
                    steps {
                        sh 'mvn org.owasp:dependency-check-maven:check'
                        dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Package and Push') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
                sh 'docker push registry.example.com/myapp:${BUILD_NUMBER}'
            }
        }
    }
}
```

### Matrix Build (Multi-Platform / Multi-Version)

```groovy
pipeline {
    agent none

    stages {
        stage('Test Matrix') {
            matrix {
                agent { label "${PLATFORM}" }

                axes {
                    axis {
                        name 'JAVA_VERSION'
                        values '11', '17', '21'
                    }
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'windows'
                    }
                }

                excludes {
                    exclude {
                        axis { name 'JAVA_VERSION'; values '11' }
                        axis { name 'PLATFORM';     values 'windows' }
                    }
                }

                stages {
                    stage('Compile') {
                        steps {
                            sh "JAVA_HOME=/usr/lib/jvm/java-${JAVA_VERSION} mvn compile"
                        }
                    }
                    stage('Test') {
                        steps {
                            sh "JAVA_HOME=/usr/lib/jvm/java-${JAVA_VERSION} mvn test"
                        }
                    }
                }
            }
        }
    }
}
```

---

## Step 4: Error Handling and the `post` Block

```groovy
pipeline {
    agent any

    stages {
        stage('Risky Step') {
            steps {
                // catchError allows the pipeline to continue even if this step fails
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh 'run-flaky-test.sh'
                }
            }
        }

        stage('Always-critical Step') {
            steps {
                // retry wraps the block and retries on failure
                retry(3) {
                    sh 'curl -f https://api.example.com/health'
                }
            }
        }

        stage('Timeout Example') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    sh 'long-running-script.sh'
                }
            }
        }

        stage('Manual Approval Gate') {
            steps {
                input message: 'Approve deployment to production?',
                      ok: 'Deploy',
                      submitter: 'release-managers',
                      parameters: [
                          string(name: 'REASON', description: 'Reason for approval')
                      ]
            }
        }
    }

    post {
        always   { echo "Pipeline finished — cleaning up"; cleanWs() }
        success  { echo "SUCCESS — notifying team" }
        failure  { echo "FAILURE — creating incident ticket" }
        unstable { echo "UNSTABLE — flaky tests detected" }
        aborted  { echo "ABORTED — someone cancelled the build" }
        changed  {
            // Fires only when the result differs from the last build
            echo "Build result CHANGED — was: ${currentBuild.previousBuild?.result}"
        }
    }
}
```

---

## Step 5: Multibranch Pipelines

Multibranch Pipeline is the standard approach in enterprise environments. Jenkins automatically discovers branches and PRs in your repository and creates a pipeline for each one.

### How to Set Up

1. From the Jenkins main page, click **New Item → Multibranch Pipeline**
2. Under **Branch Sources**, add your GitHub/GitLab repository and configure credentials
3. Under **Build Configuration**, set the script path to `Jenkinsfile`
4. Set **Scan Multibranch Pipeline Triggers** to scan on webhook push or on a schedule

### Jenkinsfile for Multibranch

```groovy
pipeline {
    agent any

    environment {
        IS_MAIN    = "${env.BRANCH_NAME == 'main'}"
        IS_PR      = "${env.CHANGE_ID != null}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // Print branch context for audit
                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "PR: ${env.CHANGE_ID ?: 'N/A'}"
                    echo "Commit: ${env.GIT_COMMIT}"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            // Run full analysis only on main; PR analysis uses PR decoration
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    changeRequest()
                }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.pullrequest.key=${env.CHANGE_ID ?: ''} \
                          -Dsonar.pullrequest.branch=${env.CHANGE_BRANCH ?: env.BRANCH_NAME} \
                          -Dsonar.pullrequest.base=${env.CHANGE_TARGET ?: 'main'}
                    """
                }
            }
        }

        stage('Build Docker Image') {
            when { branch 'main' }
            steps {
                sh "docker build -t myapp:${env.BUILD_NUMBER} ."
            }
        }

        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                sh "kubectl set image deployment/myapp myapp=myapp:${env.BUILD_NUMBER} -n staging"
            }
        }

        stage('Deploy to Production') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                input message: "Deploy build ${env.BUILD_NUMBER} to production?", ok: "Deploy"
                sh "kubectl set image deployment/myapp myapp=myapp:${env.BUILD_NUMBER} -n production"
            }
        }
    }
}
```

---

## Step 6: Shared Libraries

Shared Libraries are the most powerful Jenkins feature for enterprise teams. They let you write reusable Groovy code — functions, pipeline steps, full templates — that any Jenkinsfile in your organisation can consume. This removes duplication, enforces standards, and lets platform teams push pipeline upgrades without touching every project.

### Shared Library Directory Structure

```
jenkins-shared-library/               ← your Git repository
├── vars/
│   ├── buildMavenApp.groovy           ← global variable / step
│   ├── deployToKubernetes.groovy
│   ├── runSecurityScan.groovy
│   └── notifySlack.groovy
├── src/
│   └── com/
│       └── myorg/
│           └── pipeline/
│               ├── MavenBuilder.groovy
│               └── DockerPublisher.groovy
├── resources/
│   └── com/myorg/pipeline/
│       └── helm-values-template.yaml
└── Jenkinsfile                        ← optional: test the library itself
```

### Registering the Library in Jenkins

Go to **Manage Jenkins → System → Global Pipeline Libraries** and add:
- **Name**: `jenkins-shared-lib`
- **Default version**: `main`
- **Retrieval method**: Modern SCM → Git → your repo URL

### Writing a Shared Step (`vars/buildMavenApp.groovy`)

```groovy
// vars/buildMavenApp.groovy
// Callable as: buildMavenApp(goal: 'package', skipTests: false)

def call(Map config = [:]) {
    def goal      = config.goal      ?: 'package'
    def skipTests = config.skipTests ?: false
    def opts      = skipTests ? '-DskipTests' : ''

    stage("Maven ${goal.capitalize()}") {
        sh "mvn clean ${goal} ${opts}"
    }
}
```

### Writing a Shared Library Class (`src/com/myorg/pipeline/DockerPublisher.groovy`)

```groovy
// src/com/myorg/pipeline/DockerPublisher.groovy
package com.myorg.pipeline

class DockerPublisher implements Serializable {
    private def script
    private String registry
    private String imageName

    DockerPublisher(script, String registry, String imageName) {
        this.script    = script
        this.registry  = registry
        this.imageName = imageName
    }

    void build(String tag) {
        script.sh "docker build -t ${registry}/${imageName}:${tag} ."
    }

    void push(String tag) {
        script.withCredentials([script.usernamePassword(
            credentialsId: 'docker-hub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            script.sh "echo \$DOCKER_PASS | docker login ${registry} -u \$DOCKER_USER --password-stdin"
            script.sh "docker push ${registry}/${imageName}:${tag}"
        }
    }
}
```

### Consuming the Shared Library in a Jenkinsfile

```groovy
// @Library annotation loads the library by name registered in Jenkins
@Library('jenkins-shared-lib@main') _
import com.myorg.pipeline.DockerPublisher

pipeline {
    agent any

    environment {
        REGISTRY  = 'registry.example.com'
        APP_NAME  = 'myapp'
    }

    stages {
        stage('Build') {
            steps {
                // Using the global var step
                buildMavenApp(goal: 'package', skipTests: false)
            }
        }

        stage('Security Scan') {
            steps {
                runSecurityScan(tool: 'trivy', image: "${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}")
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    def publisher = new DockerPublisher(this, env.REGISTRY, env.APP_NAME)
                    publisher.build(env.BUILD_NUMBER)
                    publisher.push(env.BUILD_NUMBER)
                    publisher.push('latest')
                }
            }
        }
    }
}
```

### Shared Library: Security Scan Step (`vars/runSecurityScan.groovy`)

```groovy
// vars/runSecurityScan.groovy
def call(Map config = [:]) {
    def tool  = config.tool  ?: 'trivy'
    def image = config.image ?: ''

    stage("Security Scan — ${tool.toUpperCase()}") {
        if (tool == 'trivy') {
            sh """
                trivy image \
                  --exit-code 1 \
                  --severity HIGH,CRITICAL \
                  --format template \
                  --template '@/usr/local/share/trivy/templates/html.tpl' \
                  -o trivy-report.html \
                  ${image}
            """
            publishHTML([
                allowMissing: false,
                reportDir:    '.',
                reportFiles:  'trivy-report.html',
                reportName:   'Trivy Security Report'
            ])
        } else if (tool == 'snyk') {
            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                sh "snyk container test ${image} --severity-threshold=high --json > snyk-report.json || true"
            }
        }
    }
}
```

---

## Step 7: DevSecOps Pipeline — Full Example

This is a production-grade pipeline that integrates the security scanning stages expected in any mature DevSecOps organisation.

```groovy
@Library('jenkins-shared-lib@main') _

pipeline {
    agent none

    environment {
        REGISTRY     = 'registry.example.com'
        APP_NAME     = 'sysfoo'
        IMAGE        = "${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
        SONAR_URL    = 'http://sonarqube:9000'
        KUBE_NS_STG  = 'staging'
        KUBE_NS_PROD = 'production'
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds(abortPrevious: true)
    }

    stages {

        // ─── SOURCE ───────────────────────────────────────────────────────────
        stage('Checkout') {
            agent { label 'linux' }
            steps {
                checkout scm
                script {
                    env.GIT_AUTHOR  = sh(script: 'git log -1 --format="%an"', returnStdout: true).trim()
                    env.GIT_MESSAGE = sh(script: 'git log -1 --format="%s"',  returnStdout: true).trim()
                }
                echo "Commit by ${env.GIT_AUTHOR}: ${env.GIT_MESSAGE}"
            }
        }

        // ─── BUILD & UNIT TEST ────────────────────────────────────────────────
        stage('Build & Unit Test') {
            agent {
                docker { image 'maven:3.9-eclipse-temurin-17'; args '-v $HOME/.m2:/root/.m2' }
            }
            steps {
                sh 'mvn clean package'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
            post {
                always { junit '**/target/surefire-reports/*.xml' }
            }
        }

        // ─── QUALITY GATES (parallel) ─────────────────────────────────────────
        stage('Quality Gates') {
            parallel {

                stage('SAST — SonarQube') {
                    agent { docker { image 'maven:3.9-eclipse-temurin-17' } }
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh 'mvn sonar:sonar -Dsonar.projectKey=${APP_NAME}'
                        }
                        timeout(time: 10, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }

                stage('SCA — OWASP Dependency Check') {
                    agent { docker { image 'maven:3.9-eclipse-temurin-17' } }
                    steps {
                        sh 'mvn org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7'
                    }
                    post {
                        always {
                            dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                        }
                    }
                }

                stage('Secret Scan — Gitleaks') {
                    agent { label 'linux' }
                    steps {
                        sh 'gitleaks detect --source . --report-format json --report-path gitleaks-report.json'
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                        }
                    }
                }

            }
        }

        // ─── CONTAINER BUILD & SCAN ───────────────────────────────────────────
        stage('Docker Build') {
            agent { label 'docker' }
            steps {
                sh "docker build -t ${IMAGE} ."
            }
        }

        stage('Container Scan — Trivy') {
            agent { label 'docker' }
            steps {
                sh """
                    trivy image \
                      --exit-code 1 \
                      --severity HIGH,CRITICAL \
                      --format json \
                      -o trivy-results.json \
                      ${IMAGE}
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-results.json'
                }
            }
        }

        stage('Docker Push') {
            agent { label 'docker' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry-creds',
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS'
                )]) {
                    sh "echo \$REG_PASS | docker login ${REGISTRY} -u \$REG_USER --password-stdin"
                    sh "docker push ${IMAGE}"
                    sh "docker tag ${IMAGE} ${REGISTRY}/${APP_NAME}:latest"
                    sh "docker push ${REGISTRY}/${APP_NAME}:latest"
                }
            }
        }

        // ─── DEPLOY TO STAGING ────────────────────────────────────────────────
        stage('Deploy to Staging') {
            agent { label 'kubernetes' }
            when { branch 'main' }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-staging']) {
                    sh "helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                          --namespace ${KUBE_NS_STG} \
                          --set image.tag=${BUILD_NUMBER} \
                          --wait --timeout 5m"
                }
            }
        }

        // ─── DAST — OWASP ZAP ────────────────────────────────────────────────
        stage('DAST — OWASP ZAP') {
            agent { label 'linux' }
            when { branch 'main' }
            steps {
                sh """
                    docker run --rm \
                      -v \$(pwd)/zap-reports:/zap/wrk:rw \
                      ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                      -t http://staging.example.com \
                      -r zap-report.html \
                      -I
                """
            }
            post {
                always {
                    publishHTML([
                        reportDir:   'zap-reports',
                        reportFiles: 'zap-report.html',
                        reportName:  'ZAP DAST Report'
                    ])
                }
            }
        }

        // ─── PRODUCTION GATE ─────────────────────────────────────────────────
        stage('Approve Production Deploy') {
            when { branch 'main' }
            steps {
                timeout(time: 24, unit: 'HOURS') {
                    input message: "Promote build ${BUILD_NUMBER} to production?",
                          ok: 'Approve',
                          submitter: 'release-managers'
                }
            }
        }

        stage('Deploy to Production') {
            agent { label 'kubernetes' }
            when { branch 'main' }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                    sh "helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                          --namespace ${KUBE_NS_PROD} \
                          --set image.tag=${BUILD_NUMBER} \
                          --wait --timeout 10m"
                }
            }
        }

    }

    post {
        success {
            notifySlack(channel: '#deployments', message: "✅ ${APP_NAME} #${BUILD_NUMBER} deployed to prod by ${env.GIT_AUTHOR}")
        }
        failure {
            notifySlack(channel: '#ci-failures',  message: "❌ ${APP_NAME} #${BUILD_NUMBER} FAILED — ${env.BUILD_URL}")
        }
        always {
            cleanWs()
        }
    }
}
```

---

## Step 8: Secrets Management — Jenkins Credentials + HashiCorp Vault

### Jenkins Built-in Credentials

Always use Jenkins Credentials to inject secrets — never hardcode or print them.

```groovy
pipeline {
    agent any

    stages {
        stage('Use Credentials') {
            steps {
                // Username + Password
                withCredentials([usernamePassword(
                    credentialsId: 'db-credentials',
                    usernameVariable: 'DB_USER',
                    passwordVariable: 'DB_PASS'
                )]) {
                    sh 'psql -U $DB_USER -h db.example.com mydb'
                }

                // Secret text (API token, etc.)
                withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                    sh 'curl -H "Authorization: token $GH_TOKEN" https://api.github.com/user'
                }

                // SSH private key
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'deploy-key',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh 'ssh -i $SSH_KEY $SSH_USER@prod.example.com "systemctl restart myapp"'
                }

                // Certificate / file
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh 'kubectl --kubeconfig=$KUBECONFIG_FILE get pods'
                }
            }
        }
    }
}
```

### HashiCorp Vault Integration

In enterprise environments, secrets are typically stored in Vault rather than Jenkins. The HashiCorp Vault Plugin lets pipelines fetch secrets at runtime.

```groovy
// Install: HashiCorp Vault Plugin
// Configure Vault in: Manage Jenkins → System → Vault Plugin Config
// Vault URL: https://vault.example.com
// Vault Credential: Vault Token or AppRole

pipeline {
    agent any

    environment {
        VAULT_ADDR = 'https://vault.example.com'
    }

    stages {
        stage('Fetch Secrets from Vault') {
            steps {
                withVault(
                    configuration: [
                        vaultUrl:              'https://vault.example.com',
                        vaultCredentialId:     'vault-approle',
                        engineVersion:         2
                    ],
                    vaultSecrets: [
                        [
                            path:     'secret/myapp/db',
                            engineVersion: 2,
                            secretValues: [
                                [vaultKey: 'username', envVar: 'DB_USER'],
                                [vaultKey: 'password', envVar: 'DB_PASS']
                            ]
                        ],
                        [
                            path:     'secret/myapp/api',
                            engineVersion: 2,
                            secretValues: [
                                [vaultKey: 'token', envVar: 'API_TOKEN']
                            ]
                        ]
                    ]
                ) {
                    // DB_USER, DB_PASS, API_TOKEN are available here and masked in logs
                    sh 'deploy.sh --db-user=$DB_USER --api-token=$API_TOKEN'
                }
            }
        }
    }
}
```

---

## Step 9: Pipeline Best Practices

- **Store Jenkinsfiles in source control** alongside your application code. Never create pipeline logic in the Jenkins UI.
- **Use Shared Libraries** to centralise common logic. Treat your pipeline library as a first-class software product with its own tests and versioning.
- **Pin agent images** to a specific digest or version tag (not `:latest`) to ensure reproducible builds.
- **Never print secrets** — even a `sh 'echo $SECRET'` will expose credentials in logs. Use `withCredentials` and let Jenkins mask them.
- **Set pipeline timeouts** at both the pipeline and stage level to prevent hung builds from consuming executors indefinitely.
- **Use `cleanWs()`** in the `post { always {} }` block to free disk space on agents between builds.
- **Run security scans in parallel** with tests to keep pipeline duration low.
- **Require a manual approval gate** before production deployments. Combine with a `timeout` so unclaimed approvals don't block the system.
- **Archive test results and security reports** as Jenkins artifacts and HTML reports for every build — auditors need this evidence.
- **Set `numToKeepStr`** on all jobs. Retaining unlimited builds fills disks and slows the Jenkins UI.
- **Set master/controller executors to 0**. The controller should only orchestrate — all build work should run on agents.

---

## Interview Questions & Answers — Pipeline as Code

### Q1: What is the difference between Declarative and Scripted pipelines?

**A:** Declarative is a structured, opinionated DSL that enforces a specific schema (`pipeline { agent {} stages {} post {} }`). It is easier to read, supports linting (`jenkins-pipeline-linter`), and is the recommended default. Scripted is raw Groovy inside a `node {}` block — it gives you full programming flexibility (loops, conditionals, dynamic stage generation) but at the cost of readability and enforceability. In practice, most teams use Declarative and drop into `script {}` blocks only when they need logic that Declarative cannot express.

### Q2: How do you prevent a secret from being printed in Jenkins logs?

**A:** Use the `withCredentials` step. Jenkins masks the secret values in console output by replacing them with `****`. Never assign secrets to environment variables declared outside `withCredentials`, and never run `sh "echo $SECRET"`. For an extra layer, you can enable the Mask Passwords plugin to catch edge cases.

### Q3: What is a Shared Library and why would you use one?

**A:** A Shared Library is a Git repository of Groovy code (under `vars/` for global steps and `src/` for classes) that multiple Jenkinsfiles can import using `@Library`. It allows platform teams to define standard pipeline patterns — build steps, notification logic, security scan wrappers — once, and have every project consume them without copy-pasting. Updates to the library are picked up by all pipelines automatically (or on a version tag, if you pin the version in `@Library('lib@v1.2.0')`).

### Q4: What is a Multibranch Pipeline and how does it differ from a regular Pipeline job?

**A:** A regular Pipeline job is tied to a single branch. A Multibranch Pipeline job discovers all branches and pull requests in a repository and automatically creates child pipeline jobs for each one, each running the `Jenkinsfile` from that branch. This is the standard approach for feature branch workflows and PR validation. When a branch is deleted, Jenkins automatically cleans up the corresponding job.

### Q5: How would you implement a Blue-Green deployment in a Jenkins pipeline?

**A:** A Blue-Green deployment maintains two identical production environments (Blue = current, Green = new). The pipeline would: (1) build and deploy the new version to the Green environment, (2) run smoke tests against Green, (3) if tests pass, switch the load balancer to point at Green (via a DNS change, an ingress annotation update in Kubernetes, or an ALB target group swap in AWS), (4) keep Blue running for a defined rollback window. The `input` step provides the manual approval gate before the switch, and the pipeline can automatically revert the load balancer if post-switch health checks fail.

### Q6: How do you run parallel stages only when certain conditions are met?

**A:** Combine `parallel` with `when` inside each branch:

```groovy
stage('Parallel Scans') {
    parallel {
        stage('Trivy') {
            when { branch 'main' }
            steps { sh 'trivy image myapp:latest' }
        }
        stage('ZAP') {
            when { not { changeRequest() } }
            steps { sh 'zap-baseline.py ...' }
        }
    }
}
```

### Q7: Your pipeline is failing because a `sh` step exits with a non-zero code but you want the pipeline to continue. How do you handle this?

**A:** Three options: (1) append `|| true` to the shell command to suppress the exit code — simple but hides failures; (2) use `catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE')` — this marks the stage as failed and the build as unstable but continues execution; (3) use `script { def result = sh(script: 'cmd', returnStatus: true); if (result != 0) { ... } }` — this gives you full control over the response logic.

### Q8: What is the `when` directive and what conditions does it support?

**A:** The `when` directive controls whether a stage executes. Supported conditions include: `branch` (matches branch name pattern), `tag` (matches a Git tag), `changeRequest` (PR builds), `environment` (checks an env variable), `expression` (arbitrary Groovy expression), `buildingTag`, `triggeredBy`, `not`, `allOf`, `anyOf`. Conditions can be nested using `allOf` and `anyOf` for complex logic.

### Q9: How do you pass data between stages in a Declarative pipeline?

**A:** Environment variables are the primary mechanism — set them in `environment {}` or write to `env.MY_VAR` inside a `script {}` block. For files, `stash` / `unstash` lets you transfer workspace files between agents in a pipeline. For structured data, write to a file (e.g., JSON), archive it, and read it in a later stage.

### Q10: How would you implement pipeline-as-code across 200 microservices in an organisation?

**A:** Use a Shared Library to define a standardised pipeline template. Each microservice's `Jenkinsfile` is then just a few lines — it imports the library and calls the template function, passing service-specific overrides. All 200 services get consistent build, test, scan, and deploy behaviour. Platform teams can update security policies centrally. Use Multibranch Pipelines with a GitHub Organisation folder job so Jenkins automatically discovers every repository and branch. Use JCasC to codify the Jenkins controller configuration itself so the entire setup is reproducible from Git.
