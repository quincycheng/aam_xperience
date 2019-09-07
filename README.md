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
2. System Update on all three servers
```
sudo apt update
sudo apt upgrade
```

3. Install openssl on `host-1` & `host-2`
```
sudo apt install openssh-server
```

4. Install Docker on `conjur` server
Please refer to https://docs.docker.com/install/linux/docker-ce/ubuntu/ for details

5. Install `docker-compose` & `jq` on `conjur` server 
```
sudo apt install docker-compose jq
```

## Managing the servers using Ansible 

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

To disable host key checking
```
export ANSIBLE_HOST_KEY_CHECKING=False
````

Let's try the playbook
```
ansible-playbook -i inventory playbook.yml
````


## The risk

- Who's view the secrets?
- When the secrets are used?
- How to rotate the secrets?
- What are auditing?

## Securing 

### Install Conjur OSS

ref: https://www.conjur.org/get-started/quick-start/oss-environment/

```
git clone https://github.com/cyberark/conjur-quickstart.git
cd conjur-quickstart
docker-compose pull
docker-compose run --no-deps --rm conjur data-key generate > data_key
export CONJUR_DATA_KEY="$(< data_key)"
docker-compose up -d
docker ps -a
docker-compose exec conjur conjurctl account create myConjurAccount > admin_data
docker-compose exec client conjur init -u conjur -a myConjurAccount
```

The admin password is located in `admin_data` file
```
cat admin_data
```

Login as Conjur admin
```
docker-compose exec client conjur authn login -u admin
```

### Loading Policy & Secrets to Conjur

Policy: root.yml

```
- !policy
  id: db

- !policy
  id: ansible
```

Policy: db.yml
```
- &variables
  - !variable host1/host
  - !variable host1/user
  - !variable host1/pass
  - !variable host2/host
  - !variable host2/user
  - !variable host2/pass

- !group secrets-users

- !permit
  resource: *variables
  privileges: [ read, execute ]
  roles: !group secrets-users

# Entitlements 
- !grant
  role: !group secrets-users
  member: !layer /ansible
```

ansible.yml
```
- !layer
- !host ansible-01
- !grant
  role: !layer
  member: !host ansible-01
```

Load root policy
```
docker cp root.yml conjur_client:/tmp/
docker-compose exec client conjur policy load --replace root /tmp/root.yml
```

Load ansible Policy
```
docker cp ansible.yml conjur_client:/tmp/
docker-compose exec client conjur policy load ansible /tmp/ansible.yml | tee ansible.out

```
Load db Policy
```
docker cp db.yml conjur_client:/tmp/
docker-compose exec client conjur policy load db /tmp/db.yml
```

Let's create secrets and add them to Conjur

Host 1 IP: 
```
docker-compose exec client conjur variable values add db/host1/host "host-1" 
```

Host 1 User name: 
```
docker-compose exec client conjur variable values add db/host1/user "service01" 
```

Host 1 Password: 
```
docker-compose exec client conjur variable values add db/host1/pass "W/4m=cS6QSZSc*nd"
```

Host 2 IP: 
```
docker-compose exec client conjur variable values add db/host2/host "host-2" 
```

Host 2 User name: 
```
docker-compose exec client conjur variable values add db/host2/user "service02" 
```

Host 2 Password: 
```
docker-compose exec client conjur variable values add db/host2/pass "5;LF+J4Rfqds:DZ8"
```


### Integrating Ansible

#### Setting up Role & Lookup plugin
```
ansible-galaxy install cyberark.conjur-lookup-plugin
```


#### Configure SSL and Conjur settings

Download the SSL Certificate 
```
openssl s_client -showcerts -connect conjur:8443 < /dev/null 2> /dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > conjur-demo.pem
```


```
export CONJUR_CERT_FILE="$PWD/conjur-demo.pem"
export CONJUR_ACCOUNT="myConjurAccount"
export CONJUR_APPLIANCE_URL="https://localhost:8443/"
export CONJUR_AUTHN_LOGIN="host/ansible/ansible-01"
export CONJUR_AUTHN_API_KEY="$(tail -n +2 ansible.out | jq -r '.created_roles."myConjurAccount:host:ansible/ansible-01".api_key')"
```

### Updating Ansible Inventory & Playbook


#### Update inventory file
```
[db_servers]
host1
host2
```

#### Update playbook.yml
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

#### Let's run the playbook again
```
ansible-playbook -i inventory playbook.yml
```

### Cleanup

If you want to try it again, you can remote the files created after execute the following commands to remove the Conjur containers:

```
docker-compose down
```
