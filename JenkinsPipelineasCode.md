# Jenkins Pipeline as Code (Declarative Pipeline)

## Project Overview
This project demonstrates how to write Jenkins pipelines as code using a **Jenkinsfile**.
Pipeline as Code allows you to define your entire CI/CD workflow in a version-controlled file
living alongside your application source code.

### Objectives
1. Understand Declarative vs Scripted Pipeline syntax
2. Create a Jenkinsfile for a Maven project
3. Use environment variables, parameters and credentials
4. Implement parallel stages
5. Configure post-build actions (notifications, cleanup)

---

## Prerequisites
- Jenkins installed and running (see JenkinsManualPipeline.md for setup)
- Maven Integration plugin installed
- Git plugin installed
- A forked copy of the [sysfoo](https://github.com/udbc/sysfoo) repository

---

## Step 1: Create a Jenkinsfile in Your Repository

A **Jenkinsfile** is a text file containing the definition of a Jenkins Pipeline,
checked into source control alongside your application code.

1. In the root of your forked `sysfoo` repository, create a new file named `Jenkinsfile`.
2. Add the following basic declarative pipeline:

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven'   // Must match the name in Global Tool Configuration
    }

    environment {
        APP_NAME = 'sysfoo'
        VERSION  = "1.0.${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/sysfoo.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
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
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
    }

    post {
        success { echo "Build ${VERSION} succeeded!" }
        failure { echo "Build ${VERSION} failed." }
        always  { cleanWs() }
    }
}
```

---

## Step 2: Connect Jenkins to Your Repository

1. From the Jenkins dashboard, click **New Item**.
2. Enter a name (e.g. `sysfoo-pipeline`) and select **Pipeline**, then click **OK**.
3. Under **Pipeline Definition**, select **Pipeline script from SCM**.
4. Set **SCM** to **Git** and enter your repository URL.
5. Set the **Branch Specifier** to `*/main`.
6. Leave the **Script Path** as `Jenkinsfile` (default).
7. Click **Save** then click **Build Now** to run your first pipeline build.

---

## Step 3: Add Build Parameters

Parameters allow you to pass values into your pipeline at runtime.

```groovy
pipeline {
    agent any

    parameters {
        string(
            name: 'DEPLOY_ENV',
            defaultValue: 'dev',
            description: 'Target environment (dev | staging | prod)'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Check to skip unit tests'
        )
    }

    tools { maven 'Maven' }

    stages {

        stage('Build') {
            steps { sh 'mvn clean compile' }
        }

        stage('Test') {
            when {
                expression { return !params.SKIP_TESTS }
            }
            steps { sh 'mvn test' }
        }

        stage('Package') {
            steps { sh 'mvn package -DskipTests' }
        }

        stage('Deploy') {
            steps {
                echo "Deploying to: ${params.DEPLOY_ENV}"
            }
        }
    }
}
```

After saving, your pipeline will show **Build with Parameters** instead of **Build Now**.

---

## Step 4: Use Credentials Securely

Never hard-code secrets in your Jenkinsfile. Store them in Jenkins Credentials.

1. Go to **Manage Jenkins → Credentials → System → Global credentials**.
2. Click **Add Credentials** → **Username with password**.
3. Set ID to `docker-hub` and fill in your Docker Hub username and password.

```groovy
pipeline {
    agent any

    environment {
        DOCKER_CREDS = credentials('docker-hub')
    }

    stages {
        stage('Docker Login') {
            steps {
                sh 'echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin'
            }
        }

        stage('Docker Build and Push') {
            steps {
                sh """
                    docker build -t ${DOCKER_CREDS_USR}/sysfoo:${BUILD_NUMBER} .
                    docker push ${DOCKER_CREDS_USR}/sysfoo:${BUILD_NUMBER}
                """
            }
        }
    }

    post {
        always { sh 'docker logout' }
    }
}
```

---

## Step 5: Parallel Stages

Run independent stages simultaneously to reduce total build duration.

```groovy
pipeline {
    agent any
    tools { maven 'Maven' }

    stages {

        stage('Build') {
            steps { sh 'mvn clean compile' }
        }

        stage('Quality Checks') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'mvn test' }
                    post {
                        always { junit '**/target/surefire-reports/*.xml' }
                    }
                }
                stage('Code Style') {
                    steps { sh 'mvn checkstyle:check' }
                }
                stage('Dependency Scan') {
                    steps { sh 'mvn dependency:tree' }
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
    }
}
```

---

## Step 6: Trigger Pipeline from GitHub Webhook

1. In your GitHub repository go to **Settings → Webhooks → Add webhook**.
2. Set **Payload URL** to: `http://<your-jenkins-ip>:8080/github-webhook/`
3. Set **Content type** to `application/json`.
4. Choose **Just the push event** and click **Add webhook**.

In your Jenkins pipeline job:
1. Go to **Configure → Build Triggers**.
2. Check **GitHub hook trigger for GITScm polling**.
3. Save.

Now every push to the repository will automatically trigger a new pipeline run.

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `No such DSL method` | Plugin not installed | Install the required plugin |
| Pipeline skips stages | `when` condition not met | Check branch name or expression |
| Credentials not found | Wrong credential ID | Verify ID in Manage Jenkins → Credentials |
| Maven not found | Tool name mismatch | Match name in `tools{}` to Global Tool Configuration |
| Workspace dirty on next build | No `cleanWs()` in post | Add `cleanWs()` in `post { always { } }` |

---

## Additional Resources
- [Jenkins Declarative Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Pipeline Steps Reference](https://www.jenkins.io/doc/pipeline/steps/)
- [Using Credentials in Pipeline](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#handling-credentials)
