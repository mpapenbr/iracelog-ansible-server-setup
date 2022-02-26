# Installation via Ansible

This guide shows how to setup the iracelog backend on a server.
In order to perform this you need Python (minimum 3.8) installed an the system. 
See https://python.org for installation instructions.
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

**Disclaimer** 
- Why Hetzner and Netcup? 

  Hetzner provides Infrastructure as a Server (IaaS). They charge per hour and the rates a quite fair. In this example we use a CX21 server which costs about 0.01€/h or 6€/month at most.

  Netcup offers .de domains for 5€/year. Hetzner also offers domains but they charge ~12€/year
  (Information as of 2022-02-26)

  Another important aspect is they both provide API access to their services and there are ansible roles available.

- Why not using a cloud provider?

  So far this is only at experimentable stage.

- Can I use my own server environment?

  This setup is designed for people who like to evalute this software and are not that familar with setting up linux servers in the internet. 

  See [advanced setup](README-advanced.md) for further information. 


In case you don't already have an ssh key, here's an example how to get one

```console
ssh-keygen -t ed25519 -f hetzner -a 100
```
You may choose the enter a password for the private key.

This will produce the files `hetzner` (the private key) and `hetzner.pub` (the public key) in the current directory.

**Important:** The files are sensible data. Handle them with care. This is especially true for the private key. Even if you added a password to it you should consider this file as a good-to-protect secret.

### Hetzner
You will need to add the public key to a project. Login to hetzner.com and choose the section *Cloud*. After that choose the *Default* project, select the key symbol on the left side and add the key by clicking on the red button "add ssh-key". Please name it **hetzner** in order to be compatible with the value in `roles/hetzner/create/tasks/order.yml` in the list *ssh_keys*

Next you will need to generate an API-Token. 
Next to *SSH-KEYS* menu you will find a *API-TOKENS* menu. 
Create a token with Read/Write access, give it a name (for example admin-token) and copy the shown token to the file `group_vars/all/vault_vars` for the key *vault_hetzner_secret*

### Netcup
Head over to the "Stammdaten" section (last entry on the left side menu). In the upper area you will find a ">_ API" menu link. Here you can create an API password and an API key.
Your netcup customer id is shown in brackets behind your name.

Enter the data into the vault variables.

Now is good time to order the domain at netcup. Enter the domain setup 
You don't need to do anything else, the following setup will take care about. Just make sure you enter the domain in `group_vars/all/vars.yml` for the key *main_domain*.

Once the domain is available (this may take some time, check this via netcup adminstration) you can proceed with the next steps.

Order the server
```console
ansible-playbook playbooks/order.yml -i hosts.yml
```

After this step you will see something like this (the values shown may be different):

```console
....
TASK [hetzner/create : print status] ***************************************************************************************
ok: [localhost] => {
    "msg": [
        "created server",
        "id: 18298251",
        "type:        cx21",
        "status:      running",
        "datacenter:  fsn1-dc14",
        "IPv4:        142.132.235.223",
        "IPv6:        2a01:4f8:c010:36bf::/64"
    ]
}
...
```
Copy the IPv4 value to the hosts.yml file for the ansible_host key like this:

```console
all:
  children:
    prod:
      hosts:
        server:
          ansible_host: 142.132.235.223
          ansible_user: root

```

### Removing the server

If you don't need the server anymore you can remove it with this command

```console
ansible-playbook playbooks/remove.yml -i hosts.yml
```

This command will do the following:
- remove the DNS entries which were created during the ordering process 
- remove the server 



## Install the software on the server

```console
ansible-playbook playbooks/init.yml -i hosts.yml
```
## Updating the software 

When new versions of the components are available, edit the *iracelog_*-variables in `group_vars/all/vars.yml`. 
Afterwards issue the follwing command

```console
ansible-playbook playbooks/update.yml -i hosts.yml
```



