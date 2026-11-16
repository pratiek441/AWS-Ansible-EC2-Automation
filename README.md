# AWS Ansible EC2 Automation

## Project Overview

This project demonstrates configuration management and server automation using **Ansible on AWS EC2 instances**.

Ansible is used to connect to multiple EC2 instances through SSH and automate common system administration tasks, including:

- Updating Linux packages.
- Installing essential packages.
- Installing and configuring Apache Web Server.
- Installing and configuring Docker.
- Managing Linux services.
- Deploying website content.

## 1. AWS Infrastructure Setup

### Launch EC2 Instances

Launch EC2 instances in the same VPC.

**Infrastructure:**
- 1 Ansible Server (Control Node).
- Target EC2 instances (Managed Nodes).
- Operating System: Amazon Linux.
- Create an SSH key pair named `ansible.pem`.

### Security Group Configuration

Configure the required inbound rules:

| Type | Port | Purpose |
|---|---:|---|
| SSH | 22 | Ansible SSH connectivity |
| HTTP | 80 | Apache web access |

**Important:** Allow SSH access from the Ansible server to the managed nodes. Prefer private IP addresses when instances are in the same VPC.

## 2. Install Ansible on Control Node

Connect to the Ansible server using SSH.

### Update packages

```bash
sudo yum update -y
```

### Install Python and pip

```bash
sudo yum install python -y
sudo yum install pip -y
sudo yum install python3-pip -y
```

### Install Ansible

```bash
pip install ansible
```

Search for Ansible package:

```bash
sudo yum search ansible
```

Install Ansible using the package manager:

```bash
sudo yum install ansible.noarch -y
```

Check Ansible version:

```bash
ansible --version
```

*Note: The pip and yum installation methods are alternatives. Use a compatible installation method for your Amazon Linux version.*

## 3. Project Directory Structure

Create the following project structure:

```text
ansible/
│
├── ansible.cfg
│
├── aws/
│   └── ansible.pem
│
├── inventory/
│   └── hosts
│
├── playbooks/
│   ├── apache.yml
│   ├── docker.yml
│   └── update.yml
│
└── roles/
```

Create directories:

```bash
mkdir -p ansible/{aws,inventory,playbooks,roles}
cd ansible
```

## 4. Configure SSH Private Key

Copy your `ansible.pem` key into the `aws` directory.

Set the correct permissions:

```bash
chmod 400 aws/ansible.pem
```

This key is used by Ansible to connect to managed EC2 instances.

## 5. Configure Ansible

Create the configuration file:

```bash
vi ansible.cfg
```

Add the following configuration:

```ini
[defaults]
inventory = ./inventory
remote_user = ec2-user
private_key_file = aws/ansible.pem
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false
```

### Configuration Explanation

| Parameter | Description |
|---|---|
| inventory | Defines inventory location |
| remote_user | Remote EC2 login user |
| private_key_file | SSH private key location |
| host_key_checking | Disables SSH host key checking |
| retry_files_enabled | Disables retry files |
| become | Enables privilege escalation |
| become_method | Uses sudo |
| become_user | Executes tasks as root |

## 6. Configure Inventory

Create the inventory file:

```bash
vi inventory/hosts
```

Add your managed EC2 instances:

```ini
[all]
<DEV_PUBLIC_IP>
<PROD_PUBLIC_IP>

[dev]
<DEV_PUBLIC_IP>

[prod]
<PROD_PUBLIC_IP>
```

Replace the placeholders with your actual EC2 IP addresses.

### Inventory Groups

- `[all]` – All managed EC2 instances.
- `[dev]` – Development server.
- `[prod]` – Production server.

## 7. Test Ansible Connectivity

Test connectivity with all managed instances:

```bash
sudo ansible -m ping all
```

Test development server:

```bash
sudo ansible -m ping dev
```

Test production server:

```bash
sudo ansible -m ping prod
```

Expected output:

```text
SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## 8. Playbook 1 – Update System Packages

Create the playbook:

```bash
vi playbooks/update.yml
```

Add the following YAML:

```yaml
---
- name: Basic setup for Amazon Linux
  hosts: all
  become: true

  tasks:
    - name: Update all packages
      yum:
        name: '*'
        state: latest

    - name: Install essential packages
      yum:
        name:
          - git
          - wget
          - unzip
          - tree
        state: present
```

### Execute Playbook

```bash
sudo ansible-playbook -i inventory/hosts playbooks/update.yml
```

### Purpose

- Updates installed packages.
- Installs Git.
- Installs Wget.
- Installs Unzip.
- Installs Tree.

## 9. Playbook 2 – Install and Configure Apache

Create the playbook:

```bash
vi playbooks/apache.yml
```

Add the following YAML:

```yaml
---
- name: Install and configure Apache on Amazon Linux
  hosts: dev
  become: true

  tasks:
    - name: Install Apache (httpd)
      yum:
        name: httpd
        state: present

    - name: Start and enable Apache service
      systemd:
        name: httpd
        state: started
        enabled: true

    - name: Deploy index.html
      copy:
        content: "<h1>Hello from Ansible on Amazon Linux</h1>"
        dest: /var/www/html/index.html
        mode: '0644'
```

### Execute Playbook

```bash
sudo ansible-playbook -i inventory/hosts playbooks/apache.yml
```

### Verify Apache Service

```bash
sudo systemctl status httpd
```

### Test Website

```bash
curl http://localhost
```

Open the following URL in your browser:

```text
http://YOUR-EC2-PUBLIC-IP
```

Expected output:

```html
<h1>Hello from Ansible on Amazon Linux</h1>
```

**Note:** The playbook uses `hosts: dev` because the inventory defines `[dev]`. You can use `hosts: web` if you create a corresponding `[web]` inventory group.

## 10. Playbook 3 – Install and Configure Docker

Create the playbook:

```bash
vi playbooks/docker.yml
```

Add the following YAML:

```yaml
---
- name: Install Docker on Amazon Linux 2
  hosts: all
  become: true

  tasks:
    - name: Install Docker package
      yum:
        name: docker
        state: present

    - name: Start and enable Docker service
      systemd:
        name: docker
        state: started
        enabled: true

    - name: Add ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes
```

### Execute Playbook

```bash
sudo ansible-playbook -i inventory/hosts playbooks/docker.yml
```

### Verify Docker Service

```bash
sudo systemctl status docker
```

Check Docker version:

```bash
docker --version
```

Check running containers:

```bash
docker ps
```

**Note:** This Docker playbook follows the Amazon Linux 2 installation method. Amazon Linux 2023 may require a different installation procedure. Also, reconnect to the managed instance after adding `ec2-user` to the Docker group.

## 11. Useful Ansible Commands

| Command | Purpose |
|---|---|
| `ansible --version` | Check Ansible version |
| `ansible-inventory -i inventory/hosts --graph` | Display inventory groups |
| `ansible -m ping all` | Test all managed nodes |
| `ansible -m ping dev` | Test development server |
| `ansible -m ping prod` | Test production server |
| `ansible-playbook --syntax-check playbooks/update.yml` | Validate YAML syntax |
| `ansible-playbook -i inventory/hosts playbooks/update.yml` | Execute update playbook |
| `ansible-playbook -i inventory/hosts playbooks/apache.yml` | Execute Apache playbook |
| `ansible-playbook -i inventory/hosts playbooks/docker.yml` | Execute Docker playbook |

## 12. Ansible Architecture

```text
                AWS CLOUD
                    |
                    |
          Ansible Control Node
              (EC2 Server)
                    |
                    |
               SSH / Port 22
                    |
          Ansible Inventory
                    |
          ---------------------
          |                   |
       DEV Server          PROD Server
        (EC2)                (EC2)
          |                   |
     Apache Setup         Package Setup
          |
      Docker Setup
          |
     Website Deployment
```

## 13. What I Learned

Through this hands-on project, I practiced:

- Setting up Ansible on AWS EC2.
- Configuring SSH authentication using PEM keys.
- Creating Ansible inventory groups.
- Configuring `ansible.cfg`.
- Connecting to remote EC2 instances.
- Writing YAML-based Ansible playbooks.
- Automating Linux package management.
- Installing and configuring Apache.
- Installing and managing Docker.
- Managing Linux services using systemd.
- Deploying website content using Ansible.

## 14. Security Best Practices

- Never upload `ansible.pem` to GitHub.
- Use private IP addresses for communication within the same VPC.
- Restrict SSH access to trusted sources.
- Avoid disabling SSH host key checking in production.
- Use trusted users when granting Docker group membership.

Add the following to `.gitignore`:

```gitignore
# Private SSH keys
*.pem

# Ansible retry files
*.retry

# Python cache
__pycache__/
*.pyc
```

## Conclusion

This project demonstrates how Ansible can automate infrastructure configuration and application setup across multiple AWS EC2 instances.

Instead of manually installing packages and configuring services on every server, Ansible allows administrators and DevOps engineers to manage infrastructure consistently through reusable YAML playbooks.
