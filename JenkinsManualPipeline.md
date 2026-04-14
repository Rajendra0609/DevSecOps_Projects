# Build a CICD pipeline manually with Jenkins
### Project overview
The following are the objectives of this project:
1. **Setup Jenkins along with Docker Server (DIND) as your Continuous Integration Environment**: configure plugins, admin user and get Jenkins ready to start building your CI workflows. This setup will allow you to leverage Docker-in-Docker (DIND) to manage and run your builds within isolated containers, ensuring a clean and consistent environment for every build.
2. **Create a pipeline for a Maven project**: Manually develop a Jenkins pipeline for build, test, and deployment processes of a Maven project. This step will help you understand the steps and configurations required to set up a pipeline from scratch.
3. **Configure artifact versioning**: Implement a versioning strategy for your build artifacts. This ensures that each build is uniquely identifiable, facilitating easier tracking, rollback, and deployment of specific versions.
4. **Connecting jobs with Upstream and Downstream**: Link various jobs to create a dependency chain, where the output of one job can trigger the next in sequence. This allows for complex workflows and better coordination between different stages of your CI/CD pipeline.
5. **Create a pipeline view for your pipeline**: Visualize your entire CI/CD process with a pipeline view. This provides a clear and comprehensive overview of the build stages, job statuses, and helps in identifying bottlenecks or failures quickly.
# Step 1
## Set up Jenkins server Docker(DIND)
There are various options for setting up a Jenkins environment, such as setting it up on Windows or Linux-based systems, or on a virtual machine (VM). In this project, I will be setting up Jenkins using Docker. This option offers the flexibility and convenience of a containerized environment, allowing you to continuously use your environment for as long as you want for your CI/CD practice.

To get started, you'll need to have Docker Engine installed on your local machine or VM. Once Docker is installed, you can set up the Jenkins server inside a Docker container. This approach simplifies the setup process and ensures that your Jenkins environment is consistent and easily replicable.
1. To install the Docker engine go to [Get Docker](https://docs.docker.com/get-docker/) and select the best installation path for you.
2. Start by validating your environment with:
```
docker version
docker-compose
```
The first command will show you both client and server information and the second command will return the help menu for docker-compose, which validates that you have Docker installed, started and
docker-compose ready to go.

3.  Launch a Jenkins container on your docker host by using the recommended Jenkins image similar to what is recommended in the official documentation [Docker](https://www.jenkins.io/doc/book/installing/docker/) using the following sequence of commands.
```
git clone https://github.com/udbc/bootcamp.git
cd bootcamp/jenkins
docker-compose up -d
```
The first command clones the repository that will help you set up your environment quickly. It provides a declarative approach to setting up the environment you need. The next command changes your directory to where your Jenkins folder is located. When you run the third command, it will read the Dockerfile and build the custom Jenkins image with all the required plugins (including Blue Ocean) and configurations.

4. Validate that Jenkins is set up along with Docker using the following command:
```
docker-compose ps
```
If all goes well you should see your running containers (Jenkins and Docker:dind)
![Screenshot 2024-06-07 152005](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/8454e410-a97a-45fd-88c4-f2a7d8fba3d2)

5. You can now access Jenkins UI by browsing to `http://localhost:8080`
![Screenshot 2024-06-07 152128](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/0125dfd1-86cb-4676-a2d0-12e10f2c0219)

6. To access Jenkins UI, you will need to provide the initial admin password. To get the admin password run the following command:
```
docker-compose logs jenkins
```
From the output copy the admin password as highlighted in the image below and paste in the Jenkins UI for access

![Screenshot 2024-06-07 152049](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/21d093c7-fa28-4d98-af5c-6a435848640f)

7. Next, choose "install suggested plugins" to configure the default plugins automatically. Once plugins have been successfully installed, create your Jenkins Admin user. This would provide you with your Jenkins configuration page.

![Screenshot 2024-06-07 152648](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/83e3070e-15ac-418f-bf56-835b219a88a8)

![Screenshot 2024-06-07 152825](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/7a148e74-a288-4cfd-b3bd-bc6b61241e8c)

## Starting, stopping and resetting Jenkins
When setting up your continuous Integration workflows, there will be instances where you need to pause and resume your environment, such as when shutting down your laptop at the end of the day and restarting it the next day. To ensure your learning lab environment is up and running again, you can gracefully stop the lab environment by using specific commands:
```
docker-compose stop
```
You can restart your environment by using:
```
docker-compose up -d
```

# Step 2
## Set up a pipeline for Maven project.

In this step, I will begin the process of building a pipeline for a Java application that uses Maven as a build tool. Before starting to build the pipeline, first fork the GitHub repository of the application code [here](https://github.com/udbc/sysfoo)

### Set up Build Job

1. Go to *Manage Jenkins* then click on *Global tools* configuration and under *Maven* section, provide the name as **Maven** and select the Maven version
**3.9.6** then save the changes.

![Screenshot 2024-06-09 061346](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/e62a9f19-dff2-4ff0-962a-0521748c5794)

2. Next, install **Maven integration plugin**. Go to **Manage Jenkins** then click on **Manage Plugins**, **Available plugins**, search for **Maven Integration plugin** and install it without restart. Once installed click on **Go back to the top page** to return to Jenkins' main page.
3. From the Jenkins main page, create a new folder that will serve as a namespace. Do so by creating a **new item**, with type **folder** with name **sysfoo** then click **ok**.
4. On the next page, leave all other configurations as is and click save to proceed. That creates a folder by name sysfoo.

![Screenshot 2024-06-09 062049](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/a07d037d-5221-44f5-845e-66e464628f97)

5. Inside the sysfoo folder, create a new job and select **Maven project** as the type of project. Name the job as **build**
6. Next, go to source code management, enter the URL of your git repository that you forked from GitHub for this project, then change the **branch specifier** to `*/main`

![Screenshot 2024-06-09 062812](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/f7a8701b-5007-4845-a209-dc36031932af)

7. Click on **build** on the left hand panel, it would take you to the build configurations. Provide your path to *pom.xml*. In this example the file is inside the root directory of sysfoo, so there is nothing to change. Go to build step and provide the goal and option as *compile*.

![Screenshot 2024-06-14 124803](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/ebffb65e-fd99-49fa-a923-bcec0707edb5)

8. Save the job and click on **Build Now**. Observe the job status, console output etc.

### Adding Unit test and Packaging jobs
After creating the Build job, its time to add Test and Package jobs for our sysfoo application.
1. Start by creating a new job inside sysfoo folder and name it Test.
2. Scroll down to the bottom of the page, in the *Copy from* field, enter *build* this will ensure that everything configured in the build job is automatically reflected in the test job.

![Screenshot 2024-06-11 075245](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/8c83e785-4938-43e7-8020-de8a42901497)

3. In the test job, change your description, in this example, I set it as *test sysfoo java app*. Leave the *Source code management* repository as is.
4. Under the build step, change the goals and option to *clean test* and leave the other fields as is. Save the job and build.

The next job is package, this job will compile the application and then generate the *war file* that can be deployed on a server or container. 
1. Create a job in the same folder with the name of package, and copy configurations from the test job.

![Screenshot 2024-06-11 075657](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/621204f3-206d-4c67-a3f8-cd8e7735feaf)

2. Update the description as *package sysfoo java app*, create jar and the only change to make in the configuration is in build step.
3. In the build step for the package job, change the goal to *package -DskipTests* , save the changes and build.

![Screenshot 2024-06-11 075842](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/65ff7b32-19ea-4b4b-8404-71e33b904db4)

4. If the build is successful, it will create a *jar* file, you will find the jar file created in the *Workspace* target directory. Verify the jar file was created by clicking on workspace

![Screenshot 2024-06-11 080731](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/87c92b6b-d2f3-4969-804a-7a13d32bd61f)

5. You can publish or archive the jar file for future use. Configure the package from the **post build actions**, choose **archive the artifacts** and provide path `**target/*.jar` in the files to archive input field so that the jar file created at this path (e.g. sysfoo.war) is automatically archived/published.

![Screenshot 2024-06-15 103843](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/315fd76e-78be-4647-bccc-50fc7879ab30)

6. Save the changes and build the job. Once the build is successful, check the project page to find your artifacts right there.

![Screenshot 2024-06-11 082600](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/7efc0f7f-42e1-4728-a470-84ad909cafb5)

# Step 3
## Configure Artifact Versioning

Artifact versioning ensures every build produces a uniquely identifiable artifact. This makes it straightforward to track exactly what is deployed in each environment, roll back to a specific build, and audit release history.

### Why Versioning Matters
Without versioning, successive builds overwrite the same artifact file. If a defect is found in production you cannot reliably identify or restore the previously-working binary. A solid versioning strategy also enables traceability from a running container or WAR file all the way back to the Git commit that produced it.

### Versioning Strategy Options

| Strategy | Pattern | When to Use |
|---|---|---|
| Build number | `1.0.${BUILD_NUMBER}` | Simple projects, quick setup |
| Semantic version | `MAJOR.MINOR.PATCH` from pom.xml | Maven/Gradle projects following SemVer |
| Git commit SHA | `1.0.0-${GIT_COMMIT[0..7]}` | Strong traceability requirement |
| Timestamp | `1.0.0-20240618-153045` | Date-ordered artifacts |
| Hybrid (recommended) | `${VERSION}-${BUILD_NUMBER}-${SHORT_SHA}` | Enterprise — combines all three benefits |

### Approach 1 — Build-Number Versioning

This is the simplest approach and appropriate when Maven's pom.xml version is already managed by your release process.

1. Open the **package** job configuration page.
2. Under the **Build** step, change the Maven goal to:
   ```
   package -DskipTests -Drevision=${BUILD_NUMBER}
   ```
3. In pom.xml, use `${revision}` as the version placeholder:
   ```xml
   <version>1.0.${revision}</version>
   ```
4. The produced artifact will now be named `sysfoo-1.0.42.jar` (where 42 is the Jenkins build number), making each build uniquely identifiable.

### Approach 2 — Git Commit SHA Versioning (Recommended for Traceability)

This approach ties every artifact directly to the exact source commit.

1. From the Jenkins main page, go to **Manage Jenkins → System** and add a global environment variable:
   - **Name**: `GIT_COMMIT_SHORT`
   - This is computed per-build using the shell step below.

2. In the **package** job, add a **Pre-build Execute Shell** step before the Maven step:
   ```bash
   #!/bin/bash
   SHORT_SHA=$(git rev-parse --short HEAD)
   echo "SHORT_SHA=${SHORT_SHA}" > build.properties
   ```

3. Add an **Inject environment variables** step (requires the *EnvInject* plugin) pointing to `build.properties`. This makes `SHORT_SHA` available to subsequent steps.

4. Update the Maven build goal:
   ```
   package -DskipTests -Drevision=1.0.0-${BUILD_NUMBER}-${SHORT_SHA}
   ```

5. The resulting artifact — e.g. `sysfoo-1.0.0-42-a3f9d12.jar` — encodes the build number and commit, giving you full traceability.

### Approach 3 — pom.xml Version Extraction

When your pom.xml already declares a proper semantic version and you want to read it dynamically:

1. Add a Pre-build Execute Shell step:
   ```bash
   #!/bin/bash
   # Extract version from pom.xml
   APP_VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
   echo "APP_VERSION=${APP_VERSION}" > build.properties
   echo "BUILD_VERSION=${APP_VERSION}-${BUILD_NUMBER}" >> build.properties
   ```

2. Inject `build.properties` so `BUILD_VERSION` is available to later steps.

3. Use `BUILD_VERSION` in post-build archiving and in artifact naming to get output like `sysfoo-1.3.2-42.jar`.

### Archiving Versioned Artifacts

After applying a versioning strategy, update the **Archive the Artifacts** post-build action to use a pattern that captures the version:

- **Files to archive**: `target/sysfoo-*.jar`
- Check **Fingerprint all archived artifacts** — Jenkins will record the MD5 fingerprint of every artifact, allowing you to trace which builds, jobs, and deployments used a specific file.

![Artifact versioning fingerprint](https://www.jenkins.io/images/post-images/2021-03-fingerprinting.png)

### Promoting Artifacts to a Repository (Nexus / Artifactory)

In a real-world pipeline, archived artifacts should also be pushed to a binary repository for consumption by downstream pipelines and deployment tools.

1. Install the **Nexus Artifact Uploader** plugin (or the Artifactory plugin for JFrog).
2. Add a **Post-build Action → Nexus artifact uploader** step with:
   - **Nexus Version**: Nexus 3
   - **Protocol / Nexus URL**: `http://nexus.example.com:8081`
   - **Credentials**: select your stored Nexus credentials
   - **GroupId / ArtifactId / Version**: use `${BUILD_VERSION}` computed earlier
   - **Repository**: `maven-releases`
   - **File**: `target/sysfoo-${BUILD_VERSION}.jar`
3. Save and run — each successful build will now push a versioned artifact to Nexus, available for all teams to pull by version number.

# Step 4
## Connect Jobs with Upstream and Downstream
So far we have created 3 jobs (Build, Test and Package) that run independently of one another. The goal of this next section is to configure them to run in sequence and get triggered from GitHub, that is connecting them together and also ensuring that they are triggered whenever there is a change from the Git Repository. To make the connection between the jobs, I will be using the concepts of Upstream and Downstream. In this example pipeline, where we have Build, Test and Package, any job that comes before another is its upstream whereas any that comes after it is its downstream. From the perspective of the Test job, the Build job is upstream while the Package job is downstream. That is the way we would be setting up the pipeline to run.

To link the jobs by defining upstream and downstream configurations, follow the steps below:
1. From the **build** job configuration page, scroll all the way to post build actions, this is where you can define the downstream job. Provide test as the job to build, leave other configurations as default and save.

![Screenshot 2024-06-18 075716](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/cecce599-e3b8-4fe6-8cd9-933f9b9b9e9b)

![Screenshot 2024-06-18 075841](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/ddd4993d-3e6b-4e61-a084-d156c97135ee)

2. Next, set up upstream for the **package** job. Go to the package configuration page. From build triggers, check the box for build after other projects are built. Provide upstream project name as **test** job. This would define test as the upstream for package.
3. After upstream and downstream are defined, run build and it will automatically run test and package in that sequence.

# Step 5
## Setup Pipeline View and Pipeline trigger
In this final step, the goal is to visualize the CI/CD process with a pipeline view. This provides a clear and comprehensive overview of the build stages and job statuses. Then we learn also to trigger the pipeline with the build triggers.
1. Begin by installing `build pipeline plugin` from manage Jenkins, select Plugins, next click on Available Plugins, then search for **build pipeline**
2. Install the plugin without restarting (Ignore the warning message. But please note that this plugin is not safe to use for live production environments. This is just for demonstration purposes only)
3. From the Jenkins console, browse to sysfoo folder and create a new view by clicking on the + tab to create a new view. Select type as **pipeline view** and  provide a name for it.

![Screenshot 2024-06-11 085321](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/7f2440cc-362e-4385-b03a-bbf1cb637059)

4. Provide a name and select *Build Pipeline View*

![Screenshot 2024-06-18 090847](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/6ff2c3f1-d457-4269-b040-4c1b346c2156)

5. From the configuration page, select the first job in the pipeline and select number of builds as 5, then leave every thing else as default, then save it.

![Screenshot 2024-06-11 090229](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/6d9f6f69-6bae-4f2a-9935-88d75ea2806f)

6. Once completed, you would see build pipeline of your job. Green signifies a successful build, red is failed, blue signifies a build that is yet to run and yellow is build in progress.

![Screenshot 2024-06-11 090316](https://github.com/cynthia-okoduwa/DevOps-projects/assets/74002629/c51e55bb-4f4f-4bef-8893-1622e5cf8e12)

### Configuring GitHub Webhook Triggers

Polling SCM on a schedule is inefficient — it wastes executor time and introduces lag. The production-standard approach is to use webhooks so GitHub pushes a notification to Jenkins the moment a commit is made.

**Pre-requisite**: Your Jenkins server must be reachable from the internet, or you must use a service like [smee.io](https://smee.io) or [ngrok](https://ngrok.com) as a proxy for local development.

1. In Jenkins, go to the **build** job configuration. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**.

2. In GitHub, go to your forked **sysfoo** repository → **Settings → Webhooks → Add webhook**:
   - **Payload URL**: `http://<your-jenkins-url>/github-webhook/`
   - **Content type**: `application/json`
   - **Which events?**: Just the push event
   - Click **Add webhook**

3. GitHub will send a test ping. In Jenkins, you can verify the webhook was received by checking **Manage Jenkins → System Log**.

4. Now every push to your repository will automatically trigger the build job, which in turn triggers test and package via the upstream/downstream chain you configured.

**Security note**: Validate webhook payloads using a **Secret token**. Set the same token in GitHub's webhook configuration and in the GitHub plugin settings in Jenkins to prevent spoofed build triggers.

---

## Interview Questions & Answers — Manual Pipeline & Jenkins Fundamentals

### Q1: What is the difference between a Freestyle job and a Pipeline job in Jenkins?

**A:** A Freestyle job is configured entirely through the Jenkins UI — it is simple to set up but the configuration lives only in Jenkins, is not version-controlled, and does not scale well for complex workflows. A Pipeline job (using a Jenkinsfile) defines the build, test, and deploy steps as code stored in the source repository alongside the application. This brings version control, code review, auditability, and the ability to rebuild any historical commit with the exact same pipeline that was used at that time.

### Q2: What is the difference between upstream and downstream jobs in Jenkins?

**A:** An upstream job is one that triggers another job when it completes. A downstream job is one that is triggered by a preceding job. In the sysfoo pipeline: Build is upstream to Test (it triggers Test on success), and Test is upstream to Package. From Package's perspective, Test is its upstream. This creates a dependency chain where failures in any upstream job prevent downstream jobs from running, giving you a proper sequential gate.

### Q3: How does Maven integration work in Jenkins and what does each Maven goal do?

**A:** Jenkins uses the Maven Integration plugin to run Maven goals as build steps. Common goals used in CI pipelines: `compile` — compiles source code only, verifying there are no compilation errors; `test` — compiles and runs unit tests, generating Surefire test reports; `package` — compiles, tests, and packages the application into a distributable format (JAR or WAR); `verify` — runs integration tests in addition to unit tests; `install` — runs all of the above and installs the artifact to the local `.m2` repository; `deploy` — pushes the artifact to a remote repository like Nexus or Artifactory.

### Q4: What is artifact fingerprinting in Jenkins?

**A:** Fingerprinting records the MD5 checksum of a build artifact and stores it in Jenkins. When the same artifact is used by downstream jobs or deployed to different environments, Jenkins can cross-reference the fingerprint to show you exactly which build produced it, which jobs consumed it, and where it was deployed. This is essential for traceability and compliance — you can answer "which version is running in production right now and which commit does it correspond to?"

### Q5: How would you trigger a Jenkins job automatically when code is pushed to GitHub?

**A:** Configure a GitHub webhook on the repository that sends a POST to `http://<jenkins-url>/github-webhook/`. On the Jenkins job, enable **GitHub hook trigger for GITScm polling** under Build Triggers. Jenkins receives the webhook event and immediately starts the build. This is far more efficient than SCM polling because there is no lag and no unnecessary polling cycles.

### Q6: What is DIND (Docker-in-Docker) and why is it used in the Jenkins setup described here?

**A:** Docker-in-Docker runs a Docker daemon inside a Docker container. In the Jenkins setup used here, the Jenkins controller itself runs in a container, and DIND provides a separate Docker daemon that Jenkins can use to build and run Docker images for builds. This keeps the host Docker daemon isolated from Jenkins workloads, giving each build a clean environment and preventing builds from interfering with the host system.

### Q7: How do you handle a failing build that is blocking the rest of the pipeline?

**A:** In a manually chained upstream/downstream setup, configure the downstream job's **Build Trigger** to fire only on a **stable** upstream build. If the upstream fails, the downstream never starts — which is the correct default. For more nuanced control (e.g., running cleanup even on failure), use a Declarative Pipeline with a `post { failure {} }` block or `catchError` to control whether pipeline execution continues after a failing step.

### Q8: What is the Build Pipeline Plugin and what are its limitations?

**A:** The Build Pipeline Plugin provides a visual view of connected upstream/downstream Freestyle jobs, showing build status in a colour-coded pipeline layout. It is useful for visualising manual pipelines. However, it is not recommended for production use because it uses Freestyle jobs (not code-defined pipelines), does not support parallel stages, has limited conditional logic, and the plugin is not actively maintained for modern Jenkins versions. The correct modern replacement is the Blue Ocean UI together with a Declarative Pipeline Jenkinsfile.



