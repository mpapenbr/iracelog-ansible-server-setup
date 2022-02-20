# Installation via Ansible

## Very first start
We need certain software available on the host running the ansible scripts (the ansible client).

Run the setup script
```console
./setup.sh
```

## Sample with server at Hetzner and DNS at Netcup

Order the server
```console
ansible-playbook playbooks/order.yml -i inventory_prod.yml
```

Base initialization of the server
```console
ansible-playbook playbooks/init.yml -i inventory_prod.yml
```




