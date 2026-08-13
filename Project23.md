# CONFIGURATION MANAGEMENT WITH ANSIBLE

## Project Overview

This project demonstrates using Ansible for configuration management and automating server setup. Ansible is an open-source automation tool that simplifies configuration management, application deployment, and task automation.

### Objectives
1. Install and configure Ansible on a control node
2. Create Ansible playbooks for automated server configuration
3. Implement security hardening with Ansible
4. Use Ansible roles for modular configuration
5. Implement dynamic inventory for AWS EC2 instances

## Prerequisites
- AWS Account
- Ubuntu/Debian based control node
- Multiple target servers (can be EC2 instances)
- SSH access to target servers

## Step 1: Install and Configure Ansible

### Install Ansible on Control Node

```
bash
# Update package index
sudo apt update

# Install software-properties-common (for add-apt-repository)
sudo apt install software-properties-common -y

# Add Ansible repository
sudo add-apt-repository ppa:ansible/ansible -y

# Install Ansible
sudo apt update
sudo apt install ansible -y

# Verify installation
ansible --version
```

### Configure Ansible

```
bash
# Create working directory
mkdir -p ~/ansible/projects && cd ~/ansible/projects

# Create Ansible configuration file
cat > ansible.cfg <<EOF
[defaults]
inventory = inventory
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
EOF
```

## Step 2: Create Static Inventory

```
bash
# Create inventory file
cat > inventory <<EOF
[webservers]
web1 ansible_host=<EC2_PUBLIC_IP_1> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/key.pem
web2 ansible_host=<EC2_PUBLIC_IP_2> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/key.pem

[databases]
db1 ansible_host=<EC2_PUBLIC_IP_3> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/key.pem

[allservers:children]
webservers
databases

[allservers:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

### Test Connection

```
bash
# Test connection to all hosts
ansible all -m ping

# Gather facts from a specific host
ansible web1 -m setup
```

## Step 3: Create First Playbook

```
bash
# Create playbook directory
mkdir -p playbooks

# Create first playbook
cat > playbooks/webserver.yml <<EOF
---
- name: Configure Web Servers
  hosts: webservers
  become: true
  vars:
    http_port: 80
    server_name: example.com
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    - name: Install Apache
      apt:
        name: apache2
        state: present

    - name: Install PHP and required modules
      apt:
        name: ['php', 'libapache2-mod-php', 'php-mysql']
        state: present

    - name: Start Apache service
      service:
        name: apache2
        state: started
        enabled: true

    - name: Create document root
      file:
        path: /var/www/html
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Deploy custom index.html
      template:
        src: templates/index.html.j2
        dest: /var/www/html/index.html
      notify: restart apache

  handlers:
    - name: restart apache
      service:
        name: apache2
        state: restarted
```

### Create Template

```
bash
# Create templates directory
mkdir -p playbooks/templates

# Create template file
cat > playbooks/templates/index.html.j2 <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>Welcome to {{ server_name }}</title>
</head>
<body>
    <h1>Server: {{ ansible_hostname }}</h1>
    <p>IP Address: {{ ansible_default_ipv4.address }}</p>
    <p>OS: {{ ansible_os_family }} {{ ansible_distribution_version }}</p>
</body>
</html>
EOF
```

### Run the Playbook

```
bash
# Run playbook
ansible-playbook playbooks/webserver.yml

# Run with specific tags
ansible-playbook playbooks/webserver.yml --tags apache

# Run in check mode (dry run)
ansible-playbook playbooks/webserver.yml --check
```

## Step 4: Security Hardening Playbook

```
bash
# Create security hardening playbook
cat > playbooks/security_hardening.yml <<EOF
---
- name: Security Hardening
  hosts: all
  become: true
  vars:
    admin_email: admin@example.com
    allowed_ssh_port: 22
    
  tasks:
    - name: Ensure fail2ban is installed
      apt:
        name: fail2ban
        state: present

    - name: Configure fail2ban
      template:
        src: templates/fail2ban.j2
        dest: /etc/fail2ban/jail.local
      notify: restart fail2ban

    - name: Ensure UFW is installed
      apt:
        name: ufw
        state: present

    - name: Set default UFW policies
      ufw:
        direction: "{{ item.direction }}"
        policy: "{{ item.policy }}"
      loop:
        - { direction: 'incoming', policy: 'deny' }
        - { direction: 'outgoing', policy: 'allow' }

    - name: Allow SSH
      ufw:
        rule: allow
        port: '22'
        proto: tcp

    - name: Allow HTTP/HTTPS
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop:
        - '80'
        - '443'

    - name: Enable UFW
      ufw:
        state: enabled

    - name: Disable root login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: restart sshd

    - name: Disable password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication no'
      notify: restart sshd

    - name: Ensure automatic updates are enabled
      apt:
        name: unattended-upgrades
        state: present

    - name: Configure automatic updates
      template:
        src: templates/50unattended-upgrades.j2
        dest: /etc/apt/apt.conf.d/50unattended-upgrades
      when: ansible_os_family == "Debian"

  handlers:
    - name: restart fail2ban
      service:
        name: fail2ban
        state: restarted

    - name: restart sshd
      service:
        name: sshd
        state: restarted
EOF
```

## Step 5: Ansible Roles

### Create Role Structure

```
bash
# Create role directory structure
mkdir -p roles/common/{tasks,handlers,templates,vars,defaults,files}

# Common role - main.yml
cat > roles/common/tasks/main.yml <<EOF
---
- name: Install common packages
  apt:
    name: "{{ common_packages }}"
    state: present

- name: Set timezone
  timezone:
    name: "{{ timezone }}"
EOF

# Common role - defaults
cat > roles/common/defaults/main.yml <<EOF
---
common_packages:
  - curl
  - wget
  - git
  - vim
  - htop
timezone: UTC
EOF
```

### Use Role in Playbook

```
bash
# Create playbook using roles
cat > playbooks/site.yml <<EOF
---
- name: Apply common configuration to all servers
  hosts: all
  become: true
  roles:
    - common

- name: Configure webservers
  hosts: webservers
  become: true
  roles:
    - common
    - apache

- name: Configure databases
  hosts: databases
  become: true
  roles:
    - common
    - mysql
EOF
```

## Step 6: Dynamic Inventory for AWS

```
bash
# Install boto3 and ec2 plugin
pip3 install boto3 boto

# Update ansible.cfg for dynamic inventory
cat >> ansible.cfg <<EOF

[inventory]
enable_plugins = aws_ec2, host_list, script, yaml, ini
EOF

# Create AWS EC2 inventory plugin config
cat > inventory.aws_ec2.yml <<EOF
plugin: aws_ec2
aws_access_key: "{{ lookup('env', 'AWS_ACCESS_KEY_ID') }}"
aws_secret_key: "{{ lookup('env', 'AWS_SECRET_ACCESS_KEY') }}"
regions:
  - us-east-1
  - us-west-2
filters:
  tag:Environment:
    - dev
keyed_groups:
  - prefix: tag
    key: tags.Environment
hostnames:
  - private-ip-address
compose:
  ansible_host: public_ip_address | default(private_ip_address)
EOF

# Test dynamic inventory
ansible-inventory -i inventory.aws_ec2.yml --list
```

## Step 7: Ansible Vault for Secrets

```
bash
# Create encrypted file
ansible-vault create vault/secrets.yml

# Edit encrypted file
ansible-vault edit vault/secrets.yml

# View encrypted file
ansible-vault view vault/secrets.yml

# Encrypt existing file
ansible-vault encrypt vault/existing.yml

# Decrypt file
ansible-vault decrypt vault/secrets.yml

# Change password
ansible-vault rekey vault/secrets.yml
```

### Use Vault in Playbook

```
bash
# Create playbook with vault
cat > playbooks/deploy.yml <<EOF
---
- name: Deploy application with secrets
  hosts: webservers
  become: true
  vars_files:
    - vault/secrets.yml
  
  tasks:
    - name: Copy secrets to server
      copy:
        content: "{{ db_password }}"
        dest: /etc/app/secrets.txt
        mode: '0600'
      no_log: true
EOF

# Run with vault password
ansible-playbook playbooks/deploy.yml --ask-vault-pass
```

## Step 8: CI/CD Integration with Ansible

```
bash
# Create Jenkinsfile
cat > Jenkinsfile <<EOF
pipeline {
    agent any
    
    environment {
        ANSIBLE_HOSTS = 'production'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Ansible Lint') {
            steps {
                sh 'pip install ansible-lint'
                sh 'ansible-lint playbooks/'
            }
        }
        
        stage('Ansible Dry Run') {
            steps {
                sh 'ansible-playbook playbooks/site.yml --check --diff'
            }
        }
        
        stage('Deploy to Staging') {
            when { branch 'develop' }
            steps {
                sh 'ansible-playbook playbooks/site.yml --tags staging'
            }
        }
        
        stage('Deploy to Production') {
            when { branch 'main' }
            steps {
                withCredentials([string(credentialsId: 'vault-password', variable: 'VAULT_PASS')]) {
                    sh 'ansible-playbook playbooks/site.yml --ask-vault-pass --tags production'
                }
            }
        }
    }
}
EOF
```

## Conclusion

This project demonstrates:
- Installing and configuring Ansible
- Creating and running Ansible playbooks
- Implementing security hardening
- Using Ansible roles for modular configuration
- Dynamic inventory for AWS EC2
- Ansible Vault for secrets management
- CI/CD integration with Jenkins
