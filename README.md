# ansible-k8s-centos7

Ansible playbooks to deploy kubernetes on Centos7.

**Setup HA control plane/multi master node with Haproxy**


## preq

### install required ansible roles

```bash
ansible-galaxy install githubixx.containerd
```

### Update the inventory

update the [inventory](kubernetes-multi-master\inventory\cyverse)

### Ping for ssh connections

```bash
ansible -i inventory/ -m ping all --user root
```

## Get Started

### Run playbooks

```bash
# to make sure ssh works for all the nodes
ansible -i inventory/ -m ping all --user root

# setup the firwall 
ansible-playbook -i inventory/ firewalld-config.yml --user root

# install all dependencies for hosts
## This will also setup the haproxy for the master node proxy
ansible-playbook -i inventory/ provision-nodes.yml --user root
```

### Initialize the first Master node

This will join the first Master node to the cluster

--control-plane-endpoint (IP of the loadbalancer haproxy)

--upload-certs (to copy the certs)

```bash
# ssh root@MASTER-NODE-1.com

# check first with --dry-run
kubeadm init --control-plane-endpoint="<HAPROXY-IP>:6443" --upload-certs \
--pod-network-cidr=10.0.10.0/24 --dry-run

# run
kubeadm init --control-plane-endpoint="<HAPROXY-IP>:6443" --upload-certs \
--pod-network-cidr=10.0.10.0/24
```

#### add/copy kube config for Master node

```bash
# ssh root@MASTER-NODE-1.com
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Join the second Master node

```bash
# ssh root@MASTER-NODE-2.com

kubeadm join <HAPROXY-IP>:6443 --token 6ixj5w.dj7eai8zw1kx8y34 \
--discovery-token-ca-cert-hash sha256:341fdd8f7b14bdbbb5a440fd3d97336290c2467b055ed32b7551e0ca78a3b19a \
--control-plane --certificate-key 9ec2409309a253ec9de57899049b9a333bcfb9952e0720c5542c239453d5df29
# if the key is expired run this command from root@MASTER-NODE-1.com to get new key
```

#### add/copy kube config for Master node

```bash
# ssh root@MASTER-NODE-2.com
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Add worker nodes

You can run the same command to each **worker node** in order to join them to the cluster.

```bash
# ssh root@WORKER-NODE-1.com
kubeadm join <HAPROXY-IP>:6443 --token 6ixj5w.dj7eai8zw1kx8y34 \
--discovery-token-ca-cert-hash sha256:341fdd8f7b14bdbbb5a440fd3d97336290c2467b055ed32b7551e0ca78a3b19a
```

# Extra

## install weave CNI 
```bash
# https://www.weave.works/docs/net/latest/kubernetes/kube-addon/
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

## Tainting and Labeling VICE Worker Nodes
Once you have your nodes joined the cluster:

The CyVerse Discovery Environment uses taints and labels to ensure that some nodes are used exclusively for VICE
analyses. To mark a node as a VICE worker node, run this command on any node that has access to the Kubernetes API:

**Run this command to label node**
```bash
kubectl label nodes vice-w1.cyverse.at vice=true
```

**To prevent non-VICE pods from running on a node, run this command:**
```bash
kubectl taint nodes vice-w1.cyverse.at vice=only:NoSchedule
```

**check if labeld**
```bash
kubectl get nodes -l vice=true
```

## COPY kubeconfig to your local machine
```bash
# this will allow you to access your cluster from your local machine.
scp root@k8s-c1.cyverse.at:/etc/kubernetes/admin.conf ~/.kube/config
```
