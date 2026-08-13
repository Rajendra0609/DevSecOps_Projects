## Automate Infrastructure with IaC using Terraform — Part 4 (Terraform Cloud)

![Capture](https://user-images.githubusercontent.com/74002629/203817812-62cf36bc-623c-4182-b139-da2580de548b.PNG)

> **Series:** Part 4 of 4 (final) — [Part 1](Project16.md) (VPC & subnets) · [Part 2](Project17.md) (compute, load balancing, RDS, EFS) · [Part 3](Project18.md) (modules & remote state)
> **Level:** Advanced · **Time:** 4–6 hours · **Cost:** Same billable AWS resources as Parts 2–3, plus Packer-built EC2 instances during image builds (terminated automatically when the build finishes). Terraform Cloud itself is free for small teams (Free tier covers up to 5 users).

### Prerequisites
- Completed [Parts 1–3](Project18.md) — this is the capstone that takes that module-based codebase to a managed backend.
- A GitHub (or GitLab/Bitbucket) account, since the recommended workflow is VCS-driven.
- Packer and Ansible installed locally (installation links in step 9 below).

This is the concluding part of the 4-part project on Infrastructure as Code using Terraform. In the previous projects we built infrastructure for our architecture on our local machines, with state manually migrated to an S3 backend in Part 3. In this project we replace that manual workflow with **Terraform Cloud** — a managed service that runs Terraform CLI operations for you, either on demand or in response to VCS events (like a `git push`), and gives you a UI for reviewing plans before they apply.

### Concept: What Terraform Cloud Actually Adds

Part 3 solved *state storage and locking* (S3 + DynamoDB). Terraform Cloud goes further — it also manages *execution*: instead of an engineer's laptop running `terraform apply` against remote state, Terraform Cloud runs the plan/apply itself, in a consistent, logged environment, triggered by a Git push. This closes the last gap in the "team-safe Terraform" story that's been building across this whole series:

| Capability | Local CLI + S3 backend (Part 3) | Terraform Cloud (Part 4) |
|---|---|---|
| State storage | ✅ (S3) | ✅ (managed) |
| State locking | ✅ (DynamoDB) | ✅ (built-in) |
| Where `plan`/`apply` runs | Whoever's laptop/CI runner has credentials | Terraform Cloud's own runners |
| Plan visibility for reviewers | Only if you paste it into a PR comment yourself | Native UI, linked to the PR automatically |
| Trigger | Manual, or hand-rolled in your own CI/CD | Push to a configured branch, natively |
| Secrets handling | You manage `TF_VAR_*` / AWS creds yourself | Workspace-scoped variables, with a "sensitive" flag |

#### Migrate your .tf codes to Terraform Cloud
##### Steps
1. Create a new account with this [link](https://app.terraform.io/signup/account), then verify your email and you are ready.
2. Create an organization, select "Start from scratch", choose a name for your organization and create it.
3. Next, configure a workspace. There are 3 options to configure your workspace: 
- Version Control workflow: This is used to integrate your version control systems like GitHub, Gitlab, etc. When you publish a new version to the default branch, 
it would trigger an automatic plan and also apply to target branch if you set it to do so. It is the most commonly used.
-  CLI-driven workflow: With this you can run terraform commands from your local CLI but you are integrated with terraform cloud using some special sign in 
and settings and it will run your commands in the cloud and make use of that state.
- API-driven workflow: This allows you work with APIs

4. We will use version control workflow as the most common and recommended way to run Terraform commands triggered from our git repository. Create a new repository
 in your GitHub and call it `terraform-cloud`, push your Terraform codes developed in the previous projects to the repository.
5. Choose Version Control Workflow and you will be promped to connect to your Version Control system account to your workspace (choose which ever suits you). 
You will be required to register a new OAuth Application – follow the prompt and connect your newly created repository to the workspace.
6. Move on to "Configure settings", provide a description for your workspace and leave all the remaining settings as default, click "Create workspace"
7. Configure variables. Terraform Cloud supports two types of variables: Environment variables and Terraform variables. Either type can be marked as sensitive, 
to prevents them from being displayed in the Terraform Cloud web UI and makes them write-only. We will set two environment variables: **AWS_ACCESS_KEY_ID** and 
**AWS_SECRET_ACCESS_KEY**, set the values that you used in Project 16. These credentials will be used to provision your AWS infrastructure by Terraform Cloud.
For the Terraform variables instead entering each variable we have created in our `variables.tfvars` file here, simply rename your `terraform.tfvars` file to `terraform.auto.tfvars` in the terraform-cloud directory structure — any file ending in `.auto.tfvars` is loaded automatically by Terraform, without needing a `-var-file` flag, which matters because Terraform Cloud's UI-driven runs have no CLI flag to pass.
After you have set the 2 variables – your Terraform Cloud is all set to apply the codes from GitHub and create all necessary AWS resources.
8. Now it is time to run our Terraform scripts, we would be using Packer to build our images in this project, and Ansible to configure the infrastructure, so for that 
we would be making changes to our existing repository from [Project 18](Project18.md). Add the following folders in your code structure:
- AMI: for building packer images
- Ansible: for Ansible scripts to configure the infrastructure
9. Install the following tools on your local machine:
- [Packer](https://developer.hashicorp.com/packer/tutorials/docker-get-started/get-started-install-cli) To create custom images that are immutable and production ready.
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for your server configuration.

#### Create AMI using Packer
1. Terraform-cloud file structure create a new folder and name it `AMI`, the move all the .sh files from project 19 into it.(bastion.sh, ubuntu.sh, web.sh, nginx.sh)
2. Create 4 new files in the folder and name them as `bastion.pkr.hcl`, `nginx.pkr.hcl`, `web.pkr.hcl` and `ubuntu.pkr.hcl` respectively.
3. Inside the `bastion.pkr.hcl` paste the following user data:

> 💡 **Note on the AMI filter below:** an earlier version of this lab points `source_ami_filter` at a specific, dated RHEL-SAP AMI (`RHEL-SAP-8.2.0_HVM-20211007-...`) with no wildcard in the name. That's a real bug beyond just being outdated — `most_recent = true` only matters when the `name` filter can match *multiple* AMIs; an exact, unwildcarded name can only ever match that one specific (and by now probably deregistered) image, so `most_recent` was silently doing nothing. Below uses Amazon Linux 2023 with a proper wildcard (`al2023-ami-*-x86_64`), so `most_recent` actually does its job and this keeps working as AWS publishes new AL2023 releases.

```
variable "region" {
  type    = string
  default = "us-east-1"
}

locals {
  timestamp = regex_replace(timestamp(), "[- TZ:]", "")
}


# source blocks are generated from your builders; a source can be referenced in
# build blocks. A build block runs provisioners and post-processors on a
# source.
source "amazon-ebs" "terraform-bastion-prj-19" {
  ami_name      = "terraform-bastion-prj-19-${local.timestamp}"
  instance_type = "t2.micro"
  region        = var.region
  source_ami_filter {
    filters = {
      name                = "al2023-ami-*-x86_64"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    most_recent = true
    owners      = ["amazon"]
  }
  ssh_username = "ec2-user"
  tag {
    key   = "Name"
    value = "terraform-bastion-prj-19"
  }
}

# a build block invokes sources and runs provisioning steps on them.
build {
  sources = ["source.amazon-ebs.terraform-bastion-prj-19"]

  provisioner "shell" {
    script = "bastion.sh"
  }
}
```
4. Inside `nginx.pkr.hcl` paste:
```
variable "region" {
  type    = string
  default = "us-east-1"
}

locals { timestamp = regex_replace(timestamp(), "[- TZ:]", "") }


# source blocks are generated from your builders; a source can be referenced in
# build blocks. A build block runs provisioners and post-processors on a
# source.
source "amazon-ebs" "terraform-nginx-prj-19" {
  ami_name      = "terraform-nginx-prj-19-${local.timestamp}"
  instance_type = "t2.micro"
  region        = var.region
  source_ami_filter {
    filters = {
      name                = "al2023-ami-*-x86_64"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    most_recent = true
    owners      = ["amazon"]
  }
  ssh_username = "ec2-user"
  tag {
    key   = "Name"
    value = "terraform-nginx-prj-19"
  }
}


# a build block invokes sources and runs provisioning steps on them.
build {
  sources = ["source.amazon-ebs.terraform-nginx-prj-19"]

  provisioner "shell" {
    script = "nginx.sh"
  }
}
```
5. Inside `ubuntu.pkr.hcl` paste:

> 💡 The original filter here targets Ubuntu 16.04 "Xenial," which is long past end-of-life (standard support ended April 2021). Below targets Ubuntu 22.04 "Jammy" instead — the owner ID (`099720109477`) is Canonical's official AWS account, which was already correct.

> ⚠️ **Region consistency matters here.** `region` defaults to `us-east-1` in every Packer file, matching every other project in this series (Projects 16–18). If you deploy your VPC/ASGs in a different region, the AMI IDs Packer produces here won't exist there — an AMI is region-scoped in AWS. Keep this value in lock-step with `var.region` in your Terraform code.

```
variable "region" {
  type    = string
  default = "us-east-1"
}

locals { timestamp = regex_replace(timestamp(), "[- TZ:]", "") }


# source blocks are generated from your builders; a source can be referenced in
# build blocks. A build block runs provisioners and post-processors on a
# source.
source "amazon-ebs" "terraform-ubuntu-prj-19" {
  ami_name      = "terraform-ubuntu-prj-19-${local.timestamp}"
  instance_type = "t2.micro"
  region        = var.region
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    most_recent = true
    owners      = ["099720109477"]
  }
  ssh_username = "ubuntu"
  tag {
    key   = "Name"
    value = "terraform-ubuntu-prj-19"
  }
}


# a build block invokes sources and runs provisioning steps on them.
build {
  sources = ["source.amazon-ebs.terraform-ubuntu-prj-19"]

  provisioner "shell" {
    script = "ubuntu.sh"
  }
}
```
6. For web.pkr.hcl, paste:
```
variable "region" {
  type    = string
  default = "us-east-1"
}

locals { timestamp = regex_replace(timestamp(), "[- TZ:]", "") }


# source blocks are generated from your builders; a source can be referenced in
# build blocks. A build block runs provisioners and post-processors on a
# source.
source "amazon-ebs" "terraform-web-prj-19" {
  ami_name      = "terraform-web-prj-19-${local.timestamp}"
  instance_type = "t2.micro"
  region        = var.region
  source_ami_filter {
    filters = {
      name                = "al2023-ami-*-x86_64"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    most_recent = true
    owners      = ["amazon"]
  }
  ssh_username = "ec2-user"
  tag {
    key   = "Name"
    value = "terraform-web-prj-19"
  }
}


# a build block invokes sources and runs provisioning steps on them.
build {
  sources = ["source.amazon-ebs.terraform-web-prj-19"]

  provisioner "shell" {
    script = "web.sh"
  }
}
```
7. These Packer configurations will make use of the user data provided in their respective .sh files.

> ⚠️ **Known gap:** this lab references `bastion.sh`, `nginx.sh`, `ubuntu.sh`, and `web.sh` as the provisioning scripts Packer bakes into each AMI, but doesn't show their contents. At minimum, each should: update packages, install the tier's software (nginx for the reverse proxy tier; Apache/PHP for the web tier; Ansible + AWS CLI for the Ubuntu/control image), and configure a `/healthstatus` endpoint matching the target group health checks from [Part 2](Project17.md). Note also that `web.pkr.hcl`/`web.sh` implies **one shared AMI for both WordPress and Tooling** — a simplification from Parts 2–3, which built them as separate roles. If you want per-role optimized images instead, duplicate `web.pkr.hcl` into `wordpress.pkr.hcl` and `tooling.pkr.hcl`, each with its own `.sh` script.
8. Next in your terminal, navigate to the terraform-cloud directory and begin building your AMI. Type `packer build <name of packer file>` to build each packer AMI
9. Once build is complete, you should see your Web, Bastion, Nginx and Ubuntu AMIs in your AWS console.
![AMIs](https://user-images.githubusercontent.com/74002629/203560607-aba1b0bb-e8ac-461e-8d11-f490fe18afe2.PNG)
10. Copy the AMI ID of each AMI created and update the terraform.auto.tfvars file with the newly created AMIs. Note: the Ubuntu AMI will be used for Sonarqube.
![AMI ID](https://user-images.githubusercontent.com/74002629/203724846-2ec829c7-3795-4366-9d00-309b3e4c988f.PNG)
11. Push all your changes to the `terraform-cloud` repository.
12. Next in the Terraform cloud UI, run your first plan, if all goes well, run Apply. You will see your AWS resources being created.
![Pix2](https://user-images.githubusercontent.com/74002629/203728085-305eb60b-fa6b-433b-8f46-c88377f42ad3.PNG)
13. You have successfully created your resources using Terraform cloud. When you go into your AWS console you should see all the resources you have created, however our instances in the target group have failed health checks, because we have not configured the instances. Let's fix that.
14. In our terraform code, remove the instances as listeners and attachment to auto scaling groups. Comment out the nginx, loadbalancer and tooling listeners in the ALB module of your terraform code. and also comment out the attachment of autoscaling groups to the load balancers in the Autoscaling module of your terraform code. We are doing this to prevent issues till we have run our configurations, then we can reapply them.
15. Push your changes and Terraform cloud with Plan and Apply automatically

#### Update Ansible script with values from Terraform output.
1. Next we would be updating the Ansible script with values from terraform output. 
2. SSH into your Bastion server using SSH agent and clone down your repository.
3. Ansible will need to have access to your AWS accout to pull down the IP addresses of your instances, we will need to give Ansible access. Enter `aws configure` in your ansible directory, then follow the prompt and provide your secret key and access key to give Ansible access.
4. Ensure Ansible can pull down the required IPs by running: `ansible-inventory -i inventory/aws.ec2.yml --graph` 
5. Update the nginx role with DNS name for the load balancer. Go into your loadbalancer in your AWS console and copy the DNS name for the load balancer and paste it in the nginx role.
6. Next update the RDS end-point in the tooling/tasks/setup-db.yml  and wordpress/tasks/setup-db.yml files (You get this from your AWS console from the RDS you provisioned) Update it for the database and tooling credentials.
7. Also update your username name and password to correspond to what you have in your terraform.auto.tfvars
8. Next, update the access points of your filesystem for both the tooling and the wordpress site. it is located in `Ansible/roles/tooling/tasks/main.yml` and `Ansible/roles/wordpress/tasks/main.yml` respectively.
9. In the Ansible folder, create an ansible.cfg file and specify your roles path to allow Ansible find the roles when it runs, then run `export ANSIBLE_CONFIG=<path to your ansible.cfg file>` in the terminal to tell ansible where to find the roles
10. Now run Ansible playbook against your environment. `ansible-playbook -i inventory/aws.ec2.yml playbooks/site.yml` If all goes well, you should see your playbook running.
11. Now we get into your Bastion server via SSH agent. Type `ssh -A ec2-user@<Bastion-Public-IP>` and from inside the bastion we can get into the other servers in the architecture.
12. To get into the other servers from the Bastion using SSH agent, enter the command `ssh ec2-user@<Private-IP-of-target-server>`
13. Once inside your Nginx server run `sudo systemctl status nginx` to verify the server is running, then `sudo vi /etc/nginx/nginx.conf` to verify everything was configured correctly.
14. SSH into your tooling and wordpress servers respectively and run `df -h` to verify that the filesystem was successfully mounted and `sudo systemctl status httpd` to verify Apache is running.
15. in each server run change directory to `cd /var/www/html/` to see the health status of your servers, and `curl localhost` to see the webserver running in your local machine.






---

## Security Considerations

- **Terraform Cloud workspace variables replace `.tfvars` for secrets.** Marking `AWS_SECRET_ACCESS_KEY` and any sensitive Terraform variable (like `master-password`) as "sensitive" in the workspace UI means it's write-only after saving — nobody, including you, can view it again through the UI, only overwrite it. This is a genuinely stronger guarantee than a `.gitignore`'d `.tfvars` file, which still exists in plaintext on disk.
- **Prefer a dedicated IAM user/role scoped to this workspace over reusing your personal AWS credentials.** The `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` pair you give Terraform Cloud should belong to an IAM identity with exactly the permissions this project's modules need (VPC, EC2, RDS, ALB, IAM `PassRole` for the instance profile, etc.) — not your own admin user. If a workspace variable ever leaks, the blast radius should be "this project's resources," not "everything in the account."
- **VCS-driven runs mean your Git repository is now part of your deployment trust boundary.** Anyone who can push to the connected branch (or open a PR, depending on your workspace's PR-plan settings) can trigger a `plan` — and, if auto-apply is on, an `apply` — against real infrastructure. Enable branch protection on the branch Terraform Cloud watches, and consider requiring manual "Confirm & Apply" in the UI rather than auto-apply, at least for anything touching production.
- **Packer-built AMIs should themselves be scanned**, not just trusted because they came from your own pipeline — run a vulnerability scanner (Trivy, or AWS Inspector) against each AMI as a step after `packer build`, before it's referenced in `terraform.auto.tfvars`. [Project 25](Project25.md) covers wiring security scanning into a pipeline like this one.

---

## Common Issues & Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Terraform Cloud run fails at `plan` with `No valid credential sources found` | The `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` workspace variables weren't set as **environment variables** (they were added as Terraform variables instead, or not marked sensitive/category correctly) | In the workspace's Variables tab, confirm both are added under the "Environment Variables" category, not "Terraform Variables" |
| `terraform.auto.tfvars` values don't seem to apply — Terraform Cloud runs with defaults instead | The file wasn't actually renamed/committed with the `.auto.tfvars` suffix, or it's `.gitignore`'d (a leftover from local development where you deliberately excluded `.tfvars`) | Confirm the file is committed to the repo Terraform Cloud is watching — unlike local secrets-in-`.tfvars`, this file (if it has no real secrets — see below) needs to be tracked for VCS-driven runs to see it |
| Instances launch but immediately fail target group health checks (as the walkthrough itself calls out in step 13) | Expected at this stage — the ALB/ASG resources exist, but the instances haven't been configured yet (that's Ansible's job, not yet run) | This is why the walkthrough has you comment out listeners/attachments temporarily — don't treat failing health checks as a bug until *after* you've run the Ansible playbook |
| `packer build` fails with `InvalidAMIID.NotFound` or a filter that matches zero AMIs | AMI filters are region-specific — an AMI name filter valid in `us-east-1` may return nothing in another region, or AWS has deprecated/removed an older image referenced by an unwildcarded name | Confirm `var.region` in the `.pkr.hcl` file matches where you're actually building, and prefer wildcarded, actively-maintained AMI families (Amazon Linux 2023, current Ubuntu LTS) over pinned historical AMI names |
| `ansible-inventory -i inventory/aws.ec2.yml --graph` returns nothing / can't see EC2 instances | `aws configure` wasn't run inside the environment Ansible actually executes in (e.g. configured on your laptop, but you're running Ansible from the bastion host, which has its own separate credential state) | Run `aws configure` (or better, attach an IAM instance profile to the bastion so it never needs long-lived keys at all) on the host actually executing `ansible-playbook` |
| Ansible playbook runs but the application still can't reach RDS/EFS | RDS endpoint / EFS mount target hardcoded or missed in `setup-db.yml` / role `main.yml`, or security groups still reference the wrong tier | Pull these values from `terraform output` rather than typing them by hand from the console — reduces copy-paste errors, and is the natural next step toward *fully* automating this handoff instead of doing it manually as this lab does |

---

## Best Practices

- **Treat "manually copy Terraform outputs into Ansible files" (steps in this lab) as a known interim step, not the end state.** The natural evolution: have Terraform write outputs to a file Ansible can read directly (`terraform output -json > terraform_outputs.json`, consumed by a dynamic inventory script or `include_vars`), or better, wire this handoff into a single CI/CD pipeline job (see [Project 24](Project24.md)) so there's no manual copy-paste step to get wrong.
- **Pin Packer plugin and Terraform provider versions**, same principle as pinning in Part 1 — a `packer_version` constraint and pinned `amazon-ebs` plugin version keep builds reproducible months later.
- **Rebuild AMIs on a schedule, not just when you remember to.** "Golden AMI" pipelines in real organizations typically rebuild weekly (picking up OS security patches automatically) via a scheduled CI job that runs `packer build` and updates the `terraform.auto.tfvars` AMI ID as a PR, rather than relying on a human to notice an AMI is stale.
- **Use Terraform Cloud's "Plan Only" runs on pull requests**, with apply gated behind merge — this gives reviewers the plan diff directly in the PR (Terraform Cloud posts a check/comment), which is the single highest-value habit for catching an unintended `-/+ destroy and recreate` before it ships.

---

## Real-World Scenario

This is, structurally, close to a real "immutable infrastructure" pipeline: Packer builds versioned, pre-baked AMIs (so a new instance is ready to serve traffic within its boot time, not minutes of `apt install` on every launch); Terraform provisions the infrastructure referencing those AMI IDs; Ansible (or, increasingly, nothing — many teams now bake *everything* into the AMI via Packer provisioners and skip a separate configuration-management step entirely) finishes runtime configuration; and Terraform Cloud gives the whole thing a reviewable, auditable trigger via Git. The main gap between this lab and a fully mature pipeline is automation of the handoffs — this lab has you manually copy AMI IDs and RDS endpoints between tools; [Project 24](Project24.md) shows how to wire that into one CI/CD pipeline with no manual steps.

---

## Challenge

1. **Automate the AMI ID handoff.** Instead of manually copying Packer's output AMI ID into `terraform.auto.tfvars` (step 10), write a small script (or CI job) that parses `packer build -machine-readable` output and updates the `.tfvars` file automatically, then commits/pushes the change to trigger a new Terraform Cloud run.
2. **Automate the Terraform → Ansible handoff.** Add `output` blocks for the RDS endpoint, EFS mount target, and ALB DNS name; write a small script that runs `terraform output -json` and generates the corresponding Ansible `group_vars` file, replacing the manual copy-paste in steps 5–6 of the Ansible section.
3. **Re-enable the commented-out listeners/attachments** (step 14) after your Ansible run succeeds, and confirm target group health checks go green — this closes the loop the walkthrough deliberately leaves open.
4. **Set up branch-protected, plan-only PR runs** in your Terraform Cloud workspace, so every PR shows a plan as a check, and apply only happens after merge to your main branch.

---

## Interview Q&A

**Q: What does Terraform Cloud add on top of an S3 + DynamoDB remote backend that you don't already get from Part 3?**
A: Part 3's S3+DynamoDB backend solves state storage and locking — but `plan`/`apply` still runs wherever an engineer or CI job invokes the CLI. Terraform Cloud additionally manages *execution*: it runs plans/applies on its own infrastructure, triggered by VCS events, with a UI that shows the plan for review before it applies — closing the gap between "state is safely shared" and "changes are safely reviewed and applied."

**Q: Why build AMIs with Packer instead of just using a generic base image and configuring everything with Ansible at boot time (`user_data`)?**
A: Boot-time configuration means every instance launch pays the cost (and risk) of package installs and configuration steps — slower autoscaling response, and a network blip or package repo outage during boot can leave an instance half-configured. A Packer-built "golden AMI" does that work once, at build time, and produces an immutable, tested artifact; every instance launched from it starts already-correct. It also means your ASG can scale out in the time it takes to boot an EC2 instance, not the time it takes to boot *and* run a full configuration script.

**Q: In this lab, target group health checks fail immediately after `terraform apply`, before Ansible has run. Is that a bug?**
A: No — it's the expected, if slightly awkward, consequence of doing infrastructure provisioning (Terraform) and application configuration (Ansible) as two separate, sequential phases without automation gluing them together. The instances exist and are attached to the ASG, but haven't been configured to actually serve traffic on the health check path yet. The lab works around it by temporarily commenting out the ALB listeners/attachments; a more mature pipeline would instead sequence these phases automatically in CI/CD so there's never a "known-broken" intermediate state to work around by hand.

**Q: How would you keep Packer-built AMIs from silently going stale (accumulating unpatched CVEs over time)?**
A: Rebuild on a schedule (e.g. a weekly CI job), not only when someone remembers to. Combine this with pinning the AMI's *source* filter to a maintained, wildcarded family (like this lab's fix from a dated, exact-name RHEL AMI to a wildcarded Amazon Linux 2023 filter) so each rebuild actually picks up the latest upstream base image and OS patches, and scan the resulting AMI (Trivy/Inspector) before promoting its ID into `terraform.auto.tfvars`.

---

## Series Wrap-Up & Next Steps

Across Parts 1–4 you've taken a single hardcoded `main.tf` all the way to a production-shaped pipeline: variables and data sources (Part 1) → a full multi-tier architecture with load balancing, RDS, and EFS (Part 2) → modules and remote state (Part 3) → managed execution via Terraform Cloud, plus immutable AMIs via Packer (Part 4). That progression — hardcode → parameterize → modularize → automate execution — is the same arc real Terraform codebases go through as they mature, and it's a genuinely good structure to describe in an interview when asked "walk me through how you'd design this."

From here:
- **[Project 24](Project24.md)** puts this entire pipeline behind CI/CD, closing the manual-handoff gaps called out throughout this part.
- **[Project 23](Project23.md)** goes deeper on Ansible — roles, dynamic inventory, and hardening — beyond what this part used it for.
- **[Project 26](Project26.md)** replaces this series' `terraform.tfvars`/workspace-variable approach to secrets with HashiCorp Vault, for dynamic, short-lived credentials instead of long-lived ones.

Remember to destroy every AWS resource created across all four parts when you're done experimenting — check the AWS Console directly, not just `terraform destroy` output, since Packer-built AMIs and their backing EBS snapshots persist independently of your Terraform state and will continue costing storage until deregistered by hand.
