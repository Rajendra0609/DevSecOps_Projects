# 📋 DevSecOps Projects – Complete Review Report

> **Files Reviewed:** 20 README/Markdown files  
> **Total Issues Found:** 80+ (Critical bugs, typos, structural errors, outdated code, suggestions)

---

## 🔴 CRITICAL BUGS (Will Break Functionality)

| File | Line/Section | Bug | Fix |
|------|-------------|-----|-----|
| `Project7.md` | Step 2, DB Server | `sudo my sql` (space in command) | `sudo mysql` |
| `Project16.md` | VPC resource block | `enable_dns_hostnames = var.enable_dns_support` (wrong variable) | `= var.enable_dns_hostnames` |
| `Project16.md` | VPC resource block | `enable_classiclink_dns_support = var.enable_classiclink` (wrong variable) | `= var.enable_classiclink_dns_support` |
| `Project14.md` | Phase 2 Jenkinsfile | `inventory: 'inventory/dev,` (missing closing quote) | `inventory: 'inventory/dev',` |
| `Project10.md` | Certbot command | `biuldwithme.link` (typo in domain name) | `buildwithme.link` |
| `JenkinsPipelineasCode.md` | Entire file | **File is completely empty** | Needs full content |
| `Project6.md` | Step 25 | `` `sudo mount -a` `` (backtick inside code block) | `sudo mount -a` |
| `Project17.md` | terraform.tfvars | File uses `variable` blocks with `type/description` — **that is `variables.tf` syntax, not `.tfvars` syntax** | `.tfvars` should use plain `key = "value"` assignments only |

---

## 🟠 TYPOS & SPELLING MISTAKES

### Project1 (LAMP Stack)
| Mistake | Correction |
|---------|-----------|
| `ASW account setup` | `AWS account setup` |
| `I selelected` | `I selected` |
| `locxcation` | `location` |
| `Ubutun instance` | `Ubuntu instance` |
| `Tpye:` | `Type:` |
| `Esure your configuration` | `Ensure your configuration` |
| `Relaod the public IP` | `Reload the public IP` |

### Project2 (LEMP Stack)
| Mistake | Correction |
|---------|-----------|
| `Sudo apt update` | `sudo apt update` (Linux commands are case-sensitive) |
| `namo editor` | `nano editor` |
| `follwing command` | `following command` |
| `2 pacakages` | `2 packages` |
| `validate password pluggin` | `validate password plugin` |
| `LAMP stack is completely installed` (Step 5 header) | `LEMP stack is completely installed` |

### Project4 (MEAN Stack)
| Mistake | Correction |
|---------|-----------|
| `sever.js` (multiple) | `server.js` |
| `for some reson` | `for some reason` |
| `foloowing html` | `following html` |

### Project5 (MySQL Client/Server)
| Mistake | Correction |
|---------|-----------|
| `lastest updates` | `latest updates` |
| `mysql cient` | `mysql client` |
| `answer appropraitely` | `answer appropriately` |
| `grant privieges` | `grant privileges` |
| `11.  . Finally` | `11. Finally` (extra period) |

### Project6 (WordPress)
| Mistake | Correction |
|---------|-----------|
| `depemdencies` | `dependencies` |
| `accisble from the Internet` | `accessible from the Internet` |

### Project7 (DevOps Tooling)
| Mistake | Correction |
|---------|-----------|
| `sudo my sql` | `sudo mysql` **(CRITICAL – breaks execution)** |
| `grant all privilleges` | `grant all privileges` |
| `login into the websute` | `login into the website` |
| `respectievly` | `respectively` |
| `choosen text editor` | `chosen text editor` |

### Project8 (Apache Load Balancer)
| Mistake | Correction |
|---------|-----------|
| `traffic will be disctributed` | `traffic will be distributed` |

### Project9 (Jenkins CI)
| Mistake | Correction |
|---------|-----------|
| `files probuced by the build` | `files produced by the build` |
| `previouslys define remote directory` | `previously defined remote directory` |
| `Set build actionds over SSH` | `Set build actions over SSH` |
| `mointing point` | `mounting point` |

### Project10 (Nginx LB + SSL)
| Mistake | Correction |
|---------|-----------|
| `LetsEnrcypt` | `LetsEncrypt` |
| `cetrbot` | `certbot` |
| `achitecture` | `architecture` |
| `biuldwithme.link` in certbot command | `buildwithme.link` **(CRITICAL)** |

### Project11 (Ansible)
| Mistake | Correction |
|---------|-----------|
| `Continuating from project 9` | `Continuing from project 9` |
| `instal Ansible` | `install Ansible` |
| `Jenkin-Ansible` (inconsistent) | `Jenkins-Ansible` (as named earlier) |

### Project14 (CI/CD PHP)
| Mistake | Correction |
|---------|-----------|
| `Prerequsites` | `Prerequisites` |
| `Secuirty groups` | `Security groups` |
| `enviroment` | `environment` |
| `comfiguring Jenkins` | `configuring Jenkins` |
| `ansibile-config-mgt` | `ansible-config-mgt` |
| `Istall my sql client` | `Install mysql client` |
| `at the promot` | `at the prompt` |
| `reopsitory` | `repository` |
| `manaually` | `manually` |
| `newly create access token` | `newly created access token` |
| `arifactory` | `artifactory` |

### Project16 (Terraform Part 1)
| Mistake | Correction |
|---------|-----------|
| `accisble from the Internet` | `accessible from the Internet` |
| `parctice` | `practice` |
| `lenght function` (multiple) | `length function` |
| `varible.tf` | `variables.tf` |
| `shouke look like` | `should look like` |

### Project17 (Terraform Part 2)
| Mistake | Correction |
|---------|-----------|
| `creat an Internet Gateway` | `create an Internet Gateway` |
| `resouces` | `resources` |
| `infrastucture` | `infrastructure` |
| `extaernal load balancer` | `external load balancer` |
| `reverser proxy` | `reverse proxy` |
| `Autoscslaling group` | `Autoscaling group` |
| `austo-scaling group` | `auto-scaling group` |
| `route for te the internet` | `route for the internet` |
| `acess` | `access` |
| `balancaer` | `balancer` |
| `bastiopn` | `bastion` |

### Project18 (Terraform Part 3)
| Mistake | Correction |
|---------|-----------|
| `wriiten in a long list` | `written in a long list` |
| `diifcult to understand` | `difficult to understand` |
| `Apllication Load balancer` | `Application Load balancer` |
| `Autosacling` | `Autoscaling` |
| `netowrking resources` | `networking resources` |
| `decalred` | `declared` |
| `slao making sure` | `also making sure` |
| `ngnix` | `nginx` |
| `craete` | `create` |
| `neccessary` | `necessary` |
| `sonbarqube` | `sonarqube` |
| `abd jfrog` | `and jfrog` |
| `maigrating the backend` | `migrating the backend` |
| `verionsing` | `versioning` |
| `encrption` | `encryption` |
| `ami foir sonar` | `ami for sonar` |

---

## 🟡 STRUCTURAL / NUMBERING ERRORS

| File | Issue | Fix |
|------|-------|-----|
| `Project1.md` | Step numbers in STEP 4 jump from 5→7 (step 6 skipped), step 12 appears twice | Renumber steps sequentially |
| `Project1.md` | STEP 5 numbering goes 1,2,4,5,6,7,8,9 (step 3 missing) | Renumber steps sequentially |
| `Project7.md` | Step 8 appears twice | Renumber to step 7 and step 8 |
| `Project8.md` | Two consecutive steps labelled as step 8 | Renumber to step 7 and step 8 |
| `Project12.md` | Step 2 numbering: 1,2,3,4,6,7 (step 5 missing) | Renumber steps |
| `JenkinsManualPipeline.md` | Step 3 is empty (Configure artifact versioning has no content) | Add content for artifact versioning |

---

## 🔵 CONTENT MISTAKES (Wrong Information)

| File | Issue | Fix |
|------|-------|-----|
| `Project3.md` | Title says `SIMPLE WEB STACK IMPLEMENTATION (LEMP STACK)` but content is **MERN stack** | Fix title to `MERN STACK TODO APPLICATION` |
| `Project2.md` | Step 5: `LAMP stack is completely installed and fully operational` | Change to `LEMP stack` |
| `JenkinsManualPipeline.md` | Package job says "create jar" but the archive path is `**target/*.jar` — then says `e.g. sysfoo.war` (inconsistent) | Consistently use `.jar` throughout the package job steps |
| `Project16.md` | VPC resource block references wrong variables (see Critical Bugs above) | Use correct variable names |
| `Project17.md` | `terraform.tfvars` file content uses `variable "name" { type = ... description = ... }` syntax — **this is `variables.tf` syntax, not `.tfvars`** | `.tfvars` should only contain `key = "value"` pairs |

---

## 🟣 DEPRECATED / OUTDATED CODE

| File | Issue | Recommendation |
|------|-------|---------------|
| `Project9.md`, `JenkinsAdminGuide.md` | Jenkins signing key uses deprecated `apt-key add` method | Use new signed-by method: `curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \| sudo tee /usr/share/keyrings/jenkins-keyring.asc` |
| `Project4.md` | MongoDB 3.4 from Ubuntu 14.04 (Trusty) repo — **EOL since 2018** | Use MongoDB 6.x or 7.x with Ubuntu 20.04/22.04 repo |
| `Project4.md` | Node.js 12 is EOL (End of Life) | Use Node.js 18 LTS or 20 LTS |
| `Project16.md`, `Project17.md`, `Project18.md` | `enable_classiclink` and `enable_classiclink_dns_support` were **removed** from AWS Terraform provider v4+ | Remove these attributes; they no longer exist |
| `Project17.md` | `alb_target_group_arn` is deprecated in `aws_autoscaling_attachment` | Use `lb_target_group_arn` instead |

---

## 🟢 SUGGESTIONS & BEST PRACTICE IMPROVEMENTS

### JenkinsAdminGuide.md & JenkinsAdvancedNodes.md
- Code blocks are missing language identifiers (e.g., ` ```bash `, ` ```yaml `, ` ```groovy `) — they incorrectly place the language name **inside** the block as plain text on the first line. This causes incorrect syntax highlighting.

### General Security
- Projects 5, 6, 7, 8: **Never use `0.0.0.0/0` for SSH (port 22) inbound rules** in production — always restrict to specific IPs.
- Project 17 bastion security group allows SSH from anywhere (`0.0.0.0/0`) — restrict to your known IP.
- Project 2: PHP todo_list.php hardcodes `$password = "password"` — use environment variables instead.

### Terraform Projects (16, 17, 18)
- Add `sensitive = true` to all password variables to prevent them appearing in `terraform plan/apply` output.
- Use **remote state** with S3 + DynamoDB locking from Project 16 onwards (not just Project 18).
- Avoid committing `terraform.tfvars` containing real credentials to Git — use `.gitignore`.

### JenkinsPipelineasCode.md
- **Completely empty file** — This is a significant gap in the learning series. It should contain a Jenkinsfile-based declarative pipeline example covering: SCM checkout, build, test, and deploy stages.

### Project14.md
- The Ansible `ansible.cfg` code block is missing its opening ` ```ini ` or ` ```ini ` fencing — the content starts inline without a code block.
- The `Install SonarQube on Ubuntu 20.04` section has steps described with images only, missing the actual commands.

---

## 🗑️ HOW TO REMOVE / CLEAN UP THE LAB

After completing any project, run the following to clean up your environment:

### Remove EC2 Instances / AWS Resources (all projects except Terraform)
```bash
# Terminate EC2 instances via AWS Console → EC2 → Instances → Select → Terminate
# Or use AWS CLI:
aws ec2 terminate-instances --instance-ids <instance-id-1> <instance-id-2>

# Delete Security Groups
aws ec2 delete-security-group --group-id <sg-id>

# Release Elastic IPs (if allocated)
aws ec2 release-address --allocation-id <alloc-id>
```

### Remove Terraform Infrastructure (Projects 16, 17, 18)
```bash
# Navigate to your Terraform project directory
cd PBL/

# Destroy all resources (always review the plan first!)
terraform plan -destroy
terraform destroy

# Type 'yes' when prompted to confirm destruction

# If using S3 backend (Project 18), migrate state back to local first:
terraform init -migrate-state
terraform destroy
```

### Remove Docker/Jenkins Lab (JenkinsManualPipeline)
```bash
# Stop and remove all containers
cd bootcamp/jenkins
docker-compose down

# Remove volumes (WARNING: deletes all Jenkins data)
docker-compose down -v

# Remove the cloned repo
cd ../..
rm -rf bootcamp
```

### Remove Ansible Files (Projects 11, 12)
```bash
# On your local machine / Jenkins-Ansible server
rm -rf ~/ansible-config-mgt
rm -rf ~/ansible-config-artifact

# Remove installed packages (optional)
sudo apt remove ansible -y
sudo apt autoremove -y
```

### Remove the Cloned Repo Itself
```bash
# If you cloned this repo locally:
rm -rf DevSecOps_Projects

# Verify it's gone
ls -la | grep DevSecOps
```

---

*Review completed — 20 files analyzed, 80+ issues documented.*
