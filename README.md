HA Kubernetes Lab
===================

The following environment creates a HA Kubernetes cluster (1 bind9 DNS server, 3 k8s controllers, 3 k8s workers, 3 etcd3 hosts, 1 haProxy LB) onto bare metal servers running Ubuntu 16.04. This is implemented with Virtualbox VMs, but should cover different scenarios. This use case is for building a local HA Kubernetes cluster onto bare metal servers.

NOTE: For demonstration and limited resources on a macbook pro with 16 GB memory, there is only 1 worker node, worker0.lan.
Change the ansible/hosts.yaml ans/or static_inventory to include worker1.lan, and worker2.lan

###Overview:
* Create virtual hosts based on Kelsey Hightowers guide [here:] (https://github.com/kelseyhightower/kubernetes-the-hard-way) This is meant to be a bare metal guide, so that Google Cloud isn't necessary, and an on premises solution can be used.

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
The hosts are built with the vagrant box 'bento/ubuntu-16.04' and utilize systemd unit files.

NOTE: ansible user's ssh private key file is using $HOME. Adjust accordingly.
```
cat ansible/hosts.yaml
```
The ansible/hosts.yaml file is where I modify the count and specs for the vagrant boxes. For instance, I removed several workers to make the builder lighter on my dev machine.

###Run the Ansible Playbooks:
```
ansible/run_playbooks.sh
```
###Ansible playbooks overview:
Below are the individual playbooks. There are additional playbooks and roles that aren't in the ./run_playbooks.sh script.

####Infrastructure DNS server
Installs DNS in the environment, and is not necessary if DNS already exists
```
ansible-playbook provision_core_servers.yaml -i inventory.py
```

####Infrastructure DNS resolution
Updates the cluster node's resolv.conf to point to the previous playbook's DNS server(s), this is not necessary if DNS resolution already exists
```
ansible-playbook update_resolv.yaml -i inventory.py
```

####Haproxy
Install haproxy LBs (This takes the place of the GCE frontend utilized in the GCE LB). Using only one is a single point of failure. The solution is to provision multiple LBs and use keepalive.d Virtual IPs to ensure high availability, but is not implemented here.
```
ansible-playbook provision_lb_servers.yaml -i inventory.py
```

####TLS Creation
Create and serve SSL certs with Apache (This is not secure, but is useful to easily share certs)
```
ansible-playbook create_and_expose_ssl_certs.yaml -i inventory.py
```

####TLS Certs download
Download SSL certs (This cluster utilizes TLS, ABAC, Token Auth, therefore here we download the exposed certs.)
```
ansible-playbook download_certs.yaml -i inventory.py
```

####Etcd3
Install etcd3 (etcd3 with TLS)
```
ansible-playbook provision_etcd_servers.yaml -i inventory.py
```

####k8s controller nodes
Install k8s master controller servers (kube-apiserver, kube-controller-manager, kube-scheduler)
```
ansible-playbook provision_k8s_controllers.yaml -i inventory.py
```

####k8s worker nodes
Install k8s worker servers (kube-proxy, kubelet)
```
ansible-playbook provision_k8s_workers.yaml -i inventory.py
```

#After Running the Playbooks

###etcd3

Check cluster health on the etcd servers
```
etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
```
###K8s Controller servers
```
kubectl get componentstatuses
```

###k8s Worker servers
export the KUBECONFIG var to use kubectl on the worker nodes.
```
export KUBECONFIG="/var/lib/kubelet/kubeconfig"
```

###CNI Networking setup
As an alternative to Flannel, Calico, Weave, etc. K8s utilizes a CNI plugin networking solution [here](https://github.com/containernetworking/cni).

Also we are utilizing CNI Plugin architecture with the kubelet process, which allows you to swap containerization at runtime. Therefore Docker does not need to be reconfigured with an overlay network. We'll use basic L3 networking as a proof of concept.

####L3 Routes
Add routes to the workers and controllers, depending on which host you are on, you will not want to alter the newly created kubernetes CNI cbr0 default route. (This route isn't configured until pods are created.) Otherwise, add the necessary routes to the cluster. Eventually I'll create network interface ansible playbooks for this, so that further HA and fail over can be achieved, i.e. NIC bonding.

NOTE: The routes below are approximate. Until the workers cbr0 iface is created, you don't really know which L3 routes to add on each worker node. The extra playbook listed below fails occasionally, because the declarative routes overlap.
```
route add -net 10.200.0.0 netmask 255.255.255.0 gw 10.240.0.30
route add -net 10.200.1.0 netmask 255.255.255.0 gw 10.240.0.31
route add -net 10.200.2.0 netmask 255.255.255.0 gw 10.240.0.32
```

Let's begin a test of the k8s functionality
```
kubectl create -f <file>
```
Config:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeName: worker0
      containers:
      - name: nginx
        image: gcr.io/google_containers/nginx
        ports:
        - containerPort: 80
```

###KubeDNS

The kubedns containers had forwarding issues. The pods weren't assigned endpoints and continually crashed. The logs seemed to point to iptables and DNS resolution issues. Additionally I increased the memory to 4GB per node to solve the /readiness checks that kept returning 503 errors in the logs.

While adding more memory and adding routing on the nodes seemed to fix the DNS issues I was experiencing, you can further troubleshoot the kubeDNS by checking resolv.conf and nslookups:

Get pod names:
```
root@worker0:/home/vagrant# kubectl --namespace=kube-system get pods
NAME                 READY     STATUS    RESTARTS   AGE
kube-dns-v18-bd9z8   2/3       Running   0          33s
kube-dns-v18-tnt4r   2/3       Running   0          33s
```

Check resolv in pods:
```
kubectl --namespace=kube-system exec kube-dns-v18-tnt4r -c kubedns -- cat /etc/resolv.conf
```

## Additional Ansible Playbooks
####Infrastructure L3 Routing
The hosts need to resolve each other's k8s subnets, so we create routes on the hosts as mentioned above.

NOTE: These routes are not persistent across reboots, and I'll need to come up with better logic to test the routes when the docker cbr0 bridge is created.
I don't know how the k8s decides which subnet to assign to cbr0 yet, so the declarative assignment in vars/makevault.yml doesn't always work out.
```
ansible-playbook provision_k8s_l3_routes.yaml -i inventory.py
```

####Kubernetes Addons (kubeDNS, Heapster, Dashboard)
To create the addons k8s templates and create with kubectl.
```
ansible-playbook provision_k8s_addons.yaml -i inventory.py
```

####Docker Private Registry servers
This creates a private registry v2, on a host named dreg0.lan. We use openssl to create a self signed SSL cert which needs to be shared on the worker nodes. The registry files are located in /docker-registry by default. Listed in the ```vars/makevault.yml``` file are htpasswd credentials for logging into the registry.
```
ansible-playbook provision_docker_registry_servers.yaml -i inventory.py
```

####Expose Docker Registry SSL cert 
We install Apache on the Registry Server and copy the self signed cert into the public www folder. Bad practice in production, but this is a lab.
```
ansible-playbook expose_docker_registry_ssl_cert.yaml -i inventory.py
```

####Download Registry cert on worker nodes
We exposed the cert with Apache which is not secure by any means. We run update-ca-certificates after placing the cert where Docker expects.
```
ansible-playbook download_docker_registry_certs.yaml -i inventory.py
```

####Docker build app to push to registry for use
This automates an nginx app build from a Dockerfile, which can be used to expose an ingress point into the cluster. It's a simple app that returns the details of the it's container from an Nginx web server. Useful for testing load balancing and additional features in Kubernetes.
```
ansible-playbook provision_docker_nginx.yaml -i inventory.py
```

####Remove TLS certs
Purge the k8s certificates if you need to build new TLS certs.
```
ansible-playbook purge_ssl_certs.yaml -i inventory.py
```
