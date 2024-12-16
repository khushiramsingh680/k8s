---
title: "kubeadm-external-etcd"
weight: 21
no_list: true
---
## Topics
# Set up a Highly Available Kubernetes Cluster using kubeadm and external etcd
Follow this documentation to set up a highly available Kubernetes cluster using __Ubuntu 24.04 LTS__.
## Quorum
```sh
For N nodes in a cluster
Quorum = floor(N/2 + 1)
```

## Vagrant Environment
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Master|etcd1.example.com|172.16.16.221|Ubuntu 24.04|2G|1|
|Master|etcd2.example.com|172.16.16.222|Ubuntu 24.04|2G|1|
|Worker|etcd3.example.com|172.16.16.223|Ubuntu 24.04|2G|1|

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

## Dowmload and Insall etcd on all nodes
### Login to all nodes 
```sh
ssh vagrant@172.16.16.22{1..3}
```
### Make yourself root
```sh
sudo -s
```
### Update OS
```sh
apt update && apt install -y haproxy
```
#### Run the below command
On all etcd nodes
```sh
 ETCD_VER=v3.5.1
  wget -q --show-progress "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz"
  tar zxf etcd-v3.5.1-linux-amd64.tar.gz
  mv etcd-v3.5.1-linux-amd64/etcd* /usr/local/bin/
  rm -rf etcd*
```

### Create systemd unit file for etcd service
```sh
NODE_IP=$(ip addr show eth1 | grep inet | awk '{print $2}' | cut -d'/' -f1 |head -1)

ETCD_NAME=$(hostname -s)

ETCD1_IP="172.16.16.221"
ETCD2_IP="172.16.16.222"
ETCD3_IP="172.16.16.223"


cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=etcd

[Service]
Type=exec
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --initial-advertise-peer-urls http://${NODE_IP}:2380 \\
  --listen-peer-urls http://${NODE_IP}:2380 \\
  --advertise-client-urls http://${NODE_IP}:2379 \\
  --listen-client-urls http://${NODE_IP}:2379,http://127.0.0.1:2379 \\
  --initial-cluster-token etcd-cluster-1 \\
  --initial-cluster etcd1=http://${ETCD1_IP}:2380,etcd2=http://${ETCD2_IP}:2380,etcd3=http://${ETCD3_IP}:2380 \\
  --initial-cluster-state new
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
### Enable and Start etcd service
```sh

  systemctl daemon-reload
  systemctl enable --now etcd

```
### Verify Etcd cluster status
```sh
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 member list
```

## ETCD with tls
## On your local workstation (Linux)

#### Generate TLS certificates
##### Download required binaries
```sh

  wget -q --show-progress --https-only --timestamping \
  https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl_1.6.3_linux_amd64 \
  https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssljson_1.6.3_linux_amd64
  
  chmod +x cfssl_1.6.3_linux_amd64 cfssljson_1.6.3_linux_amd64
  sudo mv cfssl_1.6.3_linux_amd64 /usr/local/bin/cfssl
  sudo mv cfssljson_1.6.3_linux_amd64 /usr/local/bin/cfssljson

```
##### Create a Certificate Authority (CA)
#### We then use this CA to create other TLS certificates
```sh
cat > ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "8760h"
        },
        "profiles": {
            "etcd": {
                "expiry": "8760h",
                "usages": ["signing","key encipherment","server auth","client auth"]
            }
        }
    }
}
EOF
cat > ca-csr.json <<EOF
{
  "CN": "etcd cluster",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "GB",
      "L": "England",
      "O": "Kubernetes",
      "OU": "ETCD-CA",
      "ST": "Cambridge"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
##### Create TLS certificates
```sh
ETCD1_IP="172.16.16.221"
ETCD2_IP="172.16.16.222"
ETCD3_IP="172.16.16.223"

cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "localhost",
    "127.0.0.1",
    "${ETCD1_IP}",
    "${ETCD2_IP}",
    "${ETCD3_IP}"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "GB",
      "L": "England",
      "O": "Kubernetes",
      "OU": "etcd",
      "ST": "Cambridge"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd etcd-csr.json | cfssljson -bare etcd
```
##### Copy the certificates to etcd nodes
```sh


declare -a NODES=(172.16.16.221 172.16.16.222 172.16.16.223)

for node in ${NODES[@]}; do
  scp ca.pem etcd.pem etcd-key.pem root@$node: 
done


```
##### Copy the certificates to a standard location on all etcd nodes
```sh
 mkdir -p /etc/etcd/pki
 mv ca.pem etcd.pem etcd-key.pem /etc/etcd/pki/
  ```
##### Create systemd unit file for etcd service
 Set NODE_IP to the correct IP of the machine where you are running this
```sh


NODE_IP=$(ip addr show eth1 | grep inet | awk '{print $2}' | cut -d'/' -f1 |head -1)

ETCD_NAME=$(hostname -s)

ETCD1_IP="172.16.16.221"
ETCD2_IP="172.16.16.222"
ETCD3_IP="172.16.16.223"


cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/pki/etcd.pem \\
  --key-file=/etc/etcd/pki/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/pki/etcd.pem \\
  --peer-key-file=/etc/etcd/pki/etcd-key.pem \\
  --trusted-ca-file=/etc/etcd/pki/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/pki/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${NODE_IP}:2380 \\
  --listen-peer-urls https://${NODE_IP}:2380 \\
  --advertise-client-urls https://${NODE_IP}:2379 \\
  --listen-client-urls https://${NODE_IP}:2379,https://127.0.0.1:2379 \\
  --initial-cluster-token etcd-cluster-1 \\
  --initial-cluster etcd1=https://${ETCD1_IP}:2380,etcd2=https://${ETCD2_IP}:2380,etcd3=https://${ETCD3_IP}:2380 \\
  --initial-cluster-state new
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF


```
##### Enable and Start etcd service
```sh

  systemctl daemon-reload
  systemctl enable --now etcd

```

##### Verify Etcd cluster status
In any one of the etcd nodes
```sh
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/pki/ca.pem \
  --cert=/etc/etcd/pki/etcd.pem \
  --key=/etc/etcd/pki/etcd-key.pem \
  member list
```
#### You can alos user below
```sh
export ETCDCTL_API=3 
export ETCDCTL_ENDPOINTS=https://172.16.16.221:2379,https://172.16.16.222:2379,https://172.16.16.223:2379
export ETCDCTL_CACERT=/etc/etcd/pki/ca.pem
export ETCDCTL_CERT=/etc/etcd/pki/etcd.pem
export ETCDCTL_KEY=/etc/etcd/pki/etcd-key.pem
```
And now its a lot easier
```sh
etcdctl member list
etcdctl endpoint status
etcdctl endpoint health
```


### Provision Kubernetes cluster with kubeadm using external Etcd cluster
### Copy TLS certificates to kubernetes master node
```sh
 scp /etc/etcd/pki/ca.pem /etc/etcd/pki/etcd.pem /etc/etcd/pki/etcd-key.pem root@172.16.16.100:
 ```
 ### Copy etcd certificates to a standard location
 ```sh
  mkdir -p /etc/kubernetes/pki/etcd
  mv ca.pem etcd.pem etcd-key.pem /etc/kubernetes/pki/etcd/
  ```
  ### Create Cluster Configuration
  ```sh
ETCD1_IP="172.16.16.221"
ETCD2_IP="172.16.16.222"
ETCD3_IP="172.16.16.223"

cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "192.168.0.0/16"
etcd:
  external:
    endpoints:
      - https://172.16.16.221:2379
      - https://172.16.16.222:2379
      - https://172.16.16.223:2379
    caFile: /etc/kubernetes/pki/etcd/ca.pem
    certFile: /etc/kubernetes/pki/etcd/etcd.pem
    keyFile: /etc/kubernetes/pki/etcd/etcd-key.pem
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "172.16.16.100"

EOF

```

### Initialize Kubernetes Cluster
```sh
kubeadm init --config kubeadm-config.yaml --ignore-preflight-errors=all
```
### Deploy Calico network
```sh
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

### Cluster join command
```sh
kubeadm token create --print-join-command
```
### Use default admin kubernetes file
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



### How to write and get values to and from etcd 
```sh
etcdctl --endpoints=127.0.0.1:2379  put foo "Hello World!"
etcdctl --endpoints=127.0.0.1:2379  get foo
```