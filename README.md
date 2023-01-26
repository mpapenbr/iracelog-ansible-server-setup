# Installation via Ansible

This guide shows how to setup the iracelog backend on a server.

**Note:** See [iracelog-documentation](https://github.com/mpapenbr/iracelog-documentation) for an overview of further components.

In order to perform this you need Python (minimum 3.8) installed an the system.
See https://python.org for installation instructions.

## Very first start

In the following the term _console_ (or _shell_) describes a Linux-shell.

Windows users should use the Windows Subsystem for Linux (WSL). In case you don't already have it installed you can find the installation instructions [here](https://docs.microsoft.com/de-de/windows/wsl/install). When you setup a Linux distribution please use Ubuntu 20.04.

**Note**: Installing Python along with Ansible directly on Windows turned out to be a little error prone so the easiest way seems to be using the WSL. For the purpose of just running ansible in the WSL it is ok to have WSL 1. In case this is already available there is no need to update to WSL 2

So, from here the base for the the following commands is the Linux shell.

You need certain software available on the host running the ansible scripts (the ansible client).

- Python >= 3.8
- Ansible >= 2.8

### Python

Python should already available on the system. Check the version by

```console
python --version
```

Check the output to verify the version is 3.8 or higher. If the command fails, try

```console
python3 --version
```

If Python is not installed you may follow the instructions on [this post for python](https://linuxize.com/post/how-to-install-python-3-9-on-ubuntu-20-04/) and [this post for python-pip](https://linuxize.com/post/how-to-install-pip-on-ubuntu-20.04/)

### Ansible

Follow the instructions from the [official Ansible documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu)

Once Python and Ansible are installed, run the following commands on the console.

```console
pip install -r py_requirements.txt
ansible-galaxy collection install -r requirements.yml
```

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
  So we use a predefined environment based on Ubuntu, Docker and Traefik. The services needed are wired via docker compose and we use Let's Encrypt for SSL certificates.

  If you're interested in environments for local evaluation on your own machines you may start with [this docker compose setup](https://github.com/mpapenbr/iracelog-deployment).

In case you don't already have an ssh key, here's an example how to get one

```console
ssh-keygen -t ed25519 -f hetzner -a 100
```

You may choose the enter a password for the private key.

This will produce the files `hetzner` (the private key) and `hetzner.pub` (the public key) in the current directory.

**Important:** The files are sensible data. Handle them with care. This is especially true for the private key. Even if you added a password to it you should consider this file as a good-to-protect secret.

The next step provides some basic preparation.

```console
ansible-playbook prep.yml
```

creates the file `groups_vars/all/vault_vars`. Some passwords will be created during the preparation, other values in that file have to be assigned by you.

### Hetzner

You will need to add the public key to a project. Login to hetzner.com and choose the section _Cloud_. After that choose the _Default_ project, select the key symbol on the left side and add the key by clicking on the red button "add ssh-key". Please name it **hetzner** in order to be compatible with the value in `roles/hetzner/create/tasks/order.yml` in the list _ssh_keys_

Next you will need to generate an API-Token.
Next to _SSH-KEYS_ menu you will find a _API-TOKENS_ menu.
Create a token with Read/Write access, give it a name (for example admin-token) and copy the shown token to the file `group_vars/all/vault_vars` for the key _vault_hetzner_secret_

### Netcup

Head over to the "Stammdaten" section (last entry on the left side menu). In the upper area you will find a ">\_ API" menu link. Here you can create an API password and an API key.
Your netcup customer id is shown in brackets behind your name.

Enter the data into the vault variables.

Now is good time to order the domain at netcup. Enter the domain setup
You don't need to do anything else, the following setup will take care about. Just make sure you enter the domain in `group_vars/all/vault_vars` for the key _vault_main_domain_.

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

After this step ansible will create the file hosts.yml similar to this example:

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

When new versions of the components are available, they will be updated in the repository in the file `group_vars/all/vars.yml`.
Assuming you have cloned this repository you just have to issue a `git pull` followed by the ansible update playbook

```console
git pull
ansible-playbook playbooks/update.yml -i hosts.yml
```

**Note:** This will only update the server components. For an overview of all components of the iRacelog project see [this link](https://github.com/mpapenbr/iracelog-documentation)

## Create a database dump

In case you want to create a dump of the database you may issue the following command

```console
ansible-playbook playbooks/getDump.yml -i hosts.yml
```

This will create a database dump and store it at `/tmp/iracelog.dump` on the server. Afterwards the dump will be copied to your local system and stored at the `dumps` directories here.

**Note:** If there is already a database dump from a previous backup the file will be overwritten.
