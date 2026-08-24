# Ansible Automation Lab – Multi-Task Playbooks
 
## Overview
 
This project is part of my **#AWSDevOpsRestartJourney** — a hands-on effort to rebuild and strengthen my DevOps skills through practical, real-world automation exercises.
 
In this lab, I wrote two Ansible playbooks to automate common server configuration and maintenance tasks: creating directory/file structures, installing and managing Nginx, configuring users and groups, setting permissions, and running basic system health checks — all without logging into the managed nodes manually.
 
---
 
## Tools & Technologies
 
- **Ansible** – Configuration management and automation
- **Nginx** – Web server
- **Ubuntu** – Managed node OS
- **YAML** – Playbook syntax
---
 
## Project Structure
 
```
ansible-multi-task-lab/
├── multi-tasks.yml               # Task Definition 1
├── webserver-maintenance.yml     # Task Definition 2
├── hosts                         # Inventory file
└── README.md
```
 
---

## Prerequisite

### Setting Up the Inventory File
 
An inventory file is a list of the managed nodes (hosts/IPs, grouped and with connection details) that Ansible runs playbooks against.

Before running either playbook, create a `hosts` file in the same directory and add the IP address (or hostname) of your remote managed node(s), along with the SSH connection details Ansible should use to reach them.
 
```ini
[control]
localhost ansible_connection=local
 
[webservers]
<EC2-PUBLIC-IP> ansible_user=<username> ansible_ssh_private_key_file=~/.ssh/id_rsa
 
```
 
- **`[control]`** — represents the Ansible control node itself (the machine you're running these playbooks from). `ansible_connection=local` tells Ansible to run tasks directly on this host instead of connecting over SSH. You typically won't target this group with these playbooks, but it's useful to have listed for reference or for any local-only tasks later.
- **`[webservers]`** — the remote managed node(s) these playbooks actually configure. 
- Replace `<EC2-PUBLIC-IP>` with your remote machine's actual IP address (or a resolvable hostname).
- Replace `<username>` with the SSH login username for that machine (e.g. `ubuntu`, `ec2-user`).
---

### Setting Up Passwordless SSH to the Remote Machine
 
Ansible connects to managed nodes over SSH, so it's best to set up key-based authentication from your control node (the machine running Ansible) to avoid being prompted for a password on every run.
 
1. **Generate an SSH key pair on the control node** (skip if you already have one):
```bash
   ssh-keygen -t rsa -b 4096
```
   Press Enter to accept the defaults (no passphrase is simplest for automation).
 
2. **Copy your public key to the remote machine:**
```bash
   cat ~/.ssh/id_rsa.pub | ssh ubuntu@<ec2-public-ip> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
   Replace `ubuntu` with the remote machine's SSH username and `192.168.1.100` with its IP address. Enter the remote user's password once when prompted — this installs your public key into the remote machine's `~/.ssh/authorized_keys`.
   copy it manually instead
 
3. **Verify passwordless login works:**
```bash
   ssh ubuntu@192.168.1.100
```
   You should be logged in directly, with no password prompt.
 
Once this is set up, reference the same key in your `hosts` file via `ansible_ssh_private_key_file` (see below) so Ansible can authenticate without a password too.
 
Test connectivity before running the playbooks:
 
```bash
ansible -i hosts all -m ping
```
 
A successful `pong` response confirms Ansible can reach and authenticate to your managed node.

<img src="screenshots/ansible-ping-test.png" alt="Test connection">

---

## Task Definition 1 – `multi-tasks.yml`
 
**Goal:** Automate directory/file creation, install and enable Nginx, manage users/groups, set file permissions, and check disk usage on the managed node.
 
### What it does
- Creates `/tmp/devops` and `/tmp/devops/info.txt` with sample content
- Creates `/tmp/devops/config` and `/tmp/devops/config/app.conf` with app configuration
- Installs, starts, and enables Nginx
- Deploys a custom `index.html` web page
- Creates a `devopsgroup` group and a `devopsuser` user, and adds the user to the group
- Sets `/tmp/devops/info.txt` permissions to `0644`
- Displays disk usage
### Playbook
 
```yaml
---
- name: Multi-Task Server Setup Playbook
  hosts: webservers
  become: true
 
  tasks:
    - name: Create /tmp/devops directory
      ansible.builtin.file:
        path: /tmp/devops
        state: directory
        mode: '0755'
 
    - name: Create /tmp/devops/info.txt with content
      ansible.builtin.copy:
        dest: /tmp/devops/info.txt
        content: |
          Welcome to Ansible
          This file was created using Ansible.
 
    - name: Create /tmp/devops/config directory
      ansible.builtin.file:
        path: /tmp/devops/config
        state: directory
        mode: '0755'
 
    - name: Create /tmp/devops/config/app.conf with content
      ansible.builtin.copy:
        dest: /tmp/devops/config/app.conf
        content: |
          APP_NAME=DevOps
          ENVIRONMENT=Development
 
    - name: Install Nginx
      ansible.builtin.dnf:
        name: nginx
        state: present
        update_cache: true
 
    - name: Start Nginx service
      ansible.builtin.service:
        name: nginx
        state: started
 
    - name: Enable Nginx to start on boot
      ansible.builtin.service:
        name: nginx
        enabled: true
 
    - name: Deploy custom web page
      ansible.builtin.copy:
        dest: /var/www/html/index.html
        content: |
          <html>
          <body>
            <h1>Welcome to my Ansible Lab</h1>
            <p>This page was deployed using Ansible.</p>
          </body>
          </html>
 
    - name: Create devopsgroup group
      ansible.builtin.group:
        name: devopsgroup
        state: present
 
    - name: Create devopsuser and add to devopsgroup
      ansible.builtin.user:
        name: devopsuser
        group: devopsgroup
        state: present
 
    - name: Set permissions on info.txt
      ansible.builtin.file:
        path: /tmp/devops/info.txt
        mode: '0644'
 
    - name: Check disk usage
      ansible.builtin.command: df -h
      register: disk_usage
      changed_when: false
 
    - name: Display disk usage
      ansible.builtin.debug:
        var: disk_usage.stdout_lines

```

### How to Run
 
```bash
# Task Definition 1
ansible-playbook -i hosts multi-tasks.yml
```

Playbook Execution 

<img src="screenshots/multi-tasks-execution.png" alt="Multi tasks playbook execution">

Files, Directories and nginx status

<img src="screenshots/task1-files-nginx-status.png" alt="Task 1 files and directories">