HA Kubernetes Lab
===================

The following environment creates a HA Kubernetes cluster (1 bind9 DNS server, 3 k8s controllers, 3 k8s workers, 3 etcd3 hosts, 1 haProxy LB) onto bare metal servers running Ubuntu 16.04. This is implemented with Virtualbox VMs, but should cover different scenarios. This use case is for building a local HA Kubernetes cluster onto bare metal servers.

###Overview:
* Create hosts based on Kelsey Hightowers guide [here:] (https://github.com/kelseyhightower/kubernetes-the-hard-way)

###Dependencies:
* Ansible >= 2.0
* Vagrant >= 1.8.5
* Virtualbox >= 5.0

You probably only need to have Ansible installed to get this environment up and running. Esp. if you are deploying to bare-metal or another hypervisor. You'll have to create a simple inventory file. Look at ansible/static_inventory to see what groups the playbooks use.

Create Environment with Vagrant
===============================

I use vagrant-hostsupdater plugin to write to my Hypervisor's /etc/hosts file for DNS resolution. This is so I can resolve hostnames in my browser. There is a playbook that installs DNS into the private network, as well as updates resolv.conf to utilize the newly created DNS server. You can skip these playbooks if your environment contains DNS resolution.
```
vagrant up
```

###The vagrant environment:
Here is a yaml representation of the dynamic inventory. The idea is to use a JSON API to retrieve dynamic inventory, it's currently static:
The hosts are built with the vagrant box 'bento/ubuntu-16.04' and utilize systemd unit files

NOTE: ansible user's ssh private key file is using $HOME. Adjust accordingly.
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

Create and serve SSL certs with Apache (This is not secure, but is useful to easily share certs)
```
ansible-playbook create_and_expose_ssl_certs.yaml -i inventory.py
```

Download SSL certs (This cluster utilizes TLS, ABAC, Token Auth, therefore here we download the exposed certs.)
```
ansible-playbook download_ssl_certs.yaml -i inventory.py
```

Install etcd3 (etcd3 with TLS)
```
ansible-playbook provision_etcd_servers.yaml -i inventory.py
```

Install k8s master controller servers (kube-apiserver, kube-controller-manager, kube-scheduler)
```
ansible-playbook provision_k8s_controllers.yaml -i inventory.py
```

Install k8s worker servers (kube-proxy, kubelet)
```
ansible-playbook provision_k8s_workers.yaml -i inventory.py
```

Install haproxy LBs (This takes the place of the GCE frontend utilized in the GCE LB)
```
ansible-playbook provision_lb_servers.yaml -i inventory.py
```

Purge the certificates
```
ansible-playbook purge_ssl_certs.yaml -i inventory.py
```

###etcd3

Check cluster health on etcd servers
```
etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
```
###K8s Controller servers
```
kubectl get componentstatuses
```

###k8s Worker servers

Get pod names
```
root@worker0:/home/vagrant# kubectl --namespace=kube-system get pods
NAME                 READY     STATUS    RESTARTS   AGE
kube-dns-v18-bd9z8   2/3       Running   0          33s
kube-dns-v18-tnt4r   2/3       Running   0          33s
```
Check resolv in pods
```
kubectl --namespace=kube-system exec kube-dns-v18-tnt4r -c kubedns -- cat /etc/resolv.conf
```

