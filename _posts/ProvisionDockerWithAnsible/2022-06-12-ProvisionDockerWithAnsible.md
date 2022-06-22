---
title: Provision a Linux Server With Docker Using Anislbe
date: 2022-06-12 12:00:00 -500
categories: [Infrastructure as Code,Ansible]
tags: [ansible,linux,docker,automation,docker-compose]
---

Using Ansible to provision remote servers with Docker and Docker-compose.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Structure](#structure)
- [Files](#files)
    - [provisioning/hosts.yml](#provisioninghostsyml)
    - [provisioning/site.yml](#provisioningsiteyml)
    - [provisioning/host-vars/remote](#provisioninghost-varsremote)
    - [provisioning/roles/setup/handlers/main.yml](#provisioningrolessetuphandlersmainyml)
    - [provisioning/roles/setup/tasks/docker.yml](#provisioningrolessetuptasksdockeryml)
    - [provisioning/roles/setup/tasks/main.yml](#provisioningrolessetuptasksmainyml)
- [Build](#build)
- [Test](#test)

# Prerequisites

First of all, local server should be able to access to remote server over SSH so let's do it.


Generating a SHH key
```bash
# Current status on local
ansible@local:~$ ls -l ~/.ssh/
-rw------- 1 ansible ansible 389 Mar 24 09:17 authorized_keys
 
# Generate SSH keypair on local
ansible@local:~$ ssh-keygen -t rsa
 
# New status on local
ansible@local:~$ ls -l ~/.ssh/
-rw------- 1 ansible ansible  389 Mar 24 09:17 authorized_keys
-rw------- 1 ansible ansible 1679 Mar 24 09:29 id_rsa
-rw-r--r-- 1 ansible ansible  394 Mar 24 09:29 id_rsa.pub
```

Transferring public key
```bash
# Current status on remote
ubuntu@remote:~$ ls -l ~/.ssh/
-rw------- 1 ubuntu ubuntu 389 Mar 24 09:02 authorized_keys
 
# Current authorised users on remote
ubuntu@remote:~$ cat ~/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC16 vagrant
 
# Copy public SSH key from local
ansible@local:~$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDDUkN ansible@local
 
# Transfer local public SSH key over remote
ubuntu@remote:~$ echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDDUkN ansible@local' >> ~/.ssh/authorized_keys
 
# Current authorised users on remote
ubuntu@remote:~$ cat ~/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC16 vagrant
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDDUkN ansible@local
```

Testing SSH connection
```bash
ansible@local:~$ ssh ubuntu@192.168.99.30
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-92-generic x86_64)
 
Last login: Sat Mar 24 09:09:36 2018 from 10.0.2.2
ubuntu@remote:~$
```

Install Ansible on local server
```bash
ansible@local:~$ sudo apt-add-repository ppa:ansible/ansible
ansible@local:~$ sudo apt-get update
ansible@local:~$ sudo apt-get install ansible
 
ansible@local:~$ ansible --version
ansible 2.5.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/ubuntu/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.12 (default, Dec  4 2017, 14:50:18) [GCC 5.4.0 20160609]
  ```

# Structure
This is the file and folder structure we are aiming for after creating all of our `yaml` files.
```
$ tree
.
└── provisioning
    ├── host_vars
    │   └── remote
    ├── hosts.yml
    ├── roles
    │   └── setup
    │       ├── handlers
    │       │   └── main.yml
    │       └── tasks
    │           ├── docker.yml
    │           └── main.yml
    └── site.yml
```

# Files

#### provisioning/hosts.yml
```yaml
all:
  hosts:
    remote:
      ansible_connection: ssh
      ansible_user: ubuntu # Remote user
      ansible_host: 192.168.99.30 # Remote host
      ansible_port: 22
```

### provisioning/site.yml
```yaml

---
# This playbook sets up whole stack.
 
- name: Configurations to "remote" host
  hosts: remote
  remote_user: ubuntu # Remote user
  become: yes
  roles:
    - setup
```

### provisioning/host-vars/remote
```
---
# Variables listed here are applicable to "setup" role
 
ansible_python_interpreter: /usr/bin/python3
 
remote_user: ubuntu
docker_group: docker
```

### provisioning/roles/setup/handlers/main.yml
```yaml
---
# This playbook contains common handlers that can be called in "setup" tasks.
 
# sudo systemctl enable docker
- name: Start docker on boot
  systemd:
    name: docker
    state: started
    enabled: yes
```

### provisioning/roles/setup/tasks/docker.yml
```yaml
---
# This playbook contains docker actions that will be run on "remote" host.
 
# sudo apt-get install *
- name: Install docker packages
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  with_items:
    - apt-transport-https
    - ca-certificates
    - curl
    - software-properties-common
  tags:
    - docker
 
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
- name: Add Docker s official GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present
  tags:
    - docker
 
# sudo apt-key fingerprint 0EBFCD88
- name: Verify that we have the key with the fingerprint
  apt_key:
    id: 0EBFCD88
    state: present
  tags:
    - docker
 
# sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable"
- name: Set up the stable repository
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable
    state: present
    update_cache: yes
  tags:
    - docker
 
# sudo apt-get update
- name: Update apt packages
  apt:
    update_cache: yes
  tags:
    - docker
 
# sudo apt-get install docker-ce=18.03.*
- name: Install docker
  apt:
    name: docker-ce=18.03.*
    state: present
    update_cache: yes
  notify: Start docker on boot
  tags:
    - docker
 
# sudo groupadd docker
- name: Create "docker" group
  group:
    name: "{{ docker_group }}"
    state: present
  tags:
    - docker
 
# sudo usermod -aG docker ubuntu
- name: Add remote "ubuntu" user to "docker" group
  user:
    name: "{{ remote_user }}"
    group: "{{ docker_group }}"
    append: yes
  tags:
    - docker
 
# sudo apt-get install docker-compose=1.8.*
- name: Install docker-compose
  apt:
    name: docker-compose=1.8.*
    state: present
    update_cache: yes
  tags:
    - docker
```

### provisioning/roles/setup/tasks/main.yml
```yaml
---
# This playbook contains all actions that will be run on "local" host.
 
# sudo apt-get update
- name: Update apt packages
  apt:
    update_cache: yes
  tags:
    - system
 
# Import docker tasks
- name: Import docker tasks
  include_tasks: docker.yml
 
# sudo apt-get autoclean
- name: Remove useless apt packages from the cache
  apt:
    autoclean: yes
  tags:
    - system
 
# sudo apt-get autoremove
- name: Remove dependencies that are no longer required
  apt:
    autoremove: yes
  tags:
    - system
```

# Build
```
$ ansible-playbook provisioning/site.yml -i provisioning/hosts.yml
 
PLAY [Configurations to "remote" host] *************************************************
 
TASK [Gathering Facts] *****************************************************************
ok: [remote]
 
TASK [setup : Update apt packages] *****************************************************
changed: [remote]
 
TASK [setup : Import docker tasks] *****************************************************
included: /var/www/html/application/provisioning/roles/setup/tasks/docker.yml for remote
 
TASK [setup : Install docker packages] *************************************************
ok: [remote] => (item=[u'apt-transport-https', u'ca-certificates', u'curl', u'software-properties-common'])
 
TASK [setup : Add Docker's official GPG key] *******************************************
changed: [remote]
 
TASK [setup : Verify that we have the key with the fingerprint] ************************
ok: [remote]
 
TASK [setup : Set up the stable repository] ********************************************
changed: [remote]
 
TASK [setup : Update apt packages] *****************************************************
changed: [remote]
 
TASK [setup : Install docker] **********************************************************
changed: [remote]
 
TASK [setup : Create "docker" group] ***************************************************
ok: [remote]
 
TASK [setup : Add remote "ubuntu" user to "docker" group] ******************************
changed: [remote]
 
TASK [setup : Remove useless apt packages from the cache] ******************************
ok: [remote]
 
TASK [setup : Remove dependencies that are no longer required] *************************
ok: [remote]
 
TASK [setup : Install docker-compose] **************************************************
changed: [remote]
 
RUNNING HANDLER [setup : Start docker on boot] *****************************************
ok: [remote]
 
PLAY RECAP *****************************************************************************
remote                     : ok=14   changed=7    unreachable=0    failed=0
```

# Test
```bash
# SSH into remote server
ansible@local:~$ ssh ubuntu@192.168.99.30
 
# Docker compose version
ubuntu@remote:~$ docker-compose --version
docker-compose version 1.8.0, build unknown
 
# Docker version
ubuntu@remote:~$ docker -v
Docker version 18.03.0-ce, build 0520e24
 
# Check if "ubuntu" user member of "docker" group
ubuntu@remote:~$ groups
docker adm dialout cdrom floppy sudo audio dip video plugdev netdev lxd
 
# Check if "ubuntu" user can run "docker" commands
ubuntu@remote:~$ docker ps
CONTAINER ID     IMAGE     COMMAND     CREATED     STATUS     PORTS     NAMES
```