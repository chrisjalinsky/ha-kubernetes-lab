CoreOS PXE Boot Environment
===================

The following environment PXE boots a Kubernetes cluster (1 master, 2 nodes) onto bare metal servers. This is currently implemented with 3 Virtualbox VMs, but should cover different scenarios. This use case is for building a local Kubernetes cluster onto bare metal servers.

Included is a custom Docker build web application that is exposed and load balanced on the local network. It will be pulling it's image from a private Docker Registry, which will be built as well.

###Overview:
* Create hosts based on Kelsey Hightowers guide [here:] (https://github.com/kelseyhightower/kubernetes-the-hard-way)
###Dependencies:
* Ansible >= 2.0
* Vagrant >= 1.8.1
* Virtualbox >= 5.0
* Virtualbox Extension Pack [https://www.virtualbox.org/wiki/Downloads](Download here) - Needed to enable Virtualbox PXE Booting to work correctly

You probably only need to have Ansible installed to get this environment up and running. Esp. if you are deploying to bare-metal or another hypervisor. You'll have to create a simple inventory file. Look at ansible/static_inventory to see what groups the playbooks use.

Create Environment with Vagrant
===============================

I use vagrant-hostsupdater plugin to write to my Macbook's /etc/hosts file for DNS resolution. This is so I can resolve hostnames in my browser. There is a playbook that installs DNS into the private network, as well as updates resolv.conf to utilize the newly created DNS server. You can skip these playbooks if your environment contains DNS resolution.
```
vagrant up
```

###The vagrant environment:
Here is a yaml representation of the dynamic inventory. The idea is to use a JSON API to retrieve dynamic inventory, it's currently static:
The hosts are built with the vagrant box 'bento/ubuntu-16.04' and utilize systemd unit files

NOTE: ansible user's ssh private key file is using $HOME. Adjust accordingly. The PXE servers havent been added here.
```
cat ansible/hosts.yaml
```

###Run the Ansible Playbooks:
```
ansible/run_playbooks.sh
```
###Ansible playbooks overview:

Installs DNS in the environment, and is not necessary if DNS already exists
```
ansible-playbook provision_core_servers.yaml -i inventory.py
```

Updates the cluster node's resolv.conf to point to the previous playbook's DNS server(s), this is not necessary if DNS resolution already exists
```
ansible-playbook update_resolv.yaml -i inventory.py
```

Create and serve SSL certs with Apache
```
ansible-playbook create_ssl_certs.yaml -i inventory.py
```

Download SSL certs
```
ansible-playbook download_ssl_certs.yaml -i inventory.py
```

Install etcd3
```
ansible-playbook provision_etcd_servers.yaml -i inventory.py
```

Install k8s master controller services
```
ansible-playbook provision_k8s_controllers.yaml -i inventory.py
```

Install k8s worker services
```
ansible-playbook provision_k8s_workers.yaml -i inventory.py
```

Install haproxy LBs
```
ansible-playbook provision_lb_servers.yaml -i inventory.py
```
