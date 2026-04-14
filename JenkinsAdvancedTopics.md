# Jenkins Advanced Topics — Complete Reference for Senior DevSecOps Engineers

## Overview

This file covers every advanced Jenkins topic that a 5+ year DevSecOps engineer must know — for real-world production work and for senior-level interviews. Each section includes practical working code, real-world context, and dedicated interview Q&A.

### Topics Covered
1. Jenkins REST API and CLI Automation
2. Job DSL Plugin — Programmatic Job Generation
3. Jenkinsfile Linting and Validation
4. Pipeline Unit Testing with JenkinsPipelineUnit
5. Jenkins vs GitHub Actions vs GitLab CI — Comparison
6. Groovy Sandbox and Script Approval
7. Blue-Green Deployment Implementation
8. Canary Deployment Implementation
9. Rollback Strategy Patterns
10. Notification Integrations — Slack, Microsoft Teams, PagerDuty
11. Docker Registry Integration — ECR, ACR, Harbor
12. Container Image Signing with Cosign
13. SBOM Generation — Syft and CycloneDX
14. Terraform Pipeline Integration
15. Ansible Pipeline Integration
16. Pipeline Caching and Performance Optimisation
17. Common Jenkins Anti-Patterns
18. Multi-Stage Docker Builds in Pipelines
19. Folder and View Organisation at Scale
20. Windows Agent Specifics
21. Advanced Build Parameterisation
22. Jenkins in a GitOps Workflow (ArgoCD/Flux)
23. Build Status Badges and README Integration
24. Plugin Dependency Management
25. Jenkins with OpenTelemetry
26. Throttle Concurrent Builds and Lockable Resources
27. Parameterised Trigger Plugin
28. Master Interview Q&A — All Topics

---

## Topic 1: Jenkins REST API and CLI Automation

### Why This Matters
In real-world DevSecOps environments, Jenkins is never operated manually at scale. You will script build triggers from GitOps tools, query build status from deployment scripts, and manage Jenkins from CI pipelines that provision infrastructure. Every senior Jenkins interview includes at least one REST API question.

### Authentication for API Calls

```bash
# Always use an API token — never your password
# Generate at: http://jenkins.example.com/user/<username>/configure → API Token

JENKINS_URL="http://jenkins.example.com"
JENKINS_USER="admin"
JENKINS_TOKEN="your-api-token-here"

# Basic auth format
AUTH="${JENKINS_USER}:${JENKINS_TOKEN}"

# Test connectivity
curl -s -u "${AUTH}" "${JENKINS_URL}/api/json?pretty=true"
```

### Triggering Builds

```bash
# Trigger a simple job (no parameters)
curl -s -X POST \
  -u "${AUTH}" \
  "${JENKINS_URL}/job/myfolder/job/build-job/build"

# Trigger a parameterised build
curl -s -X POST \
  -u "${AUTH}" \
  "${JENKINS_URL}/job/myfolder/job/deploy-job/buildWithParameters" \
  --data "ENVIRONMENT=staging&IMAGE_TAG=v1.2.3&REGION=us-east-1"

# Trigger with a JSON payload (alternative)
curl -s -X POST \
  -u "${AUTH}" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  "${JENKINS_URL}/job/deploy-job/buildWithParameters" \
  --data-urlencode "json={\"parameter\": [{\"name\":\"ENV\", \"value\":\"prod\"}]}"

# Get the crumb (required when CSRF protection is enabled)
CRUMB=$(curl -s -u "${AUTH}" \
  "${JENKINS_URL}/crumbIssuer/api/json" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['crumbRequestField']+':'+d['crumb'])")

# Use the crumb in the trigger
curl -s -X POST \
  -u "${AUTH}" \
  -H "${CRUMB}" \
  "${JENKINS_URL}/job/my-pipeline/build"
```

### Querying Build Status

```bash
# Get the last build status of a job
curl -s -u "${AUTH}" \
  "${JENKINS_URL}/job/my-pipeline/lastBuild/api/json?pretty=true"

# Get a specific build number
curl -s -u "${AUTH}" \
  "${JENKINS_URL}/job/my-pipeline/42/api/json?pretty=true" | \
  python3 -c "import sys,json; b=json.load(sys.stdin); print('Result:', b['result'], '| Duration:', b['duration']//1000, 's')"

# Poll until a build completes (useful in deploy scripts)
JOB="my-pipeline"
BUILD_NUM=42

while true; do
  STATUS=$(curl -s -u "${AUTH}" \
    "${JENKINS_URL}/job/${JOB}/${BUILD_NUM}/api/json" | \
    python3 -c "import sys,json; b=json.load(sys.stdin); print(b.get('result','IN_PROGRESS'))")

  echo "Build ${BUILD_NUM}: ${STATUS}"

  if [ "${STATUS}" != "null" ] && [ "${STATUS}" != "IN_PROGRESS" ]; then
    echo "Build finished with: ${STATUS}"
    break
  fi
  sleep 10
done

# Exit with error if build failed
[ "${STATUS}" = "SUCCESS" ] || exit 1
```

### Streaming Console Output

```bash
# Stream the console log of a running or completed build
curl -s -u "${AUTH}" \
  "${JENKINS_URL}/job/my-pipeline/lastBuild/consoleText"

# Progressive console output (follows the build as it runs)
START=0
while true; do
  RESPONSE=$(curl -s -u "${AUTH}" \
    "${JENKINS_URL}/job/my-pipeline/lastBuild/logText/progressiveText?start=${START}")
  echo "${RESPONSE}"
  X_TEXT_SIZE=$(curl -sI -u "${AUTH}" \
    "${JENKINS_URL}/job/my-pipeline/lastBuild/logText/progressiveText?start=${START}" | \
    grep -i "X-Text-Size" | awk '{print $2}' | tr -d '\r')
  if [ "${START}" = "${X_TEXT_SIZE}" ]; then break; fi
  START="${X_TEXT_SIZE}"
  sleep 2
done
```

### Managing Jobs via REST API

```bash
# Get list of all jobs in a folder
curl -s -u "${AUTH}" \
  "${JENKINS_URL}/job/myfolder/api/json?tree=jobs[name,url,color]" | \
  python3 -c "import sys,json; [print(j['name'], j['color']) for j in json.load(sys.stdin)['jobs']]"

# Disable a job
curl -s -X POST -u "${AUTH}" \
  "${JENKINS_URL}/job/myfolder/job/old-job/disable"

# Enable a job
curl -s -X POST -u "${AUTH}" \
  "${JENKINS_URL}/job/myfolder/job/old-job/enable"

# Delete a job
curl -s -X POST -u "${AUTH}" \
  "${JENKINS_URL}/job/myfolder/job/old-job/doDelete"

# Create a job from an XML config file
curl -s -X POST \
  -u "${AUTH}" \
  -H "Content-Type: application/xml" \
  "${JENKINS_URL}/createItem?name=new-pipeline-job" \
  --data-binary @job-config.xml

# Get the XML config of an existing job (useful for templating)
curl -s -u "${AUTH}" \
  "${JENKINS_URL}/job/existing-job/config.xml" > job-config.xml
```

### Jenkins CLI (`jenkins-cli.jar`)

```bash
# Download the CLI jar from your Jenkins instance
curl -s -O "${JENKINS_URL}/jnlpJars/jenkins-cli.jar"

# Alias for convenience
alias jcli="java -jar jenkins-cli.jar -s ${JENKINS_URL} -auth ${AUTH}"

# Common CLI commands
jcli list-jobs                            # List all jobs
jcli list-plugins                         # List installed plugins
jcli install-plugin git workflow-aggregator   # Install plugins
jcli safe-restart                         # Graceful restart
jcli quiet-down                           # Stop taking new builds (drain queue)
jcli cancel-quiet-down                    # Resume taking builds
jcli build my-pipeline -p ENV=staging -s  # Build and wait (-s = synchronous)
jcli get-job my-pipeline                  # Print job XML config
jcli delete-job old-job                   # Delete a job
jcli groovy = < myscript.groovy           # Run a Groovy script on the server

# Reload configuration from disk (useful after JCasC change)
jcli reload-configuration

# Create a job from XML
jcli create-job new-job < config.xml

# Export all job configs (backup script)
for job in $(jcli list-jobs); do
  jcli get-job "$job" > "backup/${job}.xml"
  echo "Exported: $job"
done
```

### Automating Jenkins in a Deployment Pipeline

```bash
#!/bin/bash
# deploy.sh — called from GitHub Actions or another CI tool
# Triggers a Jenkins deploy job and waits for result

set -euo pipefail

JENKINS_URL="${JENKINS_URL}"
AUTH="${JENKINS_USER}:${JENKINS_TOKEN}"
JOB_NAME="deploy-to-production"
IMAGE_TAG="${1:-latest}"

echo "Triggering Jenkins job: ${JOB_NAME} with IMAGE_TAG=${IMAGE_TAG}"

# Trigger and capture the queue location from the Location header
LOCATION=$(curl -si -X POST \
  -u "${AUTH}" \
  "${JENKINS_URL}/job/${JOB_NAME}/buildWithParameters?IMAGE_TAG=${IMAGE_TAG}" \
  | grep -i "^Location" | awk '{print $2}' | tr -d '\r')

echo "Queued at: ${LOCATION}"

# Wait for the build to leave the queue and get a build number
BUILD_URL=""
for i in $(seq 1 30); do
  BUILD_URL=$(curl -s -u "${AUTH}" "${LOCATION}api/json" | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('executable',{}).get('url',''))")
  [ -n "${BUILD_URL}" ] && break
  echo "Waiting for build to start... (${i}/30)"
  sleep 5
done

[ -z "${BUILD_URL}" ] && { echo "ERROR: Build never started"; exit 1; }
echo "Build started: ${BUILD_URL}"

# Poll for completion
while true; do
  RESULT=$(curl -s -u "${AUTH}" "${BUILD_URL}api/json" | \
    python3 -c "import sys,json; print(json.load(sys.stdin).get('result') or 'RUNNING')")
  echo "Status: ${RESULT}"
  [ "${RESULT}" = "RUNNING" ] || break
  sleep 15
done

[ "${RESULT}" = "SUCCESS" ] && echo "Deployment succeeded!" || { echo "Deployment FAILED!"; exit 1; }
```

### Interview Q&A — REST API and CLI

**Q: How do you trigger a Jenkins build from a script and wait for the result?**
A: Use the `/buildWithParameters` endpoint via `curl` with `-u user:token` auth. Capture the `Location` header from the 201 response — it points to the queue item URL. Poll the queue URL until the `executable.url` field appears (that's the build URL). Then poll `<build-url>/api/json` checking the `result` field — it is `null` while running and `SUCCESS`/`FAILURE`/`ABORTED` when done. This pattern is the standard way to integrate Jenkins into deployment scripts or GitOps pipelines.

**Q: What is the Jenkins crumb and why is it needed?**
A: The crumb is a CSRF token. Jenkins generates one per session and requires you to send it in the `Jenkins-Crumb` header when making mutating API calls (POST requests that change state). Without it, Jenkins rejects the request with a 403. You fetch the crumb from `/crumbIssuer/api/json` using your credentials, then include it in subsequent POST requests.

**Q: What is the difference between `build` and `buildWithParameters` endpoints?**
A: `build` triggers a job with no parameters or uses all default parameter values. `buildWithParameters` lets you pass named parameters as query string or form-encoded key-value pairs. If a job has required parameters and you call `build` instead of `buildWithParameters`, the build may fail or use defaults depending on configuration.

---

## Topic 2: Job DSL Plugin — Programmatic Job Generation

### Why This Matters
In any organisation with more than 20–30 jobs, managing them through the Jenkins UI is unscalable, error-prone, and not auditable. The Job DSL plugin lets you write Groovy scripts that generate any number of jobs from a template, stored in Git like any other code.

### How Job DSL Works
A "seed job" is a special Jenkins job that runs a Job DSL Groovy script. The script calls the DSL API to create, update, or delete other Jenkins jobs. The seed job is run whenever the DSL script changes, keeping all generated jobs in sync with their template.

### Basic Job DSL Script

```groovy
// jobs/seed.groovy — run by the seed job

// Generate a pipeline job for each microservice
def services = [
    'auth-service',
    'order-service',
    'payment-service',
    'notification-service',
    'api-gateway'
]

services.each { service ->
    pipelineJob("microservices/${service}-pipeline") {
        description("CI/CD Pipeline for ${service}")

        // Discard old builds
        logRotator {
            numToKeep(20)
            artifactNumToKeep(5)
        }

        // Parameters
        parameters {
            stringParam('IMAGE_TAG', 'latest', 'Docker image tag to deploy')
            choiceParam('ENVIRONMENT', ['dev', 'staging', 'prod'], 'Target environment')
            booleanParam('SKIP_TESTS', false, 'Skip unit tests')
        }

        // Trigger on GitHub push
        triggers {
            githubPush()
        }

        // Pipeline definition from the service's own Jenkinsfile
        definition {
            cpsScm {
                scm {
                    git {
                        remote {
                            url("https://github.com/myorg/${service}.git")
                            credentials('github-creds')
                        }
                        branch('*/main')
                    }
                }
                scriptPath('Jenkinsfile')
                lightweight(true)
            }
        }
    }
}

println "Generated ${services.size()} pipeline jobs"
```

### Generating Multibranch Pipelines

```groovy
// jobs/multibranch-seed.groovy

def repos = [
    [name: 'sysfoo',       org: 'myorg', description: 'Java web application'],
    [name: 'data-pipeline', org: 'myorg', description: 'ETL data pipeline'],
    [name: 'ml-service',   org: 'ml-team', description: 'ML inference service']
]

repos.each { repo ->
    multibranchPipelineJob("projects/${repo.name}") {
        description(repo.description)

        branchSources {
            github {
                id("${repo.name}-source")
                repoOwner(repo.org)
                repository(repo.name)
                credentialsId('github-creds')
                buildForkPRHead(false)
                buildForkPRMerge(true)
                buildOriginBranch(true)
                buildOriginBranchWithPR(false)
                buildOriginPRHead(false)
                buildOriginPRMerge(true)
            }
        }

        // Only build branches with a Jenkinsfile
        factory {
            workflowBranchProjectFactory {
                scriptPath('Jenkinsfile')
            }
        }

        // Scan for new branches every hour
        triggers {
            periodic(60)
        }

        orphanedItemStrategy {
            discardOldItems {
                numToKeep(10)
                daysToKeep(30)
            }
        }
    }
}
```

### Generating Freestyle Jobs

```groovy
// Freestyle job with upstream/downstream chain
['build', 'test', 'package'].eachWithIndex { stage, index ->
    freeStyleJob("legacy-app/${stage}") {
        description("${stage.capitalize()} stage for legacy app")

        scm {
            git {
                remote {
                    url('https://github.com/myorg/legacy-app.git')
                    credentials('github-creds')
                }
                branch('*/main')
            }
        }

        // Maven build step
        steps {
            maven {
                goals(stage == 'build' ? 'compile' : stage == 'test' ? 'test' : 'package -DskipTests')
                mavenInstallation('Maven-3.9')
            }
        }

        // Chain downstream job
        publishers {
            if (index < 2) {
                downstream("legacy-app/${['build','test','package'][index+1]}", 'SUCCESS')
            }
            if (stage == 'package') {
                archiveArtifacts {
                    pattern('target/*.jar')
                    fingerprint(true)
                }
            }
        }
    }
}
```

### Seed Job Configuration

```groovy
// The seed job itself is usually created manually once, or via JCasC:
// JCasC seed job definition
jobs:
  - script: >
      freeStyleJob('_seed-job') {
        description('Regenerates all pipeline jobs from DSL scripts in Git')
        scm {
          git {
            remote { url('https://github.com/myorg/jenkins-jobs.git'); credentials('github-creds') }
            branch('*/main')
          }
        }
        triggers { githubPush() }
        steps {
          jobDsl {
            targets('jobs/**/*.groovy')
            removedJobAction('DELETE')
            removedViewAction('DELETE')
            lookupStrategy('SEED_JOB')
            sandbox(true)
          }
        }
      }
```

### Interview Q&A — Job DSL

**Q: What is the difference between Job DSL and JCasC?**
A: JCasC (Jenkins Configuration as Code) manages the Jenkins system configuration — security realm, authorisation, credentials, global tools, cloud settings. Job DSL manages job/pipeline definitions. They complement each other: JCasC bootstraps and configures the Jenkins controller, Job DSL generates all the jobs. Together, an entire Jenkins environment — from zero — can be reproduced from Git with no manual UI interaction.

**Q: What happens to existing jobs when you update a Job DSL script?**
A: The seed job re-runs and the DSL engine reconciles the diff. Jobs that are still defined in the script are updated in place. Jobs that were previously generated but are no longer in the script can be: ignored, disabled, or deleted — controlled by the `removedJobAction` setting (`IGNORE`, `DISABLE`, or `DELETE`). In production, start with `DISABLE` so you don't accidentally delete jobs.

**Q: Can Job DSL scripts be tested before running the seed job?**
A: Yes. The Job DSL testing framework (`job-dsl-core`) can run as a unit test suite. You write Groovy/Spock tests that execute your DSL scripts against a mock Jenkins context and assert on the generated XML configuration. This lets you catch DSL errors in a local build before they affect the production Jenkins instance.

---

## Topic 3: Jenkinsfile Linting and Validation

### Why This Matters
A syntax error in a Jenkinsfile means the pipeline fails immediately without running any stages — wasting build minutes and blocking the team. Catching errors before merge is a professional standard.

### Method 1 — Jenkins Declarative Linter via REST API

```bash
#!/bin/bash
# validate-jenkinsfile.sh — run in a pre-commit hook or in a PR validation job

JENKINS_URL="http://jenkins.example.com"
AUTH="admin:your-api-token"
JENKINSFILE="${1:-Jenkinsfile}"

# Jenkins exposes a linter at /pipeline-model-converter/validate
RESULT=$(curl -s -X POST \
  -u "${AUTH}" \
  -F "jenkinsfile=<${JENKINSFILE}" \
  "${JENKINS_URL}/pipeline-model-converter/validate")

echo "${RESULT}"

# Exit with error if validation failed
echo "${RESULT}" | grep -q "Errors encountered" && exit 1 || exit 0
```

### Method 2 — npm-based jenkins-pipeline-linter

```bash
# Install
npm install -g jenkins-pipeline-linter-connector

# Validate against a Jenkins server
npx jenkins-pipeline-linter-connector \
  --url http://jenkins.example.com \
  --user admin \
  --token your-api-token \
  --file Jenkinsfile
```

### Method 3 — Pre-commit Hook

```bash
# .git/hooks/pre-commit (or use pre-commit framework)
#!/bin/bash

JENKINS_URL="${JENKINS_URL:-http://localhost:8080}"
AUTH="${JENKINS_USER:-admin}:${JENKINS_TOKEN}"

for file in $(git diff --cached --name-only | grep -E "^Jenkinsfile|/Jenkinsfile$"); do
  echo "Validating: ${file}"
  RESULT=$(curl -s -X POST \
    -u "${AUTH}" \
    -F "jenkinsfile=<${file}" \
    "${JENKINS_URL}/pipeline-model-converter/validate")

  if echo "${RESULT}" | grep -q "Errors encountered\|did not conform"; then
    echo "VALIDATION FAILED for ${file}:"
    echo "${RESULT}"
    exit 1
  fi
  echo "  OK: ${file}"
done
```

### Method 4 — Validate in a PR Pipeline (Meta-pipeline)

```groovy
// A pipeline that validates Jenkinsfiles in pull requests
pipeline {
    agent any

    stages {
        stage('Validate Jenkinsfiles') {
            steps {
                script {
                    def changedFiles = sh(
                        script: "git diff --name-only origin/main...HEAD | grep -E '(^|/)Jenkinsfile' || true",
                        returnStdout: true
                    ).trim().split('\n').findAll { it }

                    if (changedFiles.isEmpty()) {
                        echo "No Jenkinsfiles changed — skipping validation"
                        return
                    }

                    changedFiles.each { file ->
                        echo "Validating: ${file}"
                        def result = sh(
                            script: """
                                curl -s -X POST \\
                                  -u "\${JENKINS_USER}:\${JENKINS_TOKEN}" \\
                                  -F "jenkinsfile=<${file}" \\
                                  "\${JENKINS_URL}/pipeline-model-converter/validate"
                            """,
                            returnStdout: true
                        ).trim()

                        if (result.contains('Errors encountered') || result.contains('did not conform')) {
                            error("Jenkinsfile validation FAILED for ${file}:\n${result}")
                        }
                        echo "  VALID: ${file}"
                    }
                }
            }
        }
    }
}
```

### Interview Q&A — Linting

**Q: How do you prevent a broken Jenkinsfile from being merged to main?**
A: Three layers: (1) A pre-commit hook on developer machines that calls the Jenkins linter API before the commit is even made. (2) A PR validation pipeline (triggered by the GitHub branch protection rule) that runs the linter on any changed Jenkinsfiles and blocks merge if it fails. (3) Require a Shared Library with tested pipeline templates — since projects only call a single function in their Jenkinsfile, there is very little Jenkinsfile syntax to get wrong.

---

## Topic 4: Pipeline Unit Testing with JenkinsPipelineUnit

### Why This Matters
Shared Libraries are production code. They need tests. JenkinsPipelineUnit (JPU) is a Groovy testing framework that lets you unit-test pipeline scripts and Shared Library steps without a running Jenkins server.

### Project Structure

```
jenkins-shared-library/
├── src/
│   └── com/myorg/
│       └── Deployer.groovy
├── vars/
│   ├── buildMavenApp.groovy
│   └── deployToKubernetes.groovy
├── test/
│   └── groovy/
│       ├── BuildMavenAppTest.groovy
│       └── DeployerTest.groovy
├── pom.xml                     ← or build.gradle
└── Jenkinsfile
```

### pom.xml — Test Dependencies

```xml
<dependencies>
    <!-- JenkinsPipelineUnit -->
    <dependency>
        <groupId>com.lesfurets</groupId>
        <artifactId>jenkins-pipeline-unit</artifactId>
        <version>1.19</version>
        <scope>test</scope>
    </dependency>

    <!-- Groovy -->
    <dependency>
        <groupId>org.codehaus.groovy</groupId>
        <artifactId>groovy-all</artifactId>
        <version>3.0.13</version>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.9.2</version>
        <scope>test</scope>
    </dependency>

    <!-- Spock (optional but popular for Groovy tests) -->
    <dependency>
        <groupId>org.spockframework</groupId>
        <artifactId>spock-core</artifactId>
        <version>2.3-groovy-3.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Writing a Test for a Shared Library Step

```groovy
// vars/buildMavenApp.groovy (the step being tested)
def call(Map config = [:]) {
    def goal      = config.goal      ?: 'package'
    def skipTests = config.skipTests ?: false
    def profile   = config.profile   ?: ''

    def goals = "clean ${goal}"
    if (skipTests) goals += " -DskipTests"
    if (profile)   goals += " -P${profile}"

    sh "mvn ${goals}"
}
```

```groovy
// test/groovy/BuildMavenAppTest.groovy
import com.lesfurets.jenkins.unit.BasePipelineTest
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import static org.junit.jupiter.api.Assertions.*

class BuildMavenAppTest extends BasePipelineTest {

    @BeforeEach
    void setUp() {
        super.setUp()
        // Register your shared library location
        helper.registerSharedLibrary(
            library().name('jenkins-shared-lib')
                     .defaultVersion('main')
                     .targetPath('.')
                     .retriever(localSource('.'))
                     .build()
        )
    }

    @Test
    void 'should run mvn clean package by default'() {
        def script = loadScript('vars/buildMavenApp.groovy')
        script.call([:])

        // Assert the sh step was called with the right command
        assertJobStatusSuccess()
        assertTrue(
            helper.callStack.findAll { it.methodName == 'sh' }
                  .any { it.args[0].toString().contains('mvn clean package') },
            "Expected 'mvn clean package' in shell calls"
        )
    }

    @Test
    void 'should add -DskipTests when skipTests is true'() {
        def script = loadScript('vars/buildMavenApp.groovy')
        script.call([skipTests: true])

        assertTrue(
            helper.callStack.findAll { it.methodName == 'sh' }
                  .any { it.args[0].toString().contains('-DskipTests') }
        )
    }

    @Test
    void 'should add Maven profile when specified'() {
        def script = loadScript('vars/buildMavenApp.groovy')
        script.call([goal: 'verify', profile: 'integration-tests'])

        assertTrue(
            helper.callStack.findAll { it.methodName == 'sh' }
                  .any { it.args[0].toString().contains('-Pintegration-tests') }
        )
    }
}
```

### Testing a Full Declarative Pipeline

```groovy
// test/groovy/PipelineTest.groovy
import com.lesfurets.jenkins.unit.declarative.DeclarativePipelineTest
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import static org.junit.jupiter.api.Assertions.*

class PipelineTest extends DeclarativePipelineTest {

    @BeforeEach
    void setUp() {
        super.setUp()

        // Mock environment variables
        binding.setVariable('env', [
            BRANCH_NAME : 'main',
            BUILD_NUMBER: '42',
            GIT_COMMIT  : 'abc1234def5678'
        ])

        // Mock pipeline steps
        helper.registerAllowedMethod('sh', [String.class], { cmd ->
            println "MOCK sh: ${cmd}"
            return 0
        })

        helper.registerAllowedMethod('withCredentials', [List.class, Closure.class], { creds, body ->
            body()
        })

        helper.registerAllowedMethod('input', [Map.class], {
            // Auto-approve the input gate in tests
            println "MOCK input: auto-approved"
        })
    }

    @Test
    void 'pipeline should succeed on main branch'() {
        // Load and run your Jenkinsfile
        def script = loadScript('Jenkinsfile')
        script.execute()

        assertJobStatusSuccess()
        println helper.callStack.collect { it.methodName }.join('\n')
    }

    @Test
    void 'deploy to prod should require input approval'() {
        binding.setVariable('env', [BRANCH_NAME: 'main', BUILD_NUMBER: '43'])

        def inputCalled = false
        helper.registerAllowedMethod('input', [Map.class], {
            inputCalled = true
        })

        def script = loadScript('Jenkinsfile')
        script.execute()

        assertTrue(inputCalled, "Expected production input gate to be triggered")
    }
}
```

### Running Tests

```bash
# Run all pipeline unit tests
mvn test

# Run with verbose output
mvn test -Dsurefire.useFile=false

# In a Jenkins job (testing the library itself)
pipeline {
    agent { docker { image 'maven:3.9-eclipse-temurin-17' } }

    stages {
        stage('Test Shared Library') {
            steps {
                sh 'mvn clean test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    jacoco execPattern: '**/target/jacoco.exec'
                }
            }
        }
    }
}
```

### Interview Q&A — Pipeline Unit Testing

**Q: Why should you unit-test a Jenkins Shared Library?**
A: Shared Libraries are production code used by tens or hundreds of pipelines. A bug in a shared step can break every pipeline simultaneously. Unit tests with JenkinsPipelineUnit let you verify logic changes locally before they propagate, catch regressions when updating the library, and document expected behaviour with executable specifications. It also enables TDD for pipeline code — you can write the test for a new step before implementing it.

---

## Topic 5: Jenkins vs GitHub Actions vs GitLab CI — Comparison

This is one of the most common senior interview questions. You must be able to argue each tool's strengths without being tribal about it.

### Feature Comparison Table

| Feature | Jenkins | GitHub Actions | GitLab CI |
|---|---|---|---|
| Hosting | Self-hosted (you manage) | Cloud (GitHub-managed) or self-hosted | Cloud or self-hosted |
| Pipeline definition | Groovy (Jenkinsfile) | YAML (.github/workflows/) | YAML (.gitlab-ci.yml) |
| Agent management | Full control (SSH, Docker, K8s) | GitHub-hosted runners or self-hosted | GitLab runners (shell, Docker, K8s) |
| Plugin ecosystem | 1,800+ plugins | Marketplace actions (thousands) | Built-in features |
| Cost (cloud compute) | You pay for agent infrastructure | Free minutes + per-minute billing | Free minutes + per-minute billing |
| Learning curve | High | Low-Medium | Medium |
| Customisability | Extremely high | High | High |
| Secret management | Jenkins Credentials / Vault | GitHub Secrets / OIDC | GitLab CI/CD Variables / Vault |
| OIDC / Keyless auth | Via plugin | Native | Native |
| Compliance / audit | Full — you control everything | Limited — GitHub controls the plane | Full on self-hosted |
| Multi-cloud support | Full control | Good | Good |
| Enterprise support | CloudBees | GitHub Enterprise | GitLab Enterprise |
| Parallel jobs | Yes (parallel stages) | Yes (matrix) | Yes (parallel) |
| Caching | Manual (stash/PVC) | Built-in `actions/cache` | Built-in `cache:` keyword |
| Container native | Via Docker/K8s plugins | Yes | Yes |

### When to Choose Jenkins

- You need full control over the build infrastructure (air-gapped environments, regulated industries, on-premise data)
- You have a large number of complex, interdependent pipelines that benefit from the Groovy programming model
- You have legacy Freestyle jobs and a migration budget
- Your organisation requires integration with many internal tools that have Jenkins plugins but no GitHub Actions
- You need fine-grained RBAC with folder-level permissions
- CloudBees support contract is a requirement

### When to Choose GitHub Actions

- Your code is already on GitHub and you want zero infrastructure management
- Your pipelines are relatively straightforward (build, test, push image)
- You want to use the marketplace ecosystem (thousands of prebuilt actions)
- Your team is small or has low operational overhead budget
- You need native GitHub PR integration with status checks

### When to Choose GitLab CI

- Your code is on GitLab
- You want a deeply integrated DevSecOps platform (SAST, DAST, dependency scanning built-in)
- You need native Merge Request pipelines
- You want review apps and environment management built into the platform

### Real-World Hybrid Pattern

Many mature organisations run both Jenkins and GitHub Actions — GitHub Actions handles lightweight PR validation (linting, unit tests, security scans) while Jenkins handles complex deployment pipelines, multi-cloud promotion chains, and legacy integration.

```yaml
# .github/workflows/pr-validation.yml
# GitHub Actions handles fast PR feedback
name: PR Validation
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lint
        run: mvn checkstyle:check
      - name: Unit Test
        run: mvn test
      - name: Trigger Jenkins Deploy Pipeline
        if: github.ref == 'refs/heads/main'
        run: |
          curl -X POST \
            -u "${{ secrets.JENKINS_USER }}:${{ secrets.JENKINS_TOKEN }}" \
            "https://jenkins.example.com/job/deploy/buildWithParameters?IMAGE_TAG=${{ github.sha }}"
```

### Interview Q&A — Tool Comparison

**Q: Your company is currently on Jenkins and is considering migrating to GitHub Actions. What factors would you evaluate?**
A: I'd evaluate: (1) Complexity of existing pipelines — simple pipelines migrate easily, complex Groovy logic and Shared Libraries do not translate directly. (2) Infrastructure requirements — if builds need access to internal systems, VPNs, or on-premise resources, self-hosted runners add operational overhead. (3) Plugin dependencies — any Jenkins plugin that has no Actions equivalent is a blocker or requires custom action development. (4) Cost — compare Jenkins EC2 agent costs vs GitHub Actions per-minute pricing at your build volume. (5) Compliance — regulated industries often cannot use GitHub-hosted runners because you lose control over where the build runs. (6) Team skill — YAML Actions are easier to onboard new engineers to; Groovy Shared Libraries require deeper Jenkins expertise to maintain.

**Q: Can Jenkins and GitHub Actions coexist in the same organisation?**
A: Yes, and this is common. The typical pattern is to use GitHub Actions for fast, lightweight PR feedback (linting, unit tests, container scanning) because it has zero setup overhead for GitHub repositories, and Jenkins for complex deployment pipelines that require access to internal infrastructure, multi-step approval gates, or integration with on-premise systems. Jenkins can also be triggered from GitHub Actions via the REST API, making them complementary rather than competing.

---

## Topic 6: Groovy Sandbox and Script Approval

### Why This Matters
Every Jenkins admin eventually hits a sandbox rejection. Understanding why it exists and how to handle it correctly is fundamental — approving the wrong signatures is a security vulnerability.

### What the Sandbox Is

When a Jenkinsfile or Groovy script runs in Jenkins, it runs inside a Groovy sandbox — a restricted execution environment that blocks certain method calls and API access by default. This prevents pipeline scripts from calling arbitrary Java/system APIs that could compromise the Jenkins controller.

The sandbox applies to: Jenkinsfiles, Shared Library scripts, and any Groovy code run via the Pipeline.

### What the Sandbox Blocks

```groovy
// These will throw sandbox violations without approval:

// File system access
new File('/etc/passwd').text                         // BLOCKED

// System commands (use sh step instead)
Runtime.exec('rm -rf /')                             // BLOCKED

// Reflection
Class.forName('java.lang.Runtime')                   // BLOCKED

// Network access from script (use HTTP Request plugin instead)
new URL('http://internal-api').text                  // BLOCKED

// Jenkins internals (use Script Console for admin tasks)
Jenkins.instance.pluginManager.plugins               // BLOCKED in pipelines
```

### Safe Way — Whitelist Specific Methods

```groovy
// In Manage Jenkins → In-Process Script Approval
// Jenkins shows you the exact signature that needs approval:
// staticMethod org.codehaus.groovy.runtime.DefaultGroovyMethods getText java.io.File
// method java.net.URL openConnection

// Always review WHAT you are approving — never blindly approve all signatures
```

### Pattern: Move Unsafe Logic to a Trusted Library Class

```groovy
// Instead of calling blocked methods directly in a Jenkinsfile,
// put the logic in a @NonCPS method in a Shared Library class

// src/com/myorg/Utils.groovy
package com.myorg

class Utils implements Serializable {

    // @NonCPS means this method is NOT run in the CPS (continuation passing) 
    // transformer — it runs as normal Groovy. Use for utility logic that
    // doesn't call Jenkins steps.
    @NonCPS
    static String parseVersion(String pomContent) {
        def matcher = pomContent =~ /<version>(.+?)<\/version>/
        return matcher ? matcher[0][1] : 'unknown'
    }

    @NonCPS
    static Map parseJson(String jsonString) {
        return new groovy.json.JsonSlurper().parseText(jsonString)
    }
}
```

### Understanding @NonCPS

```groovy
// CPS (Continuation Passing Style) transformation is how Jenkins makes
// pipelines resumable — it serializes the pipeline state to disk so it
// can survive a Jenkins restart mid-build.
//
// The CPS transformer does not work well with all Groovy constructs:
// - closures used as first-class values
// - try-catch in certain patterns
// - complex iterators
//
// @NonCPS opts the method out of CPS transformation.
// IMPORTANT: @NonCPS methods cannot call CPS-transformed steps (sh, echo, etc.)

// WRONG: calling sh inside @NonCPS
@NonCPS
def buildAndPush() {
    sh 'docker build .'   // This will fail — sh is a CPS step
}

// RIGHT: @NonCPS for pure logic, normal method for steps
@NonCPS
String buildDockerCommand(String tag) {
    return "docker build -t myapp:${tag} --build-arg VERSION=${tag} ."
}

def buildAndPush(String tag) {
    def cmd = buildDockerCommand(tag)   // call the @NonCPS helper
    sh cmd                              // call the step normally
}
```

### Script Console vs Pipeline Sandbox

```groovy
// The Script Console (Manage Jenkins → Script Console) runs WITHOUT the sandbox.
// It has full access to Jenkins internals and the JVM.
// Only admins should have access.

// Script Console example: list all offline agents
import hudson.model.Computer
Jenkins.instance.computers.findAll { !it.online }.each { computer ->
    println "${computer.displayName} has been offline since: ${computer.offlineCauseReason}"
}

// Pipelines run WITH the sandbox by default.
// To run a Groovy step outside the sandbox (only when absolutely necessary):
script {
    // 'script' block still runs in sandbox
    // To escape sandbox for a specific call, it must be approved via Script Approval
}
```

### Interview Q&A — Groovy Sandbox

**Q: A pipeline fails with "Scripts not permitted to use staticMethod..." — how do you handle it?**
A: First, understand WHY the script needs that method. If it's because a developer wrote `new File(...)` in a Jenkinsfile to read a config, the right fix is to rewrite the logic using proper pipeline steps (e.g., `readFile` step) rather than approving the raw Java API. If the method is legitimately needed in a Shared Library class and is not exploitable, go to Manage Jenkins → In-Process Script Approval, review the exact signature being requested, and approve it. Never approve method signatures that provide file system write access, network access, or `exec` capability without a security review.

**Q: What is @NonCPS and when would you use it?**
A: `@NonCPS` marks a method to be excluded from Jenkins' CPS (Continuation Passing Style) transformation. Use it for utility methods that perform pure Groovy logic — string manipulation, JSON parsing, list processing — where the CPS transformer would cause issues (particularly with closures and complex iterators). The constraint is that `@NonCPS` methods cannot call pipeline steps (`sh`, `echo`, `withCredentials`, etc.) because those are themselves CPS-transformed. The rule of thumb: use `@NonCPS` for data transformation logic, keep pipeline step calls in CPS-transformed methods.

---

## Topic 7: Blue-Green Deployment Implementation

### Concept

Blue-Green deployment maintains two identical production environments. At any time, one is live (Blue) and the other is idle (Green). You deploy to Green, verify it, then switch traffic. Blue becomes the rollback target.

### Implementation with Kubernetes

```groovy
// Jenkinsfile — Blue-Green on Kubernetes

pipeline {
    agent { label 'kubernetes' }

    environment {
        APP_NAME = 'myapp'
        REGISTRY = 'registry.example.com'
        NAMESPACE = 'production'
    }

    stages {
        stage('Determine Active Slot') {
            steps {
                script {
                    // Find which slot (blue/green) is currently active
                    env.ACTIVE_SLOT = sh(
                        script: """
                            kubectl get service ${APP_NAME}-active \
                              -n ${NAMESPACE} \
                              -o jsonpath='{.spec.selector.slot}' 2>/dev/null || echo 'blue'
                        """,
                        returnStdout: true
                    ).trim()

                    env.INACTIVE_SLOT = (env.ACTIVE_SLOT == 'blue') ? 'green' : 'blue'

                    echo "Active slot: ${env.ACTIVE_SLOT}"
                    echo "Deploying to: ${env.INACTIVE_SLOT}"
                }
            }
        }

        stage('Build & Push Image') {
            steps {
                sh "docker build -t ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ."
                withCredentials([usernamePassword(credentialsId: 'registry-creds',
                    usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    sh "echo \$REG_PASS | docker login ${REGISTRY} -u \$REG_USER --password-stdin"
                    sh "docker push ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to Inactive Slot') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                    sh """
                        # Update the inactive slot's deployment
                        kubectl set image deployment/${APP_NAME}-${INACTIVE_SLOT} \
                          ${APP_NAME}=${REGISTRY}/${APP_NAME}:${BUILD_NUMBER} \
                          -n ${NAMESPACE}

                        # Wait for rollout to complete
                        kubectl rollout status deployment/${APP_NAME}-${INACTIVE_SLOT} \
                          -n ${NAMESPACE} --timeout=5m
                    """
                }
            }
        }

        stage('Smoke Test Inactive Slot') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                    script {
                        // Get the internal service endpoint for the inactive slot
                        def internalUrl = sh(
                            script: "kubectl get service ${APP_NAME}-${INACTIVE_SLOT} -n ${NAMESPACE} -o jsonpath='{.spec.clusterIP}'",
                            returnStdout: true
                        ).trim()

                        // Run smoke tests against the inactive slot
                        sh """
                            kubectl run smoke-test-${BUILD_NUMBER} \
                              --image=curlimages/curl --restart=Never \
                              --rm --attach \
                              -n ${NAMESPACE} \
                              -- curl -f -s http://${internalUrl}/health
                        """
                    }
                }
            }
        }

        stage('Switch Traffic (Cut Over)') {
            steps {
                input message: "Smoke tests passed. Switch traffic from ${env.ACTIVE_SLOT} to ${env.INACTIVE_SLOT}?",
                      ok: "Switch",
                      submitter: 'release-managers'

                withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                    sh """
                        # Patch the active service to point to the new slot
                        kubectl patch service ${APP_NAME}-active \
                          -n ${NAMESPACE} \
                          -p '{"spec":{"selector":{"slot":"${INACTIVE_SLOT}"}}}'

                        echo "Traffic switched from ${ACTIVE_SLOT} to ${INACTIVE_SLOT}"
                        echo "Build ${BUILD_NUMBER} is now live"
                        echo "Previous version on ${ACTIVE_SLOT} is on standby for rollback"
                    """
                }
            }
        }
    }

    post {
        failure {
            // On failure, ensure the old active slot is still serving traffic
            withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                sh """
                    kubectl patch service ${APP_NAME}-active \
                      -n ${NAMESPACE} \
                      -p '{"spec":{"selector":{"slot":"${ACTIVE_SLOT}"}}}'
                    echo "Failure detected — traffic remains on ${ACTIVE_SLOT}"
                """
            }
        }
    }
}
```

---

## Topic 8: Canary Deployment Implementation

### Concept

Canary releases send a small percentage of traffic to the new version first, observe metrics, then either promote to full rollout or rollback.

```groovy
pipeline {
    agent { label 'kubernetes' }

    environment {
        APP_NAME  = 'myapp'
        REGISTRY  = 'registry.example.com'
        NAMESPACE = 'production'
    }

    parameters {
        string(name: 'CANARY_WEIGHT', defaultValue: '10', description: 'Percentage of traffic to send to canary (0-100)')
        string(name: 'OBSERVE_MINUTES', defaultValue: '15', description: 'Minutes to observe canary before promoting')
    }

    stages {
        stage('Build & Push') {
            steps {
                sh "docker build -t ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ."
                sh "docker push ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
            }
        }

        stage('Deploy Canary') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                    sh """
                        # Deploy canary with a small replica count (e.g. 1 out of 10 total pods)
                        cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-canary
  namespace: ${NAMESPACE}
  labels:
    app: ${APP_NAME}
    track: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}
      track: canary
  template:
    metadata:
      labels:
        app: ${APP_NAME}
        track: canary
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
        ports:
        - containerPort: 8080
EOF
                        kubectl rollout status deployment/${APP_NAME}-canary -n ${NAMESPACE} --timeout=3m
                        echo "Canary deployed — receiving approx ${params.CANARY_WEIGHT}% of traffic"
                    """
                }
            }
        }

        stage('Observe Canary Metrics') {
            steps {
                script {
                    echo "Observing canary for ${params.OBSERVE_MINUTES} minutes..."
                    def observeSeconds = params.OBSERVE_MINUTES.toInteger() * 60
                    def interval = 60 // check every minute

                    for (int elapsed = 0; elapsed < observeSeconds; elapsed += interval) {
                        sleep(interval)

                        // Query Prometheus or Datadog for error rate on canary pods
                        def errorRate = sh(
                            script: """
                                curl -s "http://prometheus:9090/api/v1/query" \
                                  --data-urlencode 'query=rate(http_requests_total{job="${APP_NAME}-canary",status=~"5.."}[1m]) / rate(http_requests_total{job="${APP_NAME}-canary"}[1m]) * 100' \
                                  | python3 -c "import sys,json; r=json.load(sys.stdin)['data']['result']; print(r[0]['value'][1] if r else '0')"
                            """,
                            returnStdout: true
                        ).trim().toFloat()

                        echo "Elapsed: ${elapsed}s | Canary error rate: ${errorRate}%"

                        if (errorRate > 5.0) {
                            error("Canary error rate ${errorRate}% exceeds threshold of 5% — triggering automatic rollback")
                        }
                    }
                    echo "Canary metrics healthy — proceeding to full rollout"
                }
            }
        }

        stage('Promote to Full Rollout') {
            steps {
                input message: "Canary healthy for ${params.OBSERVE_MINUTES} minutes. Promote to 100% traffic?",
                      ok: 'Promote',
                      submitter: 'release-managers'

                withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                    sh """
                        # Update the stable deployment to the new image
                        kubectl set image deployment/${APP_NAME}-stable \
                          ${APP_NAME}=${REGISTRY}/${APP_NAME}:${BUILD_NUMBER} \
                          -n ${NAMESPACE}

                        kubectl rollout status deployment/${APP_NAME}-stable -n ${NAMESPACE} --timeout=5m

                        # Remove the canary deployment
                        kubectl delete deployment ${APP_NAME}-canary -n ${NAMESPACE}

                        echo "Full promotion complete — ${BUILD_NUMBER} is now 100% live"
                    """
                }
            }
        }
    }

    post {
        failure {
            withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                sh """
                    kubectl delete deployment ${APP_NAME}-canary -n ${NAMESPACE} --ignore-not-found=true
                    echo "Canary rolled back — stable deployment unchanged"
                """
            }
        }
    }
}
```

---

## Topic 9: Rollback Strategy Patterns

### Pattern 1 — Helm Rollback

```groovy
stage('Deploy with Helm') {
    steps {
        script {
            // Record the previous revision before deploying
            env.PREVIOUS_REVISION = sh(
                script: "helm history ${APP_NAME} -n ${NAMESPACE} --max 1 -o json | python3 -c \"import sys,json; h=json.load(sys.stdin); print(h[0]['revision']) if h else print('0')\"",
                returnStdout: true
            ).trim()

            sh """
                helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                  --namespace ${NAMESPACE} \
                  --set image.tag=${BUILD_NUMBER} \
                  --wait --timeout 5m \
                  --atomic   # <-- automatically rolls back if deployment fails
            """
        }
    }
}

// Manual rollback stage (for post-deployment issues found after smoke tests)
stage('Rollback if Needed') {
    when {
        expression { return params.ROLLBACK == true }
    }
    steps {
        input message: "Rolling back to revision ${env.PREVIOUS_REVISION}. Confirm?", ok: 'Rollback'
        sh "helm rollback ${APP_NAME} ${env.PREVIOUS_REVISION} -n ${NAMESPACE} --wait"
        echo "Rollback complete — now on revision ${env.PREVIOUS_REVISION}"
    }
}
```

### Pattern 2 — Kubernetes `kubectl rollout undo`

```groovy
// Rollback Kubernetes deployment to the previous replica set
stage('Emergency Rollback') {
    steps {
        withKubeConfig([credentialsId: 'kubeconfig-prod']) {
            sh """
                # View rollout history first
                kubectl rollout history deployment/${APP_NAME} -n production

                # Roll back to the previous version
                kubectl rollout undo deployment/${APP_NAME} -n production

                # Or roll back to a specific revision
                kubectl rollout undo deployment/${APP_NAME} -n production --to-revision=5

                # Verify
                kubectl rollout status deployment/${APP_NAME} -n production --timeout=3m
            """
        }
    }
}
```

### Pattern 3 — Artifact Version Rollback Pipeline

```groovy
// A dedicated rollback pipeline with version selection
pipeline {
    agent any

    parameters {
        choice(
            name: 'ROLLBACK_VERSION',
            choices: getRecentVersions(),    // dynamic — fetches last 10 versions from Nexus
            description: 'Select the version to roll back to'
        )
        string(name: 'NAMESPACE', defaultValue: 'production', description: 'Target namespace')
    }

    stages {
        stage('Validate Rollback Target') {
            steps {
                script {
                    echo "Rolling back to: ${params.ROLLBACK_VERSION}"
                    // Verify the image exists in the registry before deploying
                    sh "docker manifest inspect ${REGISTRY}/${APP_NAME}:${params.ROLLBACK_VERSION}"
                }
            }
        }

        stage('Confirm') {
            steps {
                input message: "Deploy ${APP_NAME}:${params.ROLLBACK_VERSION} to ${params.NAMESPACE}?",
                      ok: 'Rollback',
                      submitter: 'release-managers,on-call-engineers'
            }
        }

        stage('Execute Rollback') {
            steps {
                withKubeConfig([credentialsId: "kubeconfig-${params.NAMESPACE}"]) {
                    sh """
                        helm upgrade ${APP_NAME} ./helm/${APP_NAME} \
                          --namespace ${params.NAMESPACE} \
                          --set image.tag=${params.ROLLBACK_VERSION} \
                          --wait --timeout 5m
                    """
                }
            }
        }
    }
}

// Helper function to fetch recent versions from Nexus
def getRecentVersions() {
    try {
        def response = sh(
            script: """curl -s 'http://nexus/service/rest/v1/search?repository=docker&name=${APP_NAME}&sort=version&direction=desc&limit=10' | python3 -c "import sys,json; [print(i['version']) for i in json.load(sys.stdin)['items']]" """,
            returnStdout: true
        ).trim().split('\n').toList()
        return response
    } catch (e) {
        return ['latest', 'previous']
    }
}
```

### Interview Q&A — Deployment Strategies and Rollback

**Q: What is the main operational advantage of Blue-Green over a rolling update?**
A: The key advantage is instantaneous rollback. With a rolling update, rolling back requires deploying the previous version again (another rolling update, taking minutes). With Blue-Green, rollback is just switching the load balancer back to the old slot — sub-second and requires no new deployment. The tradeoff is cost: you need double the production resources to maintain both slots simultaneously.

**Q: When would you choose a Canary release over Blue-Green?**
A: Canary is better when you want to validate a change against real production traffic before committing to it fully, and when you have observability tooling (Prometheus, Datadog) that can measure the canary's error rate, latency, and business metrics in real time. Blue-Green is better for simple go/no-go switches where you trust your pre-production testing. Canary provides more data but adds complexity — you need traffic splitting (Istio, Nginx weighted routing, or ALB weighted target groups) and automated metric monitoring.

---

## Topic 10: Notification Integrations — Slack, Teams, PagerDuty

### Slack Integration

```groovy
// Install: Slack Notification Plugin
// Configure: Manage Jenkins → System → Slack
// Workspace: your-workspace
// Credential: Slack Bot Token (stored as Secret Text in Jenkins Credentials)
// Default channel: #ci-notifications

pipeline {
    agent any

    stages {
        stage('Build') {
            steps { sh 'mvn clean package' }
        }
    }

    post {
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: """
✅ *BUILD SUCCESS* — `${env.JOB_NAME}` #${env.BUILD_NUMBER}
• Branch: `${env.BRANCH_NAME}`
• Duration: ${currentBuild.durationString}
• <${env.BUILD_URL}|View Build>
                """.stripIndent()
            )
        }

        failure {
            slackSend(
                channel: '#ci-failures',
                color: 'danger',
                message: """
❌ *BUILD FAILED* — `${env.JOB_NAME}` #${env.BUILD_NUMBER}
• Branch: `${env.BRANCH_NAME}`
• Failed Stage: ${env.FAILED_STAGE ?: 'Unknown'}
• <${env.BUILD_URL}console|View Console Log>
• <!here> Please investigate
                """.stripIndent()
            )
        }

        unstable {
            slackSend(
                channel: '#ci-notifications',
                color: 'warning',
                message: "⚠️ *UNSTABLE* — `${env.JOB_NAME}` #${env.BUILD_NUMBER} — Flaky tests detected. <${env.BUILD_URL}|View>"
            )
        }
    }
}
```

### Microsoft Teams Integration

```groovy
// Install: Office 365 Connector Plugin
// Or use a webhook URL directly

def notifyTeams(String status, String color, String message) {
    def webhookUrl = 'https://myorg.webhook.office.com/webhookb2/YOUR-WEBHOOK-URL'

    def payload = """{
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "themeColor": "${color}",
        "summary": "${status}: ${env.JOB_NAME}",
        "sections": [{
            "activityTitle": "${status}: **${env.JOB_NAME}** #${env.BUILD_NUMBER}",
            "activitySubtitle": "${message}",
            "facts": [
                { "name": "Branch", "value": "${env.BRANCH_NAME}" },
                { "name": "Duration", "value": "${currentBuild.durationString}" },
                { "name": "Triggered by", "value": "${currentBuild.getBuildCauses()[0]?.shortDescription ?: 'Unknown'}" }
            ],
            "markdown": true
        }],
        "potentialAction": [{
            "@type": "OpenUri",
            "name": "View Build",
            "targets": [{"os": "default", "uri": "${env.BUILD_URL}"}]
        }]
    }"""

    withCredentials([string(credentialsId: 'teams-webhook-url', variable: 'TEAMS_URL')]) {
        sh """
            curl -s -X POST \
              -H 'Content-Type: application/json' \
              -d '${payload}' \
              "\${TEAMS_URL}"
        """
    }
}

pipeline {
    agent any
    stages {
        stage('Deploy') { steps { sh './deploy.sh' } }
    }
    post {
        success { notifyTeams('✅ SUCCESS', '00FF00', 'Deployment completed successfully') }
        failure { notifyTeams('❌ FAILURE', 'FF0000', 'Deployment failed — immediate action required') }
    }
}
```

### PagerDuty Integration — Create Incidents on Failure

```groovy
def createPagerDutyIncident(String summary, String details) {
    withCredentials([string(credentialsId: 'pagerduty-routing-key', variable: 'PD_KEY')]) {
        sh """
            curl -s -X POST \
              -H 'Content-Type: application/json' \
              -H 'Authorization: Token token=\${PD_KEY}' \
              'https://events.pagerduty.com/v2/enqueue' \
              -d '{
                "routing_key": "\${PD_KEY}",
                "event_action": "trigger",
                "dedup_key": "${env.JOB_NAME}-${env.BUILD_NUMBER}",
                "payload": {
                  "summary": "${summary}",
                  "source": "Jenkins",
                  "severity": "critical",
                  "custom_details": {
                    "job": "${env.JOB_NAME}",
                    "build": "${env.BUILD_NUMBER}",
                    "url": "${env.BUILD_URL}",
                    "details": "${details}"
                  }
                }
              }'
        """
    }
}

// Resolve the incident when the next build succeeds
def resolvePagerDutyIncident() {
    withCredentials([string(credentialsId: 'pagerduty-routing-key', variable: 'PD_KEY')]) {
        sh """
            curl -s -X POST \
              -H 'Content-Type: application/json' \
              'https://events.pagerduty.com/v2/enqueue' \
              -d '{
                "routing_key": "\${PD_KEY}",
                "event_action": "resolve",
                "dedup_key": "${env.JOB_NAME}-${env.BUILD_NUMBER}"
              }'
        """
    }
}

pipeline {
    agent any
    stages {
        stage('Production Deploy') {
            steps { sh 'helm upgrade --install myapp ./helm/myapp --namespace production' }
        }
    }
    post {
        failure {
            // Only page on-call for production deployments on main
            script {
                if (env.BRANCH_NAME == 'main') {
                    createPagerDutyIncident(
                        "Production deployment FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        "Build URL: ${env.BUILD_URL}"
                    )
                }
            }
        }
        success {
            script {
                if (env.BRANCH_NAME == 'main') {
                    resolvePagerDutyIncident()
                }
            }
        }
    }
}
```

---

## Topic 11: Docker Registry Integration — ECR, ACR, Harbor

### AWS ECR (Elastic Container Registry)

```groovy
// ECR authentication using IAM role (preferred — no static credentials)
// Requires: EC2 instance profile or EKS service account with ECR permissions

pipeline {
    agent { label 'aws-agent' }

    environment {
        AWS_REGION   = 'us-east-1'
        AWS_ACCOUNT  = '123456789012'
        ECR_REGISTRY = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        REPO_NAME    = 'myapp'
        IMAGE        = "${ECR_REGISTRY}/${REPO_NAME}:${BUILD_NUMBER}"
    }

    stages {
        stage('Build') {
            steps {
                sh "docker build -t ${IMAGE} ."
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    // Get an ECR auth token (valid 12 hours)
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                          docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        docker push ${IMAGE}

                        # Also tag as latest
                        docker tag ${IMAGE} ${ECR_REGISTRY}/${REPO_NAME}:latest
                        docker push ${ECR_REGISTRY}/${REPO_NAME}:latest
                    """
                }
            }
        }

        stage('Scan in ECR') {
            steps {
                sh """
                    # Trigger ECR image scan
                    aws ecr start-image-scan \
                      --repository-name ${REPO_NAME} \
                      --image-id imageTag=${BUILD_NUMBER} \
                      --region ${AWS_REGION}

                    # Wait for scan to complete
                    aws ecr wait image-scan-complete \
                      --repository-name ${REPO_NAME} \
                      --image-id imageTag=${BUILD_NUMBER} \
                      --region ${AWS_REGION}

                    # Fail if CRITICAL vulnerabilities found
                    CRITICAL=$(aws ecr describe-image-scan-findings \
                      --repository-name ${REPO_NAME} \
                      --image-id imageTag=${BUILD_NUMBER} \
                      --region ${AWS_REGION} \
                      --query 'imageScanFindings.findingSeverityCounts.CRITICAL' \
                      --output text)

                    echo "Critical vulnerabilities: \${CRITICAL}"
                    [ "\${CRITICAL}" = "None" ] || [ "\${CRITICAL}" = "0" ] || \
                      { echo "Critical vulnerabilities found!"; exit 1; }
                """
            }
        }
    }
}
```

### Azure ACR (Azure Container Registry)

```groovy
pipeline {
    agent any

    environment {
        ACR_NAME     = 'myorgregistry'
        ACR_REGISTRY = "${ACR_NAME}.azurecr.io"
        REPO_NAME    = 'myapp'
        IMAGE        = "${ACR_REGISTRY}/${REPO_NAME}:${BUILD_NUMBER}"
    }

    stages {
        stage('Login to ACR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'acr-service-principal',
                    usernameVariable: 'SP_CLIENT_ID',
                    passwordVariable: 'SP_CLIENT_SECRET'
                )]) {
                    sh """
                        az login --service-principal \
                          --username \${SP_CLIENT_ID} \
                          --password \${SP_CLIENT_SECRET} \
                          --tenant your-tenant-id

                        az acr login --name ${ACR_NAME}
                    """
                }
            }
        }

        stage('Build and Push') {
            steps {
                sh """
                    docker build -t ${IMAGE} .
                    docker push ${IMAGE}
                """
            }
        }
    }
}
```

### Harbor (Private Registry with Content Trust)

```groovy
pipeline {
    agent any

    environment {
        HARBOR_REGISTRY = 'harbor.example.com'
        HARBOR_PROJECT  = 'myorg'
        IMAGE           = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/myapp:${BUILD_NUMBER}"
    }

    stages {
        stage('Build & Push to Harbor') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-creds',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh """
                        echo \${HARBOR_PASS} | docker login ${HARBOR_REGISTRY} \
                          -u \${HARBOR_USER} --password-stdin

                        docker build -t ${IMAGE} .
                        docker push ${IMAGE}
                    """
                }
            }
        }

        stage('Verify Vulnerability Policy') {
            steps {
                withCredentials([string(credentialsId: 'harbor-api-token', variable: 'HARBOR_TOKEN')]) {
                    sh """
                        # Trigger Harbor scan
                        curl -s -X POST \
                          -H "Authorization: Bearer \${HARBOR_TOKEN}" \
                          "https://${HARBOR_REGISTRY}/api/v2.0/projects/${HARBOR_PROJECT}/repositories/myapp/artifacts/${BUILD_NUMBER}/scan"

                        # Wait and check scan result
                        sleep 30
                        RESULT=\$(curl -s \
                          -H "Authorization: Bearer \${HARBOR_TOKEN}" \
                          "https://${HARBOR_REGISTRY}/api/v2.0/projects/${HARBOR_PROJECT}/repositories/myapp/artifacts/${BUILD_NUMBER}" \
                          | python3 -c "import sys,json; a=json.load(sys.stdin); print(a.get('scan_overview',{}).get('application/vnd.security.vulnerability.report; version=1.1',{}).get('severity','UNKNOWN'))")

                        echo "Scan result: \${RESULT}"
                        [[ "\${RESULT}" == "None" || "\${RESULT}" == "Low" || "\${RESULT}" == "Medium" ]] || \
                          { echo "High/Critical vulnerabilities found in Harbor scan"; exit 1; }
                    """
                }
            }
        }
    }
}
```

---

## Topic 12: Container Image Signing with Cosign

### Why This Matters
Supply chain attacks (like SolarWinds) target the software delivery pipeline. Signing container images cryptographically ensures that the image you deploy is exactly the one your CI pipeline produced — and nobody tampered with it between registry push and deployment.

### Cosign Signing in a Pipeline

```groovy
pipeline {
    agent { label 'docker' }

    environment {
        IMAGE    = "registry.example.com/myapp:${BUILD_NUMBER}"
        REGISTRY = 'registry.example.com'
    }

    stages {
        stage('Build and Push') {
            steps {
                sh "docker build -t ${IMAGE} ."
                sh "docker push ${IMAGE}"
            }
        }

        stage('Sign Image with Cosign') {
            steps {
                withCredentials([
                    string(credentialsId: 'cosign-private-key', variable: 'COSIGN_KEY'),
                    string(credentialsId: 'cosign-password',    variable: 'COSIGN_PASSWORD')
                ]) {
                    sh """
                        # Write private key to temp file
                        echo "\${COSIGN_KEY}" > /tmp/cosign.key

                        # Sign the image — sign by digest for immutability
                        IMAGE_DIGEST=\$(docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE})

                        COSIGN_PASSWORD="\${COSIGN_PASSWORD}" cosign sign \
                          --key /tmp/cosign.key \
                          "\${IMAGE_DIGEST}"

                        # Attach SBOM as attestation
                        syft ${IMAGE} -o cyclonedx-json > sbom.json
                        COSIGN_PASSWORD="\${COSIGN_PASSWORD}" cosign attest \
                          --key /tmp/cosign.key \
                          --predicate sbom.json \
                          --type cyclonedx \
                          "\${IMAGE_DIGEST}"

                        rm -f /tmp/cosign.key
                        echo "Image signed and attested: \${IMAGE_DIGEST}"
                    """
                }
            }
        }

        stage('Verify Signature Before Deploy') {
            steps {
                withCredentials([string(credentialsId: 'cosign-public-key', variable: 'COSIGN_PUB_KEY')]) {
                    sh """
                        echo "\${COSIGN_PUB_KEY}" > /tmp/cosign.pub
                        IMAGE_DIGEST=\$(docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE})

                        cosign verify \
                          --key /tmp/cosign.pub \
                          "\${IMAGE_DIGEST}"

                        rm -f /tmp/cosign.pub
                        echo "Signature verified — image is authentic"
                    """
                }
            }
        }
    }
}
```

### Keyless Signing with OIDC (GitHub/Jenkins OIDC)

```groovy
// Keyless signing uses the OIDC identity of the build environment
// No key management needed — relies on Sigstore's Fulcio CA and Rekor transparency log
stage('Sign Keyless') {
    steps {
        sh """
            # Requires OIDC token from Jenkins (configure OIDC provider in Sigstore)
            cosign sign \
              --oidc-issuer https://jenkins.example.com \
              --identity-token \$(cat /var/run/sigstore/token) \
              ${IMAGE}@\$(cosign triangulate ${IMAGE})
        """
    }
}
```

---

## Topic 13: SBOM Generation — Syft and CycloneDX

```groovy
pipeline {
    agent any

    environment {
        IMAGE    = "registry.example.com/myapp:${BUILD_NUMBER}"
        APP_NAME = 'myapp'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
                sh "docker build -t ${IMAGE} ."
            }
        }

        stage('Generate SBOM — Source') {
            steps {
                // Generate SBOM from Maven project (Java dependencies)
                sh """
                    # Using CycloneDX Maven plugin
                    mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom \
                      -DoutputFormat=json \
                      -DoutputName=sbom-source

                    echo "Source SBOM generated: target/sbom-source.json"
                """
                archiveArtifacts artifacts: 'target/sbom-source.json', fingerprint: true
            }
        }

        stage('Generate SBOM — Container') {
            steps {
                sh """
                    # Using Syft to generate SBOM from the container image
                    syft ${IMAGE} \
                      -o cyclonedx-json \
                      --file sbom-container.json

                    # Also generate SPDX format (required by some compliance frameworks)
                    syft ${IMAGE} \
                      -o spdx-json \
                      --file sbom-container-spdx.json

                    echo "Container SBOM generated"
                """
                archiveArtifacts artifacts: 'sbom-container*.json', fingerprint: true
            }
        }

        stage('Validate SBOM — Grype Vulnerability Scan') {
            steps {
                sh """
                    # Scan the SBOM for known vulnerabilities using Grype
                    grype sbom:sbom-container.json \
                      --output json \
                      --file grype-results.json \
                      --fail-on high

                    echo "Grype vulnerability scan complete"
                """
                archiveArtifacts artifacts: 'grype-results.json'
            }
        }

        stage('Publish SBOM to Registry') {
            steps {
                withCredentials([string(credentialsId: 'cosign-key', variable: 'COSIGN_KEY')]) {
                    sh """
                        # Attach SBOM as a cosign attestation to the image
                        COSIGN_PASSWORD="" cosign attest \
                          --key env://COSIGN_KEY \
                          --predicate sbom-container.json \
                          --type cyclonedx \
                          ${IMAGE}
                    """
                }
            }
        }
    }
}
```

---

## Topic 14: Terraform Pipeline Integration

```groovy
// Infrastructure pipeline: Plan → Review → Apply
// Uses Terraform with remote state in S3 and workspace per environment

pipeline {
    agent { docker { image 'hashicorp/terraform:1.6' } }

    parameters {
        choice(name: 'ACTION',      choices: ['plan', 'apply', 'destroy'], description: 'Terraform action')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'],   description: 'Target environment')
        string(name: 'TF_VAR_instance_count', defaultValue: '2', description: 'Number of instances')
    }

    environment {
        TF_WORKSPACE = "${params.ENVIRONMENT}"
        TF_IN_AUTOMATION = 'true'      // Suppresses interactive prompts
        TF_CLI_ARGS = '-no-color'      // Clean output in Jenkins logs
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Terraform Init') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key',  variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        cd terraform/environments/${params.ENVIRONMENT}

                        terraform init \
                          -backend-config="bucket=myorg-terraform-state" \
                          -backend-config="key=${params.ENVIRONMENT}/terraform.tfstate" \
                          -backend-config="region=us-east-1" \
                          -input=false

                        terraform workspace select ${params.ENVIRONMENT} || \
                          terraform workspace new ${params.ENVIRONMENT}
                    """
                }
            }
        }

        stage('Terraform Validate & Lint') {
            steps {
                sh """
                    cd terraform/environments/${params.ENVIRONMENT}
                    terraform validate
                    terraform fmt -check -recursive
                """
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',    variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        cd terraform/environments/${params.ENVIRONMENT}
                        terraform plan \
                          -var="instance_count=${params.TF_VAR_instance_count}" \
                          -out=tfplan \
                          -input=false
                        terraform show -json tfplan > tfplan.json
                    """
                }

                // Archive the plan for review
                archiveArtifacts artifacts: 'terraform/environments/${params.ENVIRONMENT}/tfplan.json'

                // Show a human-readable summary
                sh "cd terraform/environments/${params.ENVIRONMENT} && terraform show tfplan"
            }
        }

        stage('Security Scan — tfsec / Checkov') {
            steps {
                sh """
                    # tfsec static analysis
                    tfsec terraform/environments/${params.ENVIRONMENT} \
                      --format json \
                      --out tfsec-results.json || true

                    # Checkov policy scan
                    checkov -d terraform/environments/${params.ENVIRONMENT} \
                      --output json \
                      --output-file checkov-results.json || true
                """
                archiveArtifacts artifacts: 'tfsec-results.json,checkov-results.json', allowEmptyArchive: true
            }
        }

        stage('Approval Gate') {
            when {
                expression { return params.ACTION in ['apply', 'destroy'] }
            }
            steps {
                input(
                    message: "Review the Terraform plan above. Approve ${params.ACTION.toUpperCase()} on ${params.ENVIRONMENT}?",
                    ok: params.ACTION.capitalize(),
                    submitter: params.ENVIRONMENT == 'prod' ? 'infra-leads,on-call-engineers' : 'infra-team'
                )
            }
        }

        stage('Terraform Apply') {
            when {
                expression { return params.ACTION == 'apply' }
            }
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',    variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        cd terraform/environments/${params.ENVIRONMENT}
                        terraform apply -input=false tfplan
                    """
                }
            }
        }

        stage('Terraform Destroy') {
            when {
                expression { return params.ACTION == 'destroy' }
            }
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',    variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        cd terraform/environments/${params.ENVIRONMENT}
                        terraform destroy -auto-approve -input=false
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
```

---

## Topic 15: Ansible Pipeline Integration

```groovy
// Configuration management pipeline using Ansible

pipeline {
    agent { docker { image 'cytopia/ansible:latest-tools' } }

    parameters {
        choice(name: 'ENVIRONMENT',  choices: ['dev', 'staging', 'prod'], description: 'Target environment')
        choice(name: 'PLAYBOOK',     choices: ['site', 'webservers', 'databases', 'deploy-app'], description: 'Playbook to run')
        string(name: 'EXTRA_VARS',   defaultValue: '', description: 'Extra Ansible variables (key=value space-separated)')
        booleanParam(name: 'DRY_RUN', defaultValue: true, description: 'Run in check mode (--check)')
    }

    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        ANSIBLE_FORCE_COLOR       = 'true'
        ANSIBLE_STDOUT_CALLBACK   = 'yaml'
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Lint Playbooks') {
            steps {
                sh """
                    ansible-lint playbooks/${params.PLAYBOOK}.yml \
                      -R -r rules/ \
                      --parseable-severity
                """
            }
        }

        stage('Syntax Check') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ansible-deploy-key',
                    keyFileVariable: 'ANSIBLE_KEY'
                )]) {
                    sh """
                        ansible-playbook \
                          -i inventories/${params.ENVIRONMENT}/hosts \
                          --private-key \${ANSIBLE_KEY} \
                          --syntax-check \
                          playbooks/${params.PLAYBOOK}.yml
                    """
                }
            }
        }

        stage('Dry Run (Check Mode)') {
            when {
                expression { return params.DRY_RUN }
            }
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ansible-deploy-key',
                    keyFileVariable: 'ANSIBLE_KEY'
                )]) {
                    sh """
                        ansible-playbook \
                          -i inventories/${params.ENVIRONMENT}/hosts \
                          --private-key \${ANSIBLE_KEY} \
                          --check --diff \
                          ${params.EXTRA_VARS ? "--extra-vars '${params.EXTRA_VARS}'" : ''} \
                          playbooks/${params.PLAYBOOK}.yml
                    """
                }
            }
        }

        stage('Apply Playbook') {
            when {
                expression { return !params.DRY_RUN }
            }
            steps {
                input message: "Apply '${params.PLAYBOOK}' playbook to ${params.ENVIRONMENT}?",
                      ok: 'Apply'

                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ansible-deploy-key',
                    keyFileVariable: 'ANSIBLE_KEY'
                )]) {
                    sh """
                        ansible-playbook \
                          -i inventories/${params.ENVIRONMENT}/hosts \
                          --private-key \${ANSIBLE_KEY} \
                          ${params.EXTRA_VARS ? "--extra-vars '${params.EXTRA_VARS}'" : ''} \
                          playbooks/${params.PLAYBOOK}.yml
                    """
                }
            }
        }
    }
}
```

---

## Topic 16: Pipeline Caching and Performance Optimisation

### Maven Dependency Cache on Kubernetes PVC

```yaml
# persistent-volume-claim.yaml — shared Maven cache across builds
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maven-cache-pvc
  namespace: jenkins
spec:
  accessModes: [ReadWriteMany]     # Must be RWX for multiple pods to share
  storageClassName: efs-sc         # AWS EFS or NFS-based storage class
  resources:
    requests:
      storage: 20Gi
```

```groovy
// Jenkinsfile using PVC cache
pipeline {
    agent {
        kubernetes {
            yaml """
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ['cat']
    tty: true
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2/repository
  volumes:
  - name: maven-cache
    persistentVolumeClaim:
      claimName: maven-cache-pvc
"""
        }
    }

    stages {
        stage('Build') {
            steps {
                container('maven') {
                    // First build: downloads deps and caches them
                    // Subsequent builds: reads from PVC cache — much faster
                    sh 'mvn clean package -DskipTests -Dmaven.repo.local=/root/.m2/repository'
                }
            }
        }
    }
}
```

### Docker Layer Caching

```groovy
pipeline {
    agent { label 'docker' }

    environment {
        REGISTRY  = 'registry.example.com'
        IMAGE     = "${REGISTRY}/myapp"
        CACHE_TAG = "${IMAGE}:cache"
    }

    stages {
        stage('Build with Cache') {
            steps {
                sh """
                    # Pull the cache image (ignore failure if it doesn't exist yet)
                    docker pull ${CACHE_TAG} || true

                    # Build using the cached layers
                    docker build \
                      --cache-from ${CACHE_TAG} \
                      --build-arg BUILDKIT_INLINE_CACHE=1 \
                      -t ${IMAGE}:${BUILD_NUMBER} \
                      -t ${CACHE_TAG} \
                      .

                    # Push the cache tag for future builds
                    docker push ${CACHE_TAG}
                    docker push ${IMAGE}:${BUILD_NUMBER}
                """
            }
        }
    }
}
```

### npm Cache Strategy

```groovy
stage('Install Dependencies') {
    steps {
        // Use the stash/unstash pattern for npm cache across stages
        script {
            def cacheKey = sh(script: 'md5sum package-lock.json | cut -d" " -f1', returnStdout: true).trim()

            // Try to restore cache
            try {
                unstash "npm-cache-${cacheKey}"
                echo "Restored npm cache for lock file ${cacheKey}"
            } catch (e) {
                echo "No cache found — installing fresh"
            }

            sh 'npm ci'

            // Save cache for future builds
            stash name: "npm-cache-${cacheKey}", includes: 'node_modules/**', allowEmpty: true
        }
    }
}
```

### Pipeline Duration Measurement and Reporting

```groovy
// Measure and report stage durations
def stageTimes = [:]

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    def start = System.currentTimeMillis()
                    sh 'mvn clean package -DskipTests'
                    stageTimes['Build'] = System.currentTimeMillis() - start
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    def start = System.currentTimeMillis()
                    sh 'mvn test'
                    stageTimes['Test'] = System.currentTimeMillis() - start
                }
            }
        }
    }

    post {
        always {
            script {
                echo "=== Pipeline Performance Report ==="
                stageTimes.each { stage, duration ->
                    echo "${stage}: ${duration/1000}s"
                }
                echo "Total: ${currentBuild.duration/1000}s"
            }
        }
    }
}
```

---

## Topic 17: Common Jenkins Anti-Patterns

This section covers mistakes you must be able to identify and correct in both real environments and in interviews.

### Anti-Pattern 1 — Running Builds on the Controller

```groovy
// WRONG: Running builds on the Jenkins controller
pipeline {
    agent any    // 'any' includes the controller — bad!
    stages {
        stage('Build') { steps { sh 'mvn clean package' } }
    }
}

// RIGHT: Set controller executors to 0, force all builds to agents
pipeline {
    agent { label 'linux-build' }    // explicit label
    stages {
        stage('Build') { steps { sh 'mvn clean package' } }
    }
}

// On controller: Manage Jenkins → System → # of executors → set to 0
```

### Anti-Pattern 2 — Hardcoded Secrets

```groovy
// WRONG: Never do this
pipeline {
    environment {
        DB_PASSWORD = 'super-secret-password-123'    // visible in logs, config, Git history
        API_KEY     = 'sk-abc123...'
    }
}

// RIGHT: Use Jenkins Credentials
pipeline {
    stages {
        stage('Deploy') {
            steps {
                withCredentials([
                    string(credentialsId: 'db-password', variable: 'DB_PASSWORD'),
                    string(credentialsId: 'api-key',     variable: 'API_KEY')
                ]) {
                    sh 'deploy.sh'    // DB_PASSWORD and API_KEY are masked in logs
                }
            }
        }
    }
}
```

### Anti-Pattern 3 — Using `latest` Image Tags on Agents

```groovy
// WRONG: 'latest' is mutable — you get different images on different days
pipeline {
    agent {
        docker { image 'maven:latest' }    // Non-reproducible!
    }
}

// RIGHT: Pin to a specific version or digest
pipeline {
    agent {
        docker { image 'maven:3.9.6-eclipse-temurin-17' }    // Reproducible
    }
}
```

### Anti-Pattern 4 — No Pipeline Timeout

```groovy
// WRONG: A hung build occupies an executor forever
pipeline {
    agent any
    stages {
        stage('Integration Test') {
            steps { sh './run-tests.sh' }    // What if this hangs?
        }
    }
}

// RIGHT: Set timeouts at pipeline and stage level
pipeline {
    agent any
    options {
        timeout(time: 30, unit: 'MINUTES')    // kill the whole pipeline
    }
    stages {
        stage('Integration Test') {
            options {
                timeout(time: 10, unit: 'MINUTES')    // kill just this stage
            }
            steps { sh './run-tests.sh' }
        }
    }
}
```

### Anti-Pattern 5 — SCM Polling Instead of Webhooks

```groovy
// WRONG: Polling wastes executor time and introduces up to 5-minute lag
triggers {
    pollSCM('H/5 * * * *')
}

// RIGHT: Use webhook triggers — instant, no wasted cycles
triggers {
    githubPush()    // Configure webhook in GitHub to push to /github-webhook/
}
```

### Anti-Pattern 6 — Copy-Paste Jenkinsfiles Across Repositories

The symptom: 50 repositories each have a 200-line Jenkinsfile that is 90% identical.  
The fix: Move the common logic to a Shared Library. Each repository's Jenkinsfile becomes:

```groovy
// RIGHT: 5-line Jenkinsfile consuming a Shared Library template
@Library('company-pipelines@main') _

buildAndDeployJavaService(
    serviceName: 'order-service',
    environment: 'production',
    notifySlack: '#deployments'
)
```

### Anti-Pattern 7 — Unlimited Build History

```groovy
// WRONG: No log rotation — disk fills up, Jenkins UI slows down
// (no options block, no log rotation configured)

// RIGHT: Always configure log rotation
pipeline {
    options {
        buildDiscarder(logRotator(
            numToKeepStr:         '30',    // keep last 30 builds
            artifactNumToKeepStr: '5'      // keep artifacts for last 5 only
        ))
    }
}
```

### Anti-Pattern 8 — Storing State in the Workspace Across Builds

```groovy
// WRONG: Assuming the previous build's files are in the workspace
stage('Deploy') {
    steps {
        sh 'cp target/myapp.jar /opt/deploy/'    // assumes jar exists from a previous step
    }
}

// RIGHT: Always build or unstash what you need; use cleanWs in post
post { always { cleanWs() } }
```

### Anti-Pattern 9 — Too Many Executors Per Agent

```groovy
// WRONG: Setting 16 executors on an agent with 4 CPU cores
// Result: builds compete for CPU, all run slower, builds starve each other

// RIGHT: executors = CPU cores - 1 (reserve one core for OS overhead)
// For a 4-core machine: 3 executors
// For a build-heavy workload: executors = 1 (one build at a time, full resources)
```

### Anti-Pattern 10 — Using Declarative Pipeline for Everything — Including Things It Can't Do Well

```groovy
// WRONG: Contorting Declarative pipeline with deep script{} nesting to do
// something that Scripted handles naturally (e.g., dynamically generated stages)

// RIGHT: Know when to use each
// Declarative: 90% of CI/CD pipelines — standard build/test/deploy with known stages
// Scripted: Dynamic stage generation, complex conditional logic, meta-pipelines
```

---

## Topic 18: Multi-Stage Docker Builds in Pipelines

```groovy
pipeline {
    agent { label 'docker' }

    environment {
        IMAGE    = "registry.example.com/myapp:${BUILD_NUMBER}"
        REGISTRY = 'registry.example.com'
    }

    stages {
        stage('Build Multi-Stage Image') {
            steps {
                sh """
                    # BuildKit for faster, cacheable multi-stage builds
                    export DOCKER_BUILDKIT=1

                    docker build \
                      --target build \
                      --cache-from ${IMAGE}-build-cache \
                      --build-arg BUILDKIT_INLINE_CACHE=1 \
                      -t ${IMAGE}-build-cache \
                      .

                    # Build the final runtime image (uses cached build stage)
                    docker build \
                      --cache-from ${IMAGE}-build-cache \
                      --cache-from ${IMAGE}:latest \
                      --build-arg BUILDKIT_INLINE_CACHE=1 \
                      -t ${IMAGE} \
                      .

                    # Push cache layers for next build
                    docker push ${IMAGE}-build-cache
                    docker push ${IMAGE}
                """
            }
        }

        stage('Extract Test Results from Build Stage') {
            steps {
                // Run tests in the build stage and extract results
                sh """
                    # Create a temp container from the build stage to extract test results
                    docker run --rm \
                      -v \$(pwd)/test-results:/results \
                      --entrypoint sh \
                      ${IMAGE}-build-cache \
                      -c "cp /app/target/surefire-reports/*.xml /results/"
                """
                junit 'test-results/*.xml'
            }
        }
    }
}
```

### Example Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build and test
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
# Pre-download dependencies (cached separately from source code)
RUN mvn dependency:go-offline -q
COPY src/ ./src/
RUN mvn clean package -DskipTests

# Stage 2: Test (separate stage so test results can be extracted)
FROM build AS test
RUN mvn test

# Stage 3: Runtime image (minimal — no build tools, no test dependencies)
FROM eclipse-temurin:17-jre-alpine AS runtime
WORKDIR /app
# Only copy the jar — not Maven, not source code, not test dependencies
COPY --from=build /app/target/*.jar app.jar
# Run as non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:8080/health || exit 1
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

---

## Topic 19: Folder and View Organisation at Scale

### Folder Hierarchy for Large Organisations

```
Jenkins/
├── _admin/                    ← admin-only jobs (seed, backup, health checks)
│   ├── seed-job
│   └── jenkins-backup
├── platform/                  ← platform team pipelines
│   ├── infra-terraform/
│   └── k8s-cluster-upgrade/
├── team-payments/             ← team-scoped folder
│   ├── auth-service/
│   ├── payment-service/
│   └── fraud-detection/
├── team-orders/
│   ├── order-service/
│   └── inventory-service/
└── shared-libs/               ← pipelines for testing shared libraries
    └── jenkins-shared-lib/
```

### Folder-Level Credentials (JCasC)

```yaml
# Credentials scoped to a folder — only jobs inside that folder can use them
# Configure via: folder → Credentials → Add Credentials

# JCasC cannot directly scope folder credentials, but the REST API can:
# POST http://jenkins/job/team-payments/credentials/store/folder/domain/_/createCredentials
```

```groovy
// Script Console: Create folder-scoped credential programmatically
import com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl
import com.cloudbees.plugins.credentials.CredentialsScope
import jenkins.model.Jenkins

def folder = Jenkins.instance.getItem('team-payments')
def store  = folder.getProperties().get(
    com.cloudbees.plugins.credentials.FolderCredentialsProperty.class
).getStore()

def cred = new UsernamePasswordCredentialsImpl(
    CredentialsScope.GLOBAL,
    'payments-db-creds',
    'Payments database credentials',
    'payments_user',
    'secret-password'
)

store.addCredentials(
    com.cloudbees.plugins.credentials.domains.Domain.global(),
    cred
)
println "Credential added to folder: team-payments"
```

### Folder-Level RBAC

```groovy
// Using Role Strategy Plugin: grant developer role only on a specific folder
import org.jenkinsci.plugins.rolestrategy.*
import com.michelin.rolestrategy.RoleStrategy

def strategy = Jenkins.instance.authorizationStrategy as RoleStrategy

// Create a project role that only applies to the team-payments folder
def paymentsRole = new Role(
    "payments-developer",           // role name
    "team-payments/.*",             // pattern — only matches jobs in this folder
    RoleType.Project,
    [
        "hudson.model.Item.Build",
        "hudson.model.Item.Read",
        "hudson.model.Item.Workspace"
    ]
)

strategy.addRole(RoleType.Project, paymentsRole)
strategy.assignRole(RoleType.Project, "payments-dev-user", "payments-developer")
Jenkins.instance.save()
```

---

## Topic 20: Windows Agent Specifics

```groovy
pipeline {
    agent none

    stages {
        stage('Build on Linux') {
            agent { label 'linux' }
            steps {
                sh 'mvn clean package'
                stash name: 'linux-artifact', includes: 'target/*.jar'
            }
        }

        stage('Build on Windows') {
            agent { label 'windows' }
            steps {
                // bat for Windows Command Prompt
                bat 'mvn clean package'

                // powershell for more powerful scripting
                powershell '''
                    $version = (Get-Content pom.xml | Select-String -Pattern "<version>(.+?)</version>").Matches[0].Groups[1].Value
                    Write-Host "Building version: $version"
                    mvn clean package -DskipTests
                '''

                // Windows path syntax in workspace
                bat "copy target\\*.jar C:\\jenkins\\artifacts\\"

                stash name: 'windows-artifact', includes: 'target\\*.jar'
            }
        }

        stage('Test on Windows') {
            agent { label 'windows' }
            steps {
                unstash 'windows-artifact'

                powershell '''
                    # Run .NET tests
                    dotnet test tests/ --logger "trx;LogFileName=results.trx" --results-directory TestResults

                    # Check exit code
                    if ($LASTEXITCODE -ne 0) {
                        Write-Error "Tests failed"
                        exit $LASTEXITCODE
                    }
                '''

                // Publish Windows test results
                mstest testResultsFile: 'TestResults/**/*.trx'
            }
        }
    }
}
```

### Windows-Specific Considerations

```groovy
// Path separator: use \\ in strings (or / works in most cases on modern Windows)
// Environment variables: %VAR% in bat, $env:VAR in powershell, ${VAR} in Groovy

// Credential for Windows agent domain join
withCredentials([usernamePassword(
    credentialsId: 'windows-domain-creds',
    usernameVariable: 'DOMAIN_USER',
    passwordVariable: 'DOMAIN_PASS'
)]) {
    bat "net use Z: \\\\fileserver\\share /user:DOMAIN\\%DOMAIN_USER% %DOMAIN_PASS%"
}

// PowerShell with error handling
powershell '''
    $ErrorActionPreference = "Stop"    # Make PowerShell treat errors as exceptions

    try {
        Invoke-WebRequest -Uri "http://internal-api/health" -UseBasicParsing
        Write-Host "Health check passed"
    } catch {
        Write-Error "Health check failed: $_"
        exit 1
    }
'''
```

---

## Topic 21: Advanced Build Parameterisation

### Dynamic Choice Parameter (ActiveChoices Plugin)

```groovy
// The ActiveChoices plugin allows parameters to populate dynamically at runtime
// Install: Active Choices Plugin

// In Job Configuration → Add Parameter → Active Choices Parameter:
//   Name: IMAGE_TAG
//   Groovy Script:
/*
import groovy.json.JsonSlurper

def response = "curl -s 'http://nexus/service/rest/v1/search?repository=docker&name=myapp&sort=version&direction=desc&limit=20'".execute().text
def tags = new JsonSlurper().parseText(response).items.collect { it.version }
return tags ?: ['latest']
*/
```

### Upstream Parameter Propagation

```groovy
// Pass parameters from build job to deploy job using the Parameterized Trigger Plugin

stage('Trigger Deploy') {
    steps {
        build(
            job: '../deploy/deploy-to-staging',
            parameters: [
                string(name: 'IMAGE_TAG',    value: env.BUILD_NUMBER),
                string(name: 'GIT_COMMIT',   value: env.GIT_COMMIT),
                string(name: 'APP_VERSION',  value: env.APP_VERSION),
                booleanParam(name: 'RUN_SMOKE_TESTS', value: true)
            ],
            wait: true,           // wait for downstream to complete
            propagate: true       // fail this build if downstream fails
        )
    }
}
```

### Rebuild Plugin Usage

```groovy
// The Rebuild plugin adds a "Rebuild" button to each build
// that re-runs it with the same parameters — install:
// Manage Jenkins → Plugins → search "Rebuild"

// No pipeline code needed — it's a UI feature.
// Useful for re-running a failed production deployment with identical parameters
// without having to remember and re-enter them.
```

---

## Topic 22: Jenkins in a GitOps Workflow (ArgoCD / Flux)

### The Pattern

In a Kubernetes GitOps workflow, Jenkins handles CI (build, test, scan, push image) and ArgoCD or Flux handles CD (watch a Git repository for changes and sync them to the cluster). They connect via Git — Jenkins updates the image tag in the Kubernetes manifests repository, ArgoCD detects the change and deploys.

```
Developer pushes code
        │
        ▼
Jenkins Pipeline (CI)
  ├── Build & test application
  ├── Security scans (SAST, SCA, container scan)
  ├── Build Docker image
  ├── Push image to registry (tag = build number / git SHA)
  └── Update image tag in Git manifests repo (k8s-configs/)
                │
                ▼
       ArgoCD detects change
       in k8s-configs/
                │
                ▼
       Syncs to Kubernetes cluster
       (applies the updated Deployment manifest)
```

### Jenkins Pipeline — GitOps Integration

```groovy
pipeline {
    agent any

    environment {
        APP_NAME       = 'order-service'
        REGISTRY       = 'registry.example.com'
        IMAGE          = "${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
        MANIFESTS_REPO = 'https://github.com/myorg/k8s-configs.git'
        MANIFESTS_DIR  = 'apps/order-service'
    }

    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Build & Push Image') {
            steps {
                sh "docker build -t ${IMAGE} ."
                sh "docker push ${IMAGE}"
            }
        }

        stage('Update GitOps Manifests') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GH_USER',
                    passwordVariable: 'GH_TOKEN'
                )]) {
                    sh """
                        # Clone the manifests repository
                        git clone https://\${GH_USER}:\${GH_TOKEN}@github.com/myorg/k8s-configs.git /tmp/k8s-configs
                        cd /tmp/k8s-configs

                        # Update the image tag using yq (YAML processor)
                        yq e ".spec.template.spec.containers[0].image = \\"${IMAGE}\\"" \
                          -i ${MANIFESTS_DIR}/deployment.yaml

                        # Or use kustomize edit
                        cd ${MANIFESTS_DIR}
                        kustomize edit set image ${REGISTRY}/${APP_NAME}=${IMAGE}
                        cd /tmp/k8s-configs

                        # Commit and push the change
                        git config user.email "jenkins@myorg.com"
                        git config user.name  "Jenkins CI"
                        git add ${MANIFESTS_DIR}/
                        git commit -m "ci: update ${APP_NAME} to build ${BUILD_NUMBER} [skip ci]"
                        git push origin main
                    """
                }
            }
        }

        stage('Wait for ArgoCD Sync') {
            when { branch 'main' }
            steps {
                withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {
                    sh """
                        # Wait for ArgoCD to sync the application
                        argocd app wait ${APP_NAME} \
                          --server argocd.example.com \
                          --auth-token \${ARGOCD_TOKEN} \
                          --sync \
                          --health \
                          --timeout 300

                        echo "ArgoCD sync complete — ${APP_NAME} is healthy"
                    """
                }
            }
        }
    }
}
```

---

## Topic 23: Build Status Badges and README Integration

```markdown
<!-- In your repository README.md -->

## Build Status

| Pipeline | Status |
|---|---|
| Main Build | [![Build Status](https://jenkins.example.com/buildStatus/icon?job=myapp%2Fmain)](https://jenkins.example.com/job/myapp/job/main/) |
| Staging Deploy | [![Staging](https://jenkins.example.com/buildStatus/icon?job=myapp%2Fdeploy-staging)](https://jenkins.example.com/job/myapp/job/deploy-staging/) |
| Security Scan | [![Security](https://jenkins.example.com/buildStatus/icon?job=myapp%2Fsecurity)](https://jenkins.example.com/job/myapp/job/security/) |
```

```bash
# The Embeddable Build Status plugin generates these badge URLs
# Install: Embeddable Build Status Plugin

# Badge URL format:
# https://jenkins.example.com/buildStatus/icon?job=<folder>%2F<job-name>

# Shield.io alternative (fetches from Jenkins API):
# https://img.shields.io/jenkins/build?jobUrl=https%3A%2F%2Fjenkins.example.com%2Fjob%2Fmyapp%2F
```

---

## Topic 24: Plugin Dependency Management

### Plugin BOM (Bill of Materials)

```groovy
// In a custom Jenkins Docker image, pin plugin versions using the BOM
// This prevents incompatible plugin combinations

// Dockerfile
FROM jenkins/jenkins:2.440-lts

# Use the plugin installation manager
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

// plugins.txt — pin ALL plugin versions explicitly
// Format: plugin-name:version
git:5.2.1
workflow-aggregator:596.v8c21c963d92d
credentials:1319.v7eb_51b_3a_c97b_
kubernetes:4029.v5712230ccb_f8
docker-workflow:572.v950f58993843
pipeline-utility-steps:2.16.2
blueocean:1.27.9
sonar:2.17.2
hashicorp-vault-plugin:360.v0a_1c04cf807d
role-strategy:689.v731678c3e0eb_
configuration-as-code:1775.v810dc950b_514
```

```bash
# Check for conflicting plugin versions
jenkins-plugin-cli --plugin-file plugins.txt \
  --output yaml \
  --list | grep -E "(conflict|incompatible)"

# Find the minimal set of required plugins (removes redundant entries)
jenkins-plugin-cli --plugin-file plugins.txt \
  --output minimal-plugins.txt
```

---

## Topic 25: Jenkins with OpenTelemetry

```groovy
// Install: OpenTelemetry Plugin
// Configure: Manage Jenkins → System → OpenTelemetry
// Endpoint: http://otel-collector:4317 (gRPC) or http://otel-collector:4318 (HTTP)

// The plugin automatically instruments:
// - Pipeline stage durations (as OTEL spans)
// - Build queue time
// - Agent provisioning time
// - Test results

// Trace data flows to: Jaeger, Zipkin, Grafana Tempo, Elastic APM, Datadog

// Additional manual instrumentation in Jenkinsfile:
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                withSpan(name: 'custom-build-span', attributes: [
                    'app.name': 'myapp',
                    'app.version': env.BUILD_NUMBER
                ]) {
                    sh 'mvn clean package'
                }
            }
        }
    }
}

// Dashboard in Grafana: view P50/P95 build durations per stage, per branch,
// per agent — identify which stage is getting slower over time
```

---

## Topic 26: Throttle Concurrent Builds and Lockable Resources

### Throttle Concurrent Builds Plugin

```groovy
// Prevent a job from running more than N concurrent instances
// Useful to protect downstream systems (test databases, rate-limited APIs)

pipeline {
    options {
        // Maximum 2 concurrent builds of this pipeline across all branches
        throttleJobProperty(
            categories: [],
            limitOneJobWithMatchingParams: false,
            maxConcurrentPerNode: 1,
            maxConcurrentTotal: 2,
            paramsToUseForLimit: '',
            throttleEnabled: true,
            throttleOption: 'project'
        )
    }

    stages {
        stage('Integration Test') { steps { sh 'mvn verify' } }
    }
}
```

### Lockable Resources Plugin

```groovy
// Manage Jenkins → Lockable Resources → Add Resource
// Name: staging-database
// Labels: staging-db

pipeline {
    agent any

    stages {
        stage('Integration Tests') {
            steps {
                // Acquire exclusive lock on the shared resource
                lock(resource: 'staging-database', inversePrecedence: true) {
                    echo "Acquired exclusive lock on staging database"
                    sh 'mvn verify -Pintegration-tests -Ddb.url=staging-db'
                }
                // Lock released here — next build can proceed
            }
        }
    }
}

// Lock multiple resources simultaneously (prevents deadlocks via ordered acquisition)
lock(label: 'staging-db', quantity: 1, variable: 'LOCKED_RESOURCE') {
    echo "Locked: ${env.LOCKED_RESOURCE}"
    sh "run-tests.sh --database=${env.LOCKED_RESOURCE}"
}
```

---

## Topic 27: Parameterised Trigger Plugin

```groovy
// Trigger downstream jobs with parameters from the current build
// Install: Parameterized Trigger Plugin

stage('Trigger Downstream Deployments') {
    steps {
        // Trigger multiple jobs in parallel with different parameters
        parallel(
            'Deploy to EU': {
                build(
                    job: 'deploy-service',
                    parameters: [
                        string(name: 'IMAGE_TAG', value: env.BUILD_NUMBER),
                        string(name: 'REGION',    value: 'eu-west-1'),
                        string(name: 'NAMESPACE', value: 'production-eu')
                    ],
                    wait: true,
                    propagate: true
                )
            },
            'Deploy to US': {
                build(
                    job: 'deploy-service',
                    parameters: [
                        string(name: 'IMAGE_TAG', value: env.BUILD_NUMBER),
                        string(name: 'REGION',    value: 'us-east-1'),
                        string(name: 'NAMESPACE', value: 'production-us')
                    ],
                    wait: true,
                    propagate: true
                )
            }
        )
    }
}

// Trigger without waiting (fire and forget)
stage('Notify QA Pipeline') {
    steps {
        build(
            job: 'qa-regression-suite',
            parameters: [
                string(name: 'BUILD_TO_TEST', value: env.BUILD_NUMBER),
                string(name: 'APP_URL', value: "https://staging.example.com")
            ],
            wait: false    // Do not wait — continue immediately
        )
    }
}
```

---

## Topic 28: Master Interview Q&A — All Advanced Topics

### REST API and Automation

**Q: How do you trigger a Jenkins build remotely and retrieve the build result in a shell script?**
A: Trigger using `curl -X POST -u user:token <jenkins-url>/job/<name>/buildWithParameters?PARAM=value`. Capture the `Location` header from the 201 response — it contains the queue item URL. Poll the queue URL until `executable.url` is populated (the build has started). Then poll `<build-url>/api/json` and check the `result` field — it is `null` while running and `SUCCESS/FAILURE/ABORTED` when done. Exit the script with a non-zero code if result is not `SUCCESS`.

### Job DSL

**Q: What happens to jobs generated by Job DSL when their DSL script is deleted from Git?**
A: The seed job still has the previous run's output, but the next time the seed job runs with `removedJobAction: DELETE`, any jobs not declared in the current DSL will be removed. This is why you start with `IGNORE` or `DISABLE` in non-production environments until you trust the DSL, and use `DELETE` in production once the DSL is stable. Use version control history to audit which jobs were removed and when.

### Jenkinsfile Linting

**Q: A developer says their Jenkinsfile looks correct but the pipeline always fails at startup. How do you debug this?**
A: Three steps: (1) Use the Jenkins declarative linter endpoint: `curl -X POST -F "jenkinsfile=<Jenkinsfile" http://jenkins/pipeline-model-converter/validate` — it reports syntax errors that are not obvious from reading. (2) Check the Jenkins system log for detailed error messages on job load. (3) Look for common issues: missing `pipeline {}` wrapper, unclosed braces, using unsupported directives inside a `parallel` block, or a `when` condition that references a variable not yet in scope.

### Pipeline Unit Testing

**Q: How do you test a Shared Library step that sends a Slack notification, without actually sending notifications during tests?**
A: In JenkinsPipelineUnit, register an allowed method mock: `helper.registerAllowedMethod('slackSend', [Map.class], { Map args -> println "MOCK slackSend: ${args}" })`. This intercepts the `slackSend` call and records it in the `helper.callStack` without executing it. In your assertion, verify the mock was called with the correct arguments: `assertTrue(helper.callStack.findAll { it.methodName == 'slackSend' }.any { it.args[0].channel == '#deployments' })`.

### Tool Comparison

**Q: A startup wants to set up CI/CD from scratch today — would you recommend Jenkins or GitHub Actions?**
A: For a startup with no existing Jenkins infrastructure, GitHub Actions is usually the better starting point. It has zero operational overhead, no servers to manage, tight GitHub integration, a massive ecosystem of pre-built actions, and free minutes for public repos. Jenkins makes sense if: the startup works in a regulated industry that cannot use cloud-hosted build runners, has complex multi-system integrations that Jenkins plugins already solve, or has engineers who already know Jenkins well. The architectural decision should be driven by operational capacity and compliance requirements, not tool preference.

### Groovy Sandbox

**Q: A pipeline script is rejected with "Script not permitted to use method groovy.lang.GroovyShell evaluate". Should you approve it?**
A: Almost certainly not. `GroovyShell.evaluate()` executes arbitrary Groovy code from a string at runtime — this is effectively eval() and is a serious security risk. A legitimate pipeline should not need to call this. Push back on the engineer and find out why they need dynamic code evaluation. The correct pattern is almost always to refactor the logic into a proper Shared Library method. Approving `GroovyShell.evaluate` in a shared Jenkins instance effectively gives pipeline authors the ability to run any code on the Jenkins controller.

### Deployment Strategies

**Q: Your Blue-Green deployment succeeded in switching traffic but you get reports of errors 10 minutes later. What do you check?**
A: (1) Check if the errors started exactly when the switch happened — this points to a regression in the new version. (2) Check if both Blue and Green are still running — some errors could come from cached DNS or long-lived connections still going to Blue. (3) Review application logs on the Green pods for exceptions. (4) Check database migration compatibility — if the new version applied a schema migration that is not backward-compatible, the old version's in-flight transactions may fail. (5) Initiate rollback immediately (switch the load balancer back to Blue) while you diagnose. Never debug in production while users are hitting errors.

### SBOM and Image Signing

**Q: What is an SBOM and why is it increasingly required in regulated industries?**
A: An SBOM (Software Bill of Materials) is a machine-readable inventory of all components, libraries, and dependencies that make up a software artifact — analogous to an ingredients list on packaged food. It allows organisations to rapidly answer "are we using log4j 2.14.1?" when a critical CVE is disclosed, without manually checking every service. US Executive Order 14028 (2021) mandates SBOMs for software sold to the US federal government. NIST, SOC 2, and PCI-DSS auditors increasingly require them. Generating SBOMs in Jenkins using Syft or the CycloneDX Maven plugin and attaching them as build artifacts (or as cosign attestations to images) satisfies this requirement.

### Terraform Pipeline

**Q: Why do you need a manual approval gate between `terraform plan` and `terraform apply` in a Jenkins pipeline?**
A: The Terraform plan is a diff — it shows exactly what will be created, modified, and destroyed. For infrastructure changes, the impact can be severe: accidentally destroying a production database, opening a security group to the internet, or deleting a Kubernetes node group. A human must review the plan output before `apply` executes. The `input` step in Jenkins pauses the pipeline and records who approved and when — this creates an audit trail. For low-risk environments (dev), you might auto-approve, but staging and production always require human sign-off.

### Anti-Patterns

**Q: You've joined a new company and their Jenkins has executors running on the controller. Is this a problem?**
A: Yes. Running builds on the Jenkins controller is an anti-pattern for three reasons: (1) Security — the controller has access to all secrets, credentials, and the Jenkins internal API; a compromised build job on the controller can exfiltrate everything. (2) Stability — a resource-hungry build can starve the controller of CPU and memory, causing the Jenkins UI to become unresponsive and scheduled jobs to queue. (3) Reliability — if a build crashes the JVM or fills the disk, it takes the entire Jenkins instance down, not just the agent. Set controller executors to 0 and route all builds to dedicated agents.

### GitOps Integration

**Q: How does a Jenkins CI pipeline integrate with ArgoCD for GitOps-based deployments?**
A: Jenkins handles CI — it builds, tests, scans, and pushes the container image to a registry. After pushing, Jenkins clones the separate GitOps manifests repository (usually maintained by the platform team), updates the image tag in the Kubernetes Deployment manifest or Kustomize file using `yq` or `kustomize edit set image`, commits, and pushes. ArgoCD watches that repository, detects the new commit, and syncs the cluster to match the desired state. Jenkins can optionally poll the ArgoCD API after the commit to wait for the sync to complete and verify the deployment is healthy before marking the pipeline as successful.

### Performance

**Q: Your Maven builds are taking 20 minutes. Half the time is downloading dependencies. How do you fix this?**
A: Three solutions in order of impact: (1) Mount a shared Maven repository on a PersistentVolumeClaim (if using Kubernetes agents) so the `.m2/repository` is shared across all builds — dependency downloads happen once, subsequent builds read from cache. (2) Use Nexus or Artifactory as a Maven proxy — configure your `settings.xml` to point to the proxy, which caches dependencies from Maven Central on first pull and serves them locally on subsequent requests. (3) Run `mvn dependency:go-offline` as a separate Docker image layer in the Dockerfile so that base dependencies are cached in the image layer and only application-specific changes cause new downloads.
