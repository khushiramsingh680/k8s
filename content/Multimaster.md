---
title: "Multi Master kubernetes"
weight: 21
no_list: true
---
## Topics
# Set up a Highly Available Kubernetes Cluster using kubeadm
Follow this documentation to set up a highly available Kubernetes cluster using __Ubuntu 24.04 LTS__.

This documentation guides you in setting up a cluster with two master nodes, one worker node and a load balancer node using HAProxy.

## Vagrant Environment
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Load Balancer|loadbalancer.example.com|172.16.16.100|Ubuntu 24.04|1G|1|
|Master|kmaster1.example.com|172.16.16.101|Ubuntu 24.04|2G|2|
|Master|kmaster2.example.com|172.16.16.102|Ubuntu 24.04|2G|2|
|Worker|kworker1.example.com|172.16.16.201|Ubuntu 24.04|1G|1|

 * Password for the **root** account on all these virtual machines is **kubeadmin**
 * Perform all the commands as root user unless otherwise specified

## Pre-requisites
If you want to try this in a virtualized environment on your workstation
* Virtualbox installed
* Vagrant installed
* Host machine has atleast 8 cores
* Host machine has atleast 8G memory

## Bring up all the virtual machines
```t
vagrant up
```

## Set up load balancer node
##### Install Haproxy
### Login to LoadBalancer node 
```sh
ssh vagrant@172.16.16.100
```
### Make yourself root
```sh
sudo -s
```
### Update OS
```sh
apt update && apt install -y haproxy
```
#### Configure haproxy
Append the below lines to **/etc/haproxy/haproxy.cfg**
```sh
frontend kubernetes-frontend
    bind 172.16.16.100:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server kmaster1 172.16.16.101:6443 check fall 3 rise 2
    server kmaster2 172.16.16.102:6443 check fall 3 rise 2
```
##### Restart haproxy service
```sh
systemctl restart haproxy
```

## On all kubernetes nodes (kmaster1, kmaster2, kworker1)
##### Disable Firewall of enabled
```sh
ufw disable
```
##### Disable swap
```sh
swapoff -a; sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

```
##### Update sysctl settings for Kubernetes networking
```sh
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
#### Add kernel Parameters
```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```
##### Install Containerd engine
```sh
apt update -y
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
systemctl restart containerd
```
### Kubernetes Setup
##### Add Apt repository
```sh
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```
##### Install Kubernetes components
```sh
apt update && apt install -y kubeadm kubelet kubectl
```
## On any one of the Kubernetes master node (Eg: kmaster1)
##### Initialize Kubernetes Cluster
```sh
kubeadm init --control-plane-endpoint="172.16.16.100:6443" --upload-certs --apiserver-advertise-address=172.16.16.101 --pod-network-cidr=192.168.0.0/16
```
Copy the commands to join other master nodes and worker nodes.
##### Deploy Calico network
```sh
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
```

## Join other nodes to the cluster (kmaster2 & kworker1)
- Use the respective kubeadm join commands you copied from the output of kubeadm init command on the first master.

- IMPORTANT: You also need to pass --apiserver-advertise-address to the join command when you join the other master node.

#### Example 
```sh
kubeadm join 172.16.16.100:6443 --apiserver-advertise-address=172.16.16.102 --token uvk94a.j4qzcty1pkj56qy6 \
        --discovery-token-ca-cert-hash sha256:c7f16013787f799fd5db13f1ddbf377a5cb1f30d22b4e34445fe9fc569fe5a0c \
        --control-plane --certificate-key f060afdc1dd0efad58f2f0e97ab881ee68fc6cb85e228b34db78673a911d2c5c
```
## Downloading kube config to your local machine
On your host machine
```sh
mkdir ~/.kube
scp root@172.16.16.101:/etc/kubernetes/admin.conf ~/.kube/config
```
Here Password for root account is kubeadmin (if you used Vagrant setup)

## Verifying the cluster
```sh
kubectl cluster-info
kubectl get nodes
kubectl get cs
```

### Commands I used to verify cluster 
- check the logs for kubelet 
```sh
sudo journalctl -u kubelet --since "1 hour ago"
sudo journalctl -u kubelet -b -p err
journalctl -u kubelet -f
```
- Check all etcd members
```sh
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```
- Check the health of etcd cluster
```sh
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```
