---
title: "Kubernets Installation From Scatch on rocky 9"
weight: 21
no_list: true
---

## Topics
- Downloading Release Binaries
- Configuring CA
- Installing ETCD
- Configuring API Server
- Configuring Controller Manager
- Configuring Scheduler
- Validating Cluster Status
- Worker Node Configuration
- Configuring Networking
- API to Kubelet RBAC
- Configuring DNS
- Kubelet Preferred Instance Type
###  Downloading Release Binaries
- Kubernetes Binaries
  - Source Code
  - Client Binaries
  - Server Binaries
  - Node Binaries
  - Container Images
- [Download release Binaries](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)
- [Download Etcd](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)
- Install wget and unzip
 ```t
 yum install wget unzip -y
 ```
### Download Server Binaries on master node
```sh
mkdir /root/binaries
cd /root/binaries
wget https://dl.k8s.io/v1.28.7/kubernetes-server-linux-amd64.tar.gz 
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd /root/binaries/kubernetes/server/bin/
```
- Check all master binaries 
```t
ls -l /root/binaries/kubernetes/server/bin
```
- Download etcd in the same folder
```t
cd /root/binaries/
wget https://github.com/etcd-io/etcd/releases/download/v3.5.12/etcd-v3.5.12-darwin-amd64.zip
unzip etcd-v3.5.12-darwin-amd64.zip
cd etcd-v3.5.12-darwin-amd64/
```
- Check etcd relatated binaries
```t
ls -l /root/binaries/tcd-v3.5.12-darwin-amd64/
```
### Download Node Binaries on Worker node
```sh
mkdir /root/binaries
cd /root/binaries
wget https://dl.k8s.io/v1.28.7/kubernetes-node-linux-amd64.tar.gz
tar -xzvf kubernetes-node-linux-amd64.tar.gz
```
- Check all node binaries
```t
ls -l  /root/binaries/node/bin
```

### Configuring kubernetes Certificate Authority
####  Create base directory were all the certificates and keys will be stored

```sh
mkdir /root/certificates
cd /root/certificates
```
####  Creating a private key for Certificate Authority
```sh
openssl genrsa -out ca.key 2048
```
#### Creating CSR
```sh
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
```
####  Self-Sign the CSR
```sh
openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
```
####  See Contents of Certificate
```sh
openssl x509 -in ca.crt -text -noout
```
#### Remove CSR
```sh
rm -f ca.csr
```
### [Installing ETCD](https://github.com/etcd-io/etcd/releases)
#### [Documentation Referred](https://etcd.io/docs/v3.5/op-guide/configuration/)

#### Pre-Requisite: Set SERVER-IP variable
```sh
SERVER_IP=172.16.16.110 (change this to your IP)
echo $SERVER_IP
```
#### Configure the Certificates:
```sh
cd /root/certificates/
openssl genrsa -out etcd.key 2048
```
#### Note: Replace the value associated with IP.1 in the below step.

```sh
cat > etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = ${SERVER_IP}
IP.2 = 127.0.0.1
EOF
```
```sh
openssl req -new -key etcd.key -subj "/CN=etcd" -out etcd.csr -config etcd.cnf

openssl x509 -req -in etcd.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd.crt -extensions v3_req -extfile etcd.cnf -days 1000
```
#### Copy the Certificates and Key to /etc/etcd
```sh
mkdir /etc/etcd
cp etcd.crt etcd.key ca.crt /etc/etcd
```
####  Copy the ETCD and ETCDCTL Binaries to the Path
```sh
cd /root/binaries/etcd-v3.5.12-darwin-amd64/
cp etcd etcdctl /usr/local/bin/
```

####  Configure the systemd File

#### Create a service file:
```sh
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name master-1 \\
  --cert-file=/etc/etcd/etcd.crt \\
  --key-file=/etc/etcd/etcd.key \\
  --peer-cert-file=/etc/etcd/etcd.crt \\
  --peer-key-file=/etc/etcd/etcd.key \\
  --trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${SERVER_IP}:2380 \\
  --listen-peer-urls https://${SERVER_IP}:2380 \\
  --listen-client-urls https://${SERVER_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${SERVER_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster master-1=https://${SERVER_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
#### Start Service:
```sh
systemctl start etcd
systemctl status etcd
systemctl enable etcd
```
#### Verification Commands:

When we try with etcdctl --endpoints=https://127.0.0.1:2379 get foo, it gives unknown certificate authority.
```sh
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd.crt --key=/etc/etcd/etcd.key put test "Test"
```
```sh
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd.crt --key=/etc/etcd/etcd.key get test
```
### Configuring API Server

#### Reference Documentation 

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/

https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/

#### PreRequisites
-  Move the kube-apiserver binary to /usr/local/bin directory.

```sh
cd /root/binaries/kubernetes/server/bin/
cp kube-apiserver /usr/local/bin/
```
Ensure that the SERVER_IP variable is still set.

#### Step 1. Generate Configuration File for CSR Creation.
```sh
cd /root/certificates
```
```sh
cat <<EOF | sudo tee api.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 127.0.0.1
IP.2 = ${SERVER_IP}
IP.3 = 10.0.2.15
EOF
```
#### Generate Certificates for API Server
```sh
openssl genrsa -out kube-api.key 2048
openssl req -new -key kube-api.key -subj "/CN=kube-apiserver" -out kube-api.csr -config api.conf
openssl x509 -req -in kube-api.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-api.crt -extensions v3_req -extfile api.conf -days 1000
```
#### Generate Certificate for Service Account:
```sh
openssl genrsa -out service-account.key 2048
openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 100
```

#### Copy the certificate files to /var/lib/kubernetes directory
```sh
mkdir /var/lib/kubernetes
cp etcd.crt etcd.key ca.crt kube-api.key kube-api.crt service-account.crt service-account.key /var/lib/kubernetes
ls /var/lib/kubernetes
```

#### Creating Encryption key and Configuration
```sh
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```
```sh
cat > encryption-at-rest.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```
```sh
cp encryption-at-rest.yaml /var/lib/kubernetes/encryption-at-rest.yaml
```
#### Creating Systemd service file:

Systemd file:

```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
--advertise-address=${SERVER_IP} \
--allow-privileged=true \
--authorization-mode=Node,RBAC \
--client-ca-file=/var/lib/kubernetes/ca.crt \
--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
--enable-bootstrap-token-auth=true \
--etcd-cafile=/var/lib/kubernetes/ca.crt \
--etcd-certfile=/var/lib/kubernetes/etcd.crt \
--etcd-keyfile=/var/lib/kubernetes/etcd.key \
--etcd-servers=https://127.0.0.1:2379 \
--kubelet-client-certificate=/var/lib/kubernetes/kube-api.crt \
--kubelet-client-key=/var/lib/kubernetes/kube-api.key \
--service-account-key-file=/var/lib/kubernetes/service-account.crt \
--service-cluster-ip-range=10.32.0.0/24 \
--tls-cert-file=/var/lib/kubernetes/kube-api.crt \
--tls-private-key-file=/var/lib/kubernetes/kube-api.key \
--requestheader-client-ca-file=/var/lib/kubernetes/ca.crt \
--service-node-port-range=30000-32767 \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/var/log/kube-api-audit.log \
--bind-address=0.0.0.0 \
--event-ttl=1h \
--service-account-key-file=/var/lib/kubernetes/service-account.crt \
--service-account-signing-key-file=/var/lib/kubernetes/service-account.key \
--service-account-issuer=https://${SERVER_IP}:6443 \
--encryption-provider-config=/var/lib/kubernetes/encryption-at-rest.yaml \
--v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
#### Start the kube-api service:
```sh
systemctl start kube-apiserver
systemctl status kube-apiserver
systemctl enable kube-apiserver
```

### Configuring Controller Manager
#### Generate Certificates:
```sh
cd /root/certificates
```
```sh
openssl genrsa -out kube-controller-manager.key 2048
openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000
```
#### Generating KubeConfig
```sh
cp /root/binaries/kubernetes/server/bin/kubectl /usr/local/bin
```
```sh
kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig
```
```sh
kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig
```
```sh
kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```
#### Copying the files to kubernetes directory
```sh
cp kube-controller-manager.crt kube-controller-manager.key kube-controller-manager.kubeconfig ca.key /var/lib/kubernetes/
```
####  Configuring SystemD service file:
```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
--bind-address=0.0.0.0 \\
--service-cluster-ip-range=10.32.0.0/24 \\
--cluster-cidr=10.200.0.0/16 \\
--kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
--authentication-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
--authorization-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
--leader-elect=true \\
--cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
--cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
--root-ca-file=/var/lib/kubernetes/ca.crt \\
--service-account-private-key-file=/var/lib/kubernetes/service-account.key \\
--use-service-account-credentials=true \\
--v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### Start Controller Manager:
```sh
cp /root/binaries/kubernetes/server/bin/kube-controller-manager /usr/local/bin
systemctl start kube-controller-manager
systemctl status kube-controller-manager
systemctl enable kube-controller-manager
```

### Configuring Scheduler

#### Generate Certificates:
```sh
cd /root/certificates
openssl genrsa -out kube-scheduler.key 2048
openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-scheduler.crt -days 1000
```
#### Generate Kubeconfig file:
```sh
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

#### Copy the scheduler kubeconfig
```sh
cp kube-scheduler.kubeconfig /var/lib/kubernetes/
```
#### Configuring systemd service:
```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --authentication-kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --authorization-kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --bind-address=127.0.0.1 \\
  --leader-elect=true
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
#### Verification:
```sh
cp /root/binaries/kubernetes/server/bin/kube-scheduler /usr/local/bin
systemctl start kube-scheduler
systemctl status kube-scheduler
systemctl enable kube-scheduler
```

###  Validating Cluster Status
#### Generate Certificate for Administrator User
```sh
cd /root/certificates
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out admin.crt -days 1000
```

#### Create KubeConfig file:

Note: Replace the IP address from the below snippet in line 5 with your IP address.


```sh
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${SERVER_IP}:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```
#### Verify Cluster Status
```sh
kubectl get componentstatuses --kubeconfig=admin.kubeconfig
cp /root/certificates/admin.kubeconfig ~/.kube/config
kubectl get componentstatuses
```
####  Verify Kubernetes Objects Creation
```sh
kubectl create namespace test
kubectl get namespace test -o yaml

kubectl create secret generic prod-secret --from-literal=username=admin --from-literal=password=password123

kubectl get secret
```


### Worker Node Configuration

#### Configure Container Runtime. (WORKER NODE)

```sh
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```
```sh
modprobe overlay
modprobe br_netfilter
```
```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
```sh
sysctl --system
```

```t
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

dnf install -y containerd.io

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```
```sh
nano /etc/containerd/config.toml
```
  --> SystemdCgroup = true

```sh
systemctl restart containerd
```

```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```sh
sudo sysctl --system
```

#### Pre-Requisites - 2: (WORKER NODE)
```sh

dnf install socat conntrack ipset
sysctl -w net.ipv4.conf.all.forwarding=1
cd  /root/binaries/kubernetes/node/bin/
cp kube-proxy kubectl kubelet /usr/local/bin
```
#### Generate Kubelet Certificate for Worker Node. (MASTER NODE)

Note:
   1. Replace the IP Address and Hostname field in the below configurations according to your enviornement.
   2. Run this in the Kubernetes Master Node
```sh
cd /root/certificates
```
```sh
cat > openssl-worker.example.com.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = worker.example.com
IP.1 = 172.16.16.111
EOF
```
```sh
openssl genrsa -out worker.example.com.key 2048
```
```sh
openssl req -new -key worker.example.com.key -subj "/CN=system:node:worker.example.com/O=system:nodes" -out worker.example.com.csr -config openssl-worker.example.com.cnf
openssl x509 -req -in worker.example.com.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out worker.example.com.crt -extensions v3_req -extfile openssl-worker.example.com.cnf -days 1000
```

#### Step 2: Generate kube-proxy certificate: (MASTER NODE)
```sh
openssl genrsa -out kube-proxy.key 2048
openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy" -out kube-proxy.csr
openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-proxy.crt -days 1000
```
#### Step 3: Copy Certificates to Worker Node:

This can either be manual approach or via SCP.
Certificates: kubelet, kube-proxy and CA certificate.

In-case you want to automate it, then following configuration can be used.
In the demo, we had made used of manual way.

In-case, you want to transfer file from master to worker node, then you can make use of the following approach:

##### - Worker Node:

##### - Master Node:
- Copy certificate from master node to worker nodes
```sh
scp kube-proxy.crt kube-proxy.key worker.example.com.key worker.example.com.crt ca.crt vagrant@172.16.16.111:/tmp


```
##### - Worker Node:
```sh
mkdir /root/certificates
cd /tmp
mv kube-proxy.crt kube-proxy.key worker.example.com.crt worker.example.com.key ca.crt /root/certificates


```
#### Step 4: Move Certificates to Specific Location. (WORKER NODE)
```sh
mkdir /var/lib/kubernetes
cd /root/certificates
cp ca.crt /var/lib/kubernetes
mkdir /var/lib/kubelet
mv worker.example.com.crt  worker.example.com.key  kube-proxy.crt  kube-proxy.key /var/lib/kubelet/
```
#### Step 5: Generate Kubelet Configuration YAML File: (WORKER NODE)
```sh
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
      clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
runtimeRequestTimeout: "15m"
cgroupDriver: systemd
EOF
```
#### Step 6: Generate Systemd service file for kubelet: (WORKER NODE)

```sh
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
#### Step 7: Generate the Kubeconfig file for Kubelet (WORKER NODE)

```sh
cd /var/lib/kubelet
cp /var/lib/kubernetes/ca.crt .
SERVER_IP=IP-OF-API-SERVER
```
```sh
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${SERVER_IP}:6443 \
    --kubeconfig=worker.example.com.kubeconfig

  kubectl config set-credentials system:node:worker.example.com \
    --client-certificate=worker.example.com.crt \
    --client-key=worker.example.com.key \
    --embed-certs=true \
    --kubeconfig=worker.example.com.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:node:worker.example.com \
    --kubeconfig=worker.example.com.kubeconfig

  kubectl config use-context default --kubeconfig=worker.example.com.kubeconfig
}
```
```sh
mv worker.example.com.kubeconfig kubeconfig
```
### Part 2 - Kube-Proxy

#### Step 1: Copy Kube Proxy Certificate to Directory:  (WORKER NODE)
```sh
mkdir /var/lib/kube-proxy

```
#### Step 2: Generate KubeConfig file:
```sh
{
  kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${SERVER_IP}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```
```sh
mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```
#### Step 3: Generate kube-proxy configuration file: (WORKER NODE)
```sh
cd /var/lib/kube-proxy
```
```sh
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```
#### Step 4: Create kube-proxy service file: (WORKER NODE)
```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### WORKER NODE
```sh
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy
```

#### Step 6: (MASTER NODE)
```sh
kubectl get nodes
```
- Check the log of kubelet
```t
journalctl -u kubelet.service
```