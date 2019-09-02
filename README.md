# AAM Xperience

Welcome again to AAM Xperience!   This page contains everything you need for the hands-on session.

## Our goal 
Let's install an configuration management tool for managing our servers.
We will use Ansible as an example.
First we will simply configure it without securing the embedded secrets and show you the risk.
Then we will install Conjur and integrate it with Ansible.
At the end of the session, the risk of embedded secrets will be elmimated. 


## What have been prepared
1. Install 3 Ubuntu 18.04 LTS systems, named `conjur`,`host-1` and `host-2`
1. System Update on all three servers
```
sudo apt update
sudo apt upgrade
```

2. Install openssl on `host-1` & `host-2`
```
sudo apt install openssh-server
```

2. Install Docker on `conjur` server
Please refer to https://docs.docker.com/install/linux/docker-ce/ubuntu/ for details


## Steps 

1. Let's install Ansible on `conjur` server
ref: https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#latest-releases-via-apt-ubuntu

```
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

2. Create service accounts on `host-1` & `host-2`

On `host-1`:
```
sudo useradd -m -d /tmp service01
sudo passwd service01
Enter new UNIX password: W/4m=cS6QSZSc*nd
Retype new UNIX password: W/4m=cS6QSZSc*nd
```


On `host-2`:
```
sudo useradd -m -d /tmp service02
sudo passwd service02
Enter new UNIX password: 5;LF+J4Rfqds:DZ8
Retype new UNIX password: 5;LF+J4Rfqds:DZ8
```

3. Create `inventory` & playbooks

create an inventory file
```
[db_servers]
host-1 ansible_connection=ssh ansible_ssh_user=service01 ansible_ssh_pass=W/4m=cS6QSZSc*nd
host-2 ansible_connection=ssh ansible_ssh_user=service02 ansible_ssh_pass=5;LF+J4Rfqds:DZ8
```

create a playbook, called playbook.yml
```
- hosts: db_servers
  tasks:
    - name: Get user name
      shell: whoami
      register: theuser

    - name: Get host name
      shell: hostname
      register: thehost

    - debug: msg="I am {{ theuser.stdout }} at {{ thehost.stdout }}"
```




















create a playbook, called playbook.yml
```
- hosts: db_servers
  roles:
    - role: cyberark.conjur-lookup-plugin
  vars:
      ansible_connection: ssh      
      ansible_host: "{{ lookup('retrieve_conjur_variable', 'db/' + inventory_hostname+ '/host') }}"
      ansible_user: "{{ lookup('retrieve_conjur_variable', 'db/' + inventory_hostname+ '/user') }}"
      ansible_ssh_pass: "{{ lookup('retrieve_conjur_variable', 'db/' + inventory_hostname+ '/pass') }}"

  tasks:
    - name: Get user name
      shell: whoami
      register: theuser

    - name: Get host name
      shell: hostname
      register: thehost

    - debug: msg="I am {{ theuser.stdout }} at {{ thehost.stdout }}"
```
