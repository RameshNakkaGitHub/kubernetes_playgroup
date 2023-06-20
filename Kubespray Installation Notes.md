#  Kubetnetes Cluster Deployment Using Kubespray

## Steps

## 1. Create VM's for Cluster:

    2 VM's for Master Nodes - master1, master2
    4 VM's for worker Nodes - worker1..4
    3 VM's for etcd Nodes - etcd1..3
    1 VM for Ansible Node - client , kubespray will be run from here
    Add all the VM's IP's into /etc/hosts of client node
    Take note of IP's of all VM's


## 2. Setup SSH: On `client VM`


Login into client node as root

Generate ssh key :
```bash
ssh-keygen
```
Copy the key to all the VM's
```bash
ssh-copy-id root@<ip address>
```
Install keychain, so that you can avoid entering password for each ssh
```bash
sudo apt-get intall keychain
```
add below lines to .bashrcin your home directory
```bash
keychain id_rsa
. ~/.keychain/`uname -n` -sh
```

Test if you login into VM's from client VM using ssh
ssh root@<master1 ip>


## 3. Setup Python3 and pip3 : On `client VM`

```bash
which python3
which pip3
```
If python3 is not installed 
```bash
apt-get update && apt-get install python3 -y
apt-get install python3-pip -y
```



## 4. Install GIT if not present On `client VM`

```bash
sudo apt-intstall git
```


## 5. Clone kubespray repo [Kubespray](https://github.com/kubernetes-sigs/kubespray.git)  On `client VM`

```bash
got clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```
```
cluster.yml - create cluster
remove-node.yml - remove k8s node
upgrade-cluster.yml - upgrade k8s cluster with new changes 
reset.yml - reset your cluster
```
```bash
vi ansible.cfg
```
add log parameter

```bash
log= ./kubespray.log
```


## 6. Intall kubespray requirements On `client VM`

```bash
cd ~/kubespray
```

If you are using python virtual env:
```bash
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
ANSIBLE_VERSION=2.12
virtualenv  --python=$(which python3) $VENVDIR
source $VENVDIR/bin/activate
cd $KUBESPRAYDIR
pip install -U -r requirements-$ANSIBLE_VERSION.txt    
```

If you are directly running the pip3 on host:
```bash
sudo pip3 install -r requirements.txt
which ansible  --  /usr/local/bin
```
make sure you ansible is added to $PATH
reload the changes
```bash
# newgrp -
```

## 7. Ansible Inventory On `client VM`

```bash
cd /kubespray/inventory/sample
cat inventory.ini
cd ../../..
pwd 
mkdir -p kubespray/inventory/my-cluster-inventory
cp -rfp kubespray/inventory/sample kubespray/inventory/my-cluster-inventory
cd kubespray/inventory/my-cluster-inventory && ls
```

## 8. Modify inventory.ini On `client VM`

Configure 'ip' variable to bind kubernetes services on a
different ip than the default iface
We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.

    [all]
    master1 ansible_host=<master1_ip> ip=<master1_ip>
    master2 ansible_host=<master2_ip> ip=<master2_ip>
    worker1 ansible_host=<worker1_ip> ip=<worker1_ip>
    worker2 ansible_host=<worker2_ip> ip=<worker2_ip>
    worker3 ansible_host=<worker3_ip> ip=<worker3_ip>
    worker4 ansible_host=<worker4_ip> ip=<worker4_ip>
    etcd1 ansible_host=<etcd1_ip> ip=<etcd1_ip> etcd_member_name=etcd1
    etcd2 ansible_host=<etcd2_ip> ip=<etcd2_ip> etcd_member_name=etcd2
    etcd3 ansible_host=<etcd3_ip> ip=<etcd3_ip> etcd_member_name=etcd3        

    [kube_control_plane]
    master1
    master2

    [etcd]
    etcd1
    etcd2
    etcd3

    [kube_node]
    worker1
    worker2
    worker3
    worker4

    [calico_rr]

    [k8s_cluster:children]
    kube_control_plane
    kube_node
    calico_rr


## 9. Update Ansible inventory file with inventory builder On `client VM`

```bash

declare -a IPS=(List of ip's in the order)
CONFIG_FILE=inventory/my-cluster-inventory/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

check the inventory file generated
```bash
cat ./inventory/my-cluster-inventory/hosts.yml
```
Review and change parameters under ``inventory/my-cluster-inventory/group_vars``
```bash
cat ./inventory/my-cluster-inventory/group_vars/all/all.yml
cat ./inventory/my-cluster-inventory/group_vars/k8s_cluster/k8s-cluster.yml
```

## 10.  Create custom cluster configuration file On `client VM`

```bash

cd kubespray/inventory/my-cluster-inventory
touch cluster-config.yml
vi cluster-config.yml
```
   enter below 

    cluster_name: k8s-ha-cluster

## 11. Provision Cluster On `client VM`

```bash
ansible-playbook -i ./inventory/my-cluster-inventory/hosts.yaml -e  @../inventory/my-cluster-inventory/cluster-config.yml --become --become-user=root cluster.yml
```

## 12. Download the kube config file On `client VM`

```bash
mkdir ~/.kube
scp root@<master1_ip>:/etc/kubernetes/admin.conf ~./kube/config
kubectl --version
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --short
```

## 13. Test the Cluster On `client VM`

```bash
kubectl cluster-info
kubectl get nodes
kubectl -n kube-system get pods
```

## 14. Test Calico Network On `client VM`
Create 1st busybox pod

```bash
kubectl run myshell -it --rm --image busybox -- sh
kubectl get pods -o wide
hostname -i
ping pod2_ip
```


Create 2nd busybox pod 
```bash
kubectl run myshell -it --rm --image busybox -- sh
kubectl get pods -o wide
hostname -i
ping pod1_ip
```
    

    


































    





    

    


    
    





    

    
    