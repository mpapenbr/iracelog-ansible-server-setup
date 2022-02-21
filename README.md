# Installation via Ansible

This guide shows how to setup the iracelog backend on a server.
In order to perform this you need Python (minimum 3.8) installed an the system. 
See python.org for installation instructions.
## Very first start
In the following the term *console* (or *shell*) describes a Linux-shell or the command prompt on Windows.

You need certain software available on the host running the ansible scripts (the ansible client).

Run the following commands on the console.
```console
pip install -r py_requirements.txt
ansible-galaxy collection install -r requirements.yml
```

In the directory `group_vars/all` copy the file `vault_vars.sample` to `vault_vars`

## Sample with server at Hetzner and DNS at Netcup

In case you don't already have an ssh key, here's an example how to get one

```console
ssh-keygen -t ed25519 -f hetzner -a 100
```
You may choose the enter a password for the private key.

This will produce the files `hetzner` (the private key) and `hetzner.pub` (the public key) in the current directory.

**Important:** The files are sensible data. Handle them with care. This is especially true for the private key. Even if you added a password to it you should consider this file as a good-to-protect secret.

You will need to add the public key to a project. Choose the *Default* project, select the key symbol on the left side and add the key by clicking on the red button "add ssh-key". Please name it 'hetzner' in order to be compatible with the value in `roles/hetzner/create/tasks/order.yml` in the list *ssh_keys*

Next you will need to generate an API-Token. 
Next to *SSH-KEYS* menu you will find a *API-TOKENS* menu. 
Create a token with Read/Write access, give it a name (for example admin-token) and copy the shown token to the file `group_vars/all/vault_vars` for the key *vault_hetzner_secret*



Order the server
```console
ansible-playbook playbooks/order.yml -i hosts.yml
```

## Install the software on the server


```console
ansible-playbook playbooks/init.yml -i hosts.yml
```





