# ansible-role-create-users

[![Build Status](https://travis-ci.org/ryandaniels/ansible-role-create-users.svg?branch=master)](https://travis-ci.org/ryandaniels/ansible-role-create-users)

Role to manage users on linux.  
Manage users in the user list config file (list is in the file vars/secret).  
Add users, change passwords, lock/unlock user accounts, manage sudo access (per user), add ssh key(s) for sshkey based authentication, add users (append) to group(s) and group will be created if doesn't exist.  
This is done on a per "group" basis (Ansible group variables), as set in the config file. The group comes from the Ansible group as set for a server in the inventory file.  

More detailed example can be found in the blog post: [User Management with Ansible](https://ryandaniels.ca/blog/ansible-user-management/)  

Note: Deleting users is not done on purpose.  

## Distros tested

* Ubuntu 18.04 / 16.04
* CentOS / RHEL: 7.x, 6.5, 5.9

## Dependencies

Requires Ansible 2.6 (due to previous [bug 20096](https://github.com/ansible/ansible/issues/20096) with un-expiring users)

## ansible-vault

Use ansible-vault to encrypt sensitive info from git.

```bash
cat vars/secret
#encrypt if cleartext (before git commit/push)
ansible-vault encrypt vars/secret

#Edit encrypted file:
ansible-vault edit vars/secret

vi .vaultpass
-Enter the password for Ansible Vault from Password Safe
chmod 600 .vaultpass
vi ansible.cfg
#Insert the following lines
[defaults]
vault_password_file = ./.vaultpass
```

## .gitignore

```bash
vi .gitignore
#Insert the following lines
.vaultpass
.retry
secret
*.secret
```

## How to generate password

* on Ubuntu - Install "whois" package

```bash
mkpasswd --method=SHA-512
```

* on RedHat - Use Python

```bash
python -c 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))'
```

## Default Settings

```yaml
---
#Note: 'debug_enabled_default: true' will put hashed passwords in the output.
debug_enabled_default: false
default_update_password: on_create
default_shell: /bin/bash
```

## User Settings

File Location: vars/secret

* **username**: username - no spaces **(required)**
* **uid**: The numerical value of the user's ID (optional)
* **home**: The path for the new home. Default is unset (optional)
* **user_state**: present|lock **(required)**
* **password**: sha512 encrypted password (optional). If not set, password is set to "!"
* **update_password**: always|on_create (optional, default is on_create to be safe).  
  **WARNING**: when 'always', password will be change to password value.  
  If you are using 'always' on an **existing** users, **make sure to have the password set**.
* **comment**: Full name and Department or description of application (optional) (But you should set this!)
* **groups**: Comma separated list of groups the user will be added to (appended). If group doesn't exist it will be created on the specific server. This is not the primary group (primary group is not modified)
* **shell**: path to shell (optional, default is /bin/bash)
* **ssh_key**: ssh key for ssh key based authentication (optional)  
* **authorized_key_file**: Where to put the SSH authorized key file. If not set, default is ~/.ssh/authorized_keys (optional)
* **manage_dir**: Either change the owner of the directory to the key owner, or keep as it is. Default no. 

  NOTE: 1 key can go on single line, but if multiple keys, use formatting below from first example.
* **exclusive_ssh_key**: yes|no (optional, default: no)  
  **WARNING**: exclusive_ssh_key: yes - will remove any ssh keys not defined here! no - will add any key specified.
* **use_sudo**: yes|no (optional, default no)
* **use_sudo_nopass**: yes|no (optional, default no). yes = passwordless sudo.
* **servers**: sub-element list of servers where changes are made. **(required)**  
  These are the Ansible groups from your Ansible inventory file. In below examples, `webserver` would be the 3 servers in the `webserver` Ansible inventory `webserver1`, `webserver2`, and `webserver3`.  

Note:
  You can have duplicate usernames on different servers, if you want to have different settings. See below example of testuser102 has sudo on servers defined as the `webserver` group in the inventory, but no sudo on the `database` group.

## Example Ansible Inventory file

```yaml
[webserver]
webserver1
webserver2
webserver3

[database]
db1
db2
db3

[monitoring]
monitor1
```

## Example config file (vars/secret)

```yaml
---
users:
  - username: testuser101
    password: $6$/y5RGZnFaD3f$96xVdOAnldEtSxivDY02h.DwPTrJgGQl8/MTRRrFAwKTYbFymeKH/1Rxd3k.RQfpgebM6amLK3xAaycybdc.60
    update_password: on_create
    home: /data/users/testuser101
    comment: Test User 100
    shell: /bin/bash
    authorized_key_file: /etc/ssh/keys/testuser101
    manage_dir: no
    ssh_key: |
      ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx8crAHG/a9QBD4zO0ZHIjdRXy+ySKviXVCMIJ3/NMIAAzDyIsPKToUJmIApHHHF1/hBllqzBSkPEMwgFbXjyqTeVPHF8V0iq41n0kgbulJG testuser101@server1
      ssh-rsa AAAA.... testuser101@server2
    exclusive_ssh_key: yes
    use_sudo: no
    use_sudo_nopass: no
    user_state: present
    servers:
      - webserver
      - database
      - monitoring

  - username: testuser102
    password: $6$F/KXFzMa$ZIDqtYtM6sOC3UmRntVsTcy1rnsvw.6tBquOhX7Sb26jxskXpve8l6DYsQyI1FT8N5I5cL0YkzW7bLbSCMtUw1
    update_password: always
    comment: Test User 101
    groups: testcommon, testgroup102web
    shell: /bin/sh
    use_sudo: yes
    user_state: present
    servers:
      - webserver

  - username: testuser102
    password: $6$F/KXFzMa$ZIDqtYtM6sOC3UmRntVsTcy1rnsvw.6tBquOhX7Sb26jxskXpve8l6DYsQyI1FT8N5I5cL0YkzW7bLbSCMtUw1
    update_password: always
    comment: Test User 101
    groups: testcommon, testgroup102db
    shell: /bin/sh
    user_state: present
    servers:
      - database

  - username: testuser103
    password: $6$wBxBAqRmG6O$gPbg9hYShkuIe3YKMFujwiKsPKZHNFwoK4yCyTOlploljz53YSoPdCn9P5k8Qm0z062Q.8hvJ6DnnQQjwtrnS0
    user_state: present
    servers:
      - webserver

  - username: testuser104
    ssh_key: ssh-rsa AAAB.... test103@server
    exclusive_ssh_key: no
    use_sudo: no
    user_state: present
    servers:
      - webserver
      - monitoring

  - username: testuser105
    uid: 1099
    password: $6$XEnyI5UYSw$Rlc6tXtECtqdJ3uFitrbBlec1/8Fx2obfgFST419ntJqaX8sfPQ9xR7vj7dGhQsfX8zcSX3tumzR7/vwlIH6p/
    ssh_key: ssh-rsa AAAB.... test107@server
    use_sudo: no
    user_state: lock
    servers:
      - webserver
      - database
```

## Example Playbook create-users.yml

```bash
---
- hosts: '{{inventory}}'
  vars_files:
    - vars/secret
  become: yes
  roles:
  - create-users
```

## Prep

* install ansible
* create keys
* ssh to client to add entry to known_hosts file
* configure client server authorized_keys
* run ansible commands

## Usage

Create all users

```bash
ansible-playbook create-users.yml --ask-vault-pass --extra-vars "inventory=all-dev" -i hosts
```
