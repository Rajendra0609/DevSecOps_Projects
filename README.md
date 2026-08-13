# DevOps Projects

A hands-on collection of DevOps and DevSecOps engineering projects — from foundational web stacks to production-grade CI/CD, Infrastructure as Code, container orchestration, and security automation on AWS.

> **How to use this repo:** Each `ProjectN.md` file is a self-contained, hands-on lab. Work through them roughly in the order shown in [Learning Path](#learning-path) below — later projects assume the environment/skills built in earlier ones (e.g. Project 11 automates Projects 7–10 with Ansible; Projects 16–19 build the same Terraform codebase incrementally).

## Table of Contents

### Web Stack Implementation Projects

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 1](Project1.md) | LAMP Stack Implementation | AWS EC2, Apache, MySQL, PHP |
| [Project 2](Project2.md) | LEMP Stack Implementation | AWS EC2, Nginx, MySQL, PHP |
| [Project 3](Project3.md) | MERN Stack To-Do Application | MongoDB, Express, React, Node.js |
| [Project 4](Project4.md) | MEAN Stack Deployment | MongoDB, Express, Angular, Node.js |
| [Project 5](Project5.md) | Client/Server Architecture | MySQL RDBMS, Database connectivity |
| [Project 6](Project6.md) | Web Solution with WordPress | LVM Storage, Apache, MySQL, WordPress |
| [Project 7](Project7.md) | DevOps Tooling Website Solution | NFS, 3-Tier Architecture, MySQL, Apache |
| [Project 8](Project8.md) | Load Balancer Solution with Apache | Apache mod_proxy_balancer, Load Balancing |
| [Project 10](Project10.md) | Load Balancer with Nginx | Nginx, SSL/TLS, Let's Encrypt |

### CI/CD & Configuration Management (Foundations)

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 9](Project9.md) | Tooling Website Deployment Automation | Jenkins CI, SCM Polling, Build Artifacts |
| [Project 11](Project11.md) | Ansible Configuration Management | Ansible, Playbooks, Inventory, Automating Projects 7–10 |
| [Project 12](Project12.md) | Ansible Refactoring & Static Assignments | Ansible Imports, Roles, Static Assignments |
| [Project 14](Project14.md) | End-to-End CI/CD Pipeline for a PHP App | Jenkins, Ansible, SonarQube, Artifactory |

### Jenkins & CI/CD Projects

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Jenkins Manual Pipeline](JenkinsManualPipeline.md) | Manual Jenkins Setup | Jenkins, Maven, Build Jobs, Upstream/Downstream |
| [Jenkins Pipeline as Code](JenkinsPipelineasCode.md) | Jenkinsfile Pipelines | Jenkinsfile, Groovy, CI/CD Automation |
| [Jenkins Advanced - Nodes & Agents](JenkinsAdvancedNodes.md) | Distributed Builds | SSH Agents, Docker Agents, Kubernetes Agents, Cloud Agents |
| [Jenkins Administration Guide](JenkinsAdminGuide.md) | Complete Admin Guide | Installation, Security, Backup, Performance, Troubleshooting |

### Database Projects

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 35](Project35.md) | PostgreSQL Administration & DevOps | PostgreSQL, Replication, HA, Backup, Docker, Kubernetes |
| [Project 36](Project36.md) | PostgreSQL DevSecOps | Security Hardening, Vault, Vulnerability Scanning, CI/CD, Compliance |

### Infrastructure as Code (Terraform)

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 16](Project16.md) | Automate Infrastructure with IaC — Part 1 | Terraform, VPC, Subnets, Variables |
| [Project 17](Project17.md) | Automate Infrastructure with IaC — Part 2 | Terraform, IGW, NAT Gateway, ALB, ASG, RDS, EFS |
| [Project 18](Project18.md) | Automate Infrastructure with IaC — Part 3 | Terraform Modules, Backend Refactor, Remote State |
| [Project 19](Project19.md) | Automate Infrastructure with IaC — Part 4 | Terraform Cloud, Remote Backend, VCS-Driven Runs |

### DevSecOps Projects

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 22](Project22.md) | Containerization with Docker — Advanced Concepts | Docker Networking, Multi-Stage Builds, Image Hardening |
| [Project 23](Project23.md) | Configuration Management with Ansible | Ansible, Playbooks, Roles, Security Hardening, Dynamic Inventory |
| [Project 24](Project24.md) | Infrastructure as Code with Terraform | Terraform, AWS Resources, Modules, State Management, CI/CD |
| [Project 25](Project25.md) | Security Scanning in CI/CD | SAST, DAST, SonarQube, OWASP ZAP, Trivy, Container Security |
| [Project 26](Project26.md) | Secrets Management with HashiCorp Vault | Vault, KV Engine, Dynamic Secrets, Kubernetes Auth, Encryption |
| [Project 27](Project27.md) | AWS IAM Security Best Practices | IAM, MFA, Password Policies, SCPs, Access Analyzer, SSO |

### AWS Projects

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 28](Project28.md) | AWS Lambda & Serverless | Lambda, API Gateway, DynamoDB, Cognito, SAM, X-Ray |
| [Project 29](Project29.md) | ECS & EKS Fargate | ECS Fargate, EKS Fargate, ALB, Auto Scaling, Service Mesh |
| [Project 30](Project30.md) | AWS CloudFormation | CloudFormation, Nested Stacks, Stack Sets, Macros, Drift Detection |

### Docker & Container Projects

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 20](Project20.md) | Docker Containerization | Docker, Docker Networking, Multi-container |
| [Project 31](Project31.md) | Docker Advanced - Compose, Swarm & Registry | Docker Compose, Docker Swarm, Private Registry, Secrets, Networking |

### Kubernetes Projects

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 21](Project21.md) | Kubernetes Cluster From Scratch | K8s, Manual K8s Setup ("Kubernetes the Hard Way"), TLS Certificates |
| [Project 32](Project32.md) | Kubernetes Advanced - Networking, Storage & Security | K8s Networking, PV/PVC, RBAC, Network Policies, Helm, Istio |

### Tomcat & Web Server Projects

| Project | Description | Topics Covered |
|---------|-------------|----------------|
| [Project 33](Project33.md) | Tomcat Deployment with Docker & Kubernetes | Tomcat, Java WAR deployment, Clustering, SSL, K8s Deployment |
| [Project 34](Project34.md) | Nginx Advanced - Web Server, Reverse Proxy & Load Balancer | Nginx, Reverse Proxy, Load Balancing, Caching, SSL/TLS, Security |

## Learning Path

### Phase 1: Web Stack Fundamentals
- Projects 1–4: LAMP, LEMP, MERN, and MEAN stack implementation
- Project 5: MySQL client/server architecture
- Project 6: WordPress on LVM-backed storage
- Project 7: 3-tier "DevOps Tooling" website with NFS
- Project 8, 10: Load balancing with Apache and Nginx (+ SSL/TLS)

### Phase 2: CI/CD Fundamentals & Configuration Management
- Project 9: Jenkins CI fundamentals
- **Jenkins Manual Pipeline → Pipeline as Code → Advanced Nodes → Admin Guide**: full Jenkins track
- Project 11, 12: Ansible playbooks, roles, and static assignments
- Project 14: End-to-end CI/CD for a PHP app (Jenkins + Ansible + SonarQube + Artifactory)

### Phase 3: Database Management
- Project 5: MySQL database management
- Project 35: PostgreSQL Administration
- Project 36: PostgreSQL DevSecOps

### Phase 4: Containerization
- Project 20: Docker fundamentals
- Project 22: Docker advanced concepts (networking, multi-stage builds, hardening)
- Project 31: Docker Compose, Swarm & private registries

### Phase 5: Orchestration
- Project 21: Kubernetes from scratch
- Project 32: Kubernetes Advanced

### Phase 6: Infrastructure as Code
- Project 16–19: Terraform, built incrementally (VPC → full architecture → modules → Terraform Cloud)
- Project 23: Ansible
- Project 24: Terraform (DevSecOps track)
- Project 30: CloudFormation

### Phase 7: DevSecOps
- Project 25: Security scanning in CI/CD (SAST/DAST)
- Project 26: HashiCorp Vault
- Project 27: AWS IAM security
- Project 36: PostgreSQL DevSecOps

### Phase 8: Serverless & Cloud Native
- Project 28: AWS Lambda
- Project 29: ECS/EKS Fargate

### Phase 9: Web Servers & Application Servers
- Project 33: Tomcat
- Project 34: NGINX

## Getting Started

1. **Prerequisites**
   - AWS Account (most projects use the AWS Free Tier where possible — watch for resources that fall outside it, e.g. NAT Gateways, RDS)
   - Basic understanding of Linux (shell navigation, package managers, systemd)
   - Git and version control

2. **Recommended Tools**
   - AWS CLI
   - Docker Desktop
   - kubectl
   - Terraform
   - Ansible
   - Helm

3. **Cost control:** Several projects provision billable AWS resources (EC2, NAT Gateway, RDS, ALB, EKS). Always run `terraform destroy` / terminate EC2 instances / delete load balancers as soon as you're done with a lab — see each project's cleanup notes.

## Contributing

Feel free to contribute by creating issues or pull requests.

## License

MIT License
