---
title: "Kubernetes-on-Centos"
weight: 21
no_list: true
---

### Kubernetes Installation on Centos 9

### Common Steps for master and worker node
```t
sudo tee /etc/sysctl.d/99-k8s-cri.conf >/dev/null <<EOF
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
```
- update the setting without restarting the OS
```t
 sysctl --system
 modprobe overlay

 modprobe br_netfilter

 echo -e overlay\\nbr_netfilter > /etc/modules-load.d/k8s.conf
 install from EPEL
```
- Install iptables legacy
```t
 dnf --enablerepo=epel -y install iptables-legacy

 alternatives --config iptables


There are 2 programs which provide 'iptables'.

  Selection    Command
-----------------------------------------------
*+ 1           /usr/sbin/iptables-nft
   2           /usr/sbin/iptables-legacy

 switch to [iptables-legacy]
Enter to keep the current selection[+], or type selection number: 2
```

 - set Swap off setting
 ```t

 swapoff -a

 vi /etc/fstab

 comment out the Swap line
/dev/mapper/cs-swap     none                    swap    defaults        0 0

```


- Install Cri and other software
```t

 dnf -y install centos-release-okd-4.14

 sed -i -e "s/enabled=1/enabled=0/g" /etc/yum.repos.d/CentOS-OKD-4.14.repo

 dnf --enablerepo=centos-okd-4.14 -y install cri-o
 systemctl enable --now crio
 cat <<'EOF' > /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=0
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

```

- Install kubelet and cri-tools
```t
 dnf --enablerepo=kubernetes -y install kubeadm kubelet cri-tools iproute-tc container-selinux
 ```
 - Restart kubelet
 ```t
 systemctl enable kubelet 
 
 ```
 
 
 
### On Master node:
 - Bootstrap Kubernetes Cluster
 ```t
  kubeadm init --apiserver-advertise-address=10.224.29.85  --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/crio/crio.sock

```

-  Run the below command on master nodes to run kubectl command
```t
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf
```

- Execute the below command on worker nodes

   - On worker  node 
```t
kubeadm join 10.224.29.85:6443 --token sfkbuw.h6oloizivsdsovbs \
        --discovery-token-ca-cert-hash sha256:664454f0d923630af2ac8e65bf14ff1e63bd1040b6d8aaf5665d016d3a6dbdf8
```
