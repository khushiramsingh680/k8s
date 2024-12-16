---
title: "The Object Store for AI Data Infrastructure"
weight: 21
no_list: true
---
## Topics

MinIO is a high-performance, S3 compatible object store. It is built for
large scale AI/ML, data lake and database workloads. It is software-defined
and runs on any cloud or on-premises infrastructure. MinIO is dual-licensed
under open source GNU AGPL v3 and a commercial enterprise license.

### Simple
Simplicity is the foundation for exascale data infrastructure - both technically and operationally. No other object store lets you go from download to production in less time.

### High Performance
MinIO is the world's fastest object store with published GETs/PUTs results that exceed 325 GiB/sec and 165 GiB/sec on 32 nodes of NVMe drives and a 100GbE network.

### Kubernetes-Native
With a native Kubernetes operator integration, MinIO supports all the major Kubernetes distributions on public, private and edge clouds.

### AI Ready
MinIO was built for AI and works out of the box with every major AI/ML technology. From predictive models to GenAI, MinIO delivers the performance and scalability to power the AI enterprise.
### LAB
-  Create a virtual machine 
```bash
git clone https://gitlab.com/container-and-kubernetes/kubernetes-2024.git
cd kubernetes-2024/
vagrant up

```
- You will have a vm ready within 5 mins of time 
- Login to Linux vm
```bash
ssh vagrant@172.16.16.105
Enter password: vagrant
```
-  Switch from vagrant user of root user and password is also vagrant
```bash
 sudo -s
```

#### Use the following commands to download the latest stable MinIO RPM and install it.
```sh
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio-20240606093642.0.0-1.x86_64.rpm -O minio.rpm
sudo dnf install minio.rpm
```
#### Use the following commands to download the latest stable MinIO DEB and install it:
```sh
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20240606093642.0.0_amd64.deb -O minio.deb
sudo dpkg -i minio.deb
```
#### Use the following commands to download the latest stable MinIO binary and install it to the system $PATH:
```sh
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```


https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-single-node-single-drive.html#minio-snsd


https://medium.com/yavar/setting-up-minio-object-storage-server-a-step-by-step-guide-for-ubuntu-20-04-5270528273f5