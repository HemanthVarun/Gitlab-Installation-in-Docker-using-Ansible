# Gitlab-Installation-in-Docker-using-Ansible

## Project Overview
This project uses Ansible playbooks to automate the installation of Docker and deployment of GitLab in a containerized environment. It helps simplify setup and management of GitLab on a target host.

## Project Structure:
```
Gitlab-Installation-in-Docker-using-Ansible/
│
├── ansible_playbooks/
│   ├── backup.yml               # Backs up GitLab data
│   ├── docker_install.yml       # Installs Docker
│   ├── gitlab-deploy.yml        # Deploys GitLab in Docker
│   └── inventory.ini            # Contains managed node details
│
└── README.md                    # Project explanation
```

The installation and deployment of GitLab within a Docker container on an AWS EC2 instance can be automated with Ansible, as this project illustrates.  Two EC2 instances must be set up: one as a Managed Node (the target machine where GitLab will be deployed) and another as a Control Node (which runs Ansible).

## setup
1. EC2 Instances
   Control Node:
   Instance Type: t3.micro
   Purpose: Runs Ansible to manage configurations and deployments.
   
   Managed Node:
   Instance Type: t3.medium
   Additional EBS Volume: 25 GB (gp3)
   Purpose: Hosts Docker and GitLab deployment.
   
Both EC2 instances use Ubuntu as the base operating system.

## Configuration Steps
Step 1: Connect to the Instances.
Use SSH connect both instance.

Step 2: Install Ansible and sshpass on Control Node.
Ansible is required to run playbooks and manage the target node, "sshpass" is used for non-interactive SSH login via password.
```
sudo apt update && sudo apt install ansible sshpass -y
sshpass -V
```

Step 3: Enabling Password Authentication on Managed Node.

```sudo vim /etc/ssh/sshd_config.d/60-cloudimg-setting.conf```  # for enabling password authentication between managed node and control node.

PasswordAuthentication no                                 # will be in first line

PasswordAuthentication yes                                # change it to yes from no

```sudo systemctl restart ssh```                                # restart the ssh service


```sudo passwd ubuntu```                                       # set password 


## Ansible_Playbooks
1. Docker-Install.yml
```
---
- name: Install Docker on Ubuntu
  hosts: all
  become: yes

  tasks:
    - name: Install required packages
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
        state: present
        update_cache: yes

    - name: Add Docker GPG key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker APT repository
      apt_repository:
        repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable
        state: present

    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Docker
      apt:
        name: docker-ce
        state: present

    - name: Start and enable Docker service
      systemd:
        name: docker
        state: started
        enabled: yes

    - name: Create docker group
      group:
        name: docker
        state: present

    - name: Add user to docker group
      user:
        name: "{{ ansible_user_id }}"
        groups: docker
        append: yes
```

2. Gitlab-deploy.yml
```
---
- hosts: target_servers
  become: yes
  tasks:
    - name: Pull GitLab Docker Image
      docker_image:
        name: gitlab/gitlab-ce
        source: pull

    - name: Run GitLab Container
      docker_container:
        name: gitlab
        image: gitlab/gitlab-ce
        state: started
        restart_policy: always
        ports:
          - "80:80"
          - "443:443"
          - "2222:22"
        env:
          GITLAB_ROOT_PASSWORD: "Pbsvpbsv"
        volumes:
          - "/srv/gitlab/config:/etc/gitlab"
          - "/srv/gitlab/logs:/var/log/gitlab"
          - "/srv/gitlab/data:/var/opt/gitlab"
```

3. backup.yml
```
---
- name: Backup GitLab Data
  hosts: target_servers
  become: yes  # Ensure tasks run with root privileges

  tasks:
    - name: Create GitLab backup script (Root)
      ansible.builtin.copy:
        dest: "/root/gitlab_backup.sh"
        mode: "0755"
        content: |
          #!/bin/bash
          BACKUP_DIR="/root/gitlab_backups"
          TIMESTAMP=$(date +'%Y%m%d_%H%M%S')
          mkdir -p $BACKUP_DIR
          tar -czvf $BACKUP_DIR/gitlab_backup_$TIMESTAMP.tar.gz /srv/gitlab/data/git-data/

    - name: Ensure backup directory exists
      ansible.builtin.file:
        path: "/root/gitlab_backups"
        state: directory
        mode: "0755"

    - name: Schedule Daily Backup via Cron (Root)
      ansible.builtin.cron:
        name: "GitLab Daily Backup"
        job: "/root/gitlab_backup.sh"
        minute: "0"
        hour: "2"

```

## Inventory file
```
[target_servers]
[Managed-Node-ip] ansible_user=ubuntu ansible_ssh_pass=ubuntu ansible_become_pass=ubuntu
```

## Running the Playbooks                                                                                       # Commands to run playbooks
```
ansible-playbook -i /root/Ansible_Project/inventory.ini --ask-pass /root/Ansible_Project/docker-install.yml
```
```
ansible-playbook -i /root/Ansible_Project/inventory.ini --ask-pass /root/Ansible_Project/gitlab-deploy.yml
```
```
ansible-playbook -i /root/Ansible_Project/inventory.ini --ask-pass /root/Ansible_Project/backup.yml
```

## Accessing the gitlab through browser:
```
http://<managed-node-public-ip>
```

## Conclusion
By completing this project, we have:
Provisioned EC2 instances for automation
Configured Ansible to manage remote hosts using password authentication
Installed Docker and deployed GitLab using Ansible
Created a reliable backup mechanism via Ansible
This project shows how powerful and efficient configuration management tools like Ansible can be in managing cloud-based infrastructure.

