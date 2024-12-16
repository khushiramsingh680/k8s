---
title: "Container Basics"
weight: 21
no_list: true
---
## Topics

-  Create a virtual machine 
```bash
git clone https://gitlab.com/container-and-kubernetes/kubernetes-2024.git
cd kubernetes-2024/
vagrant up

```
- You will have a vm ready within 5 mins of time 
-  Login to Linux vm
```bash
ssh vagrant@172.16.16.105
Enter password: vagrant
```
-  Switch from vagrant user of root user
```bash
 sudo -s
```

## Docker Basics Lab
-  How to install docker 
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
- Run the below command

    ```t
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
- Check docker version
    ```t         
    docker version
    ```
- check all docker images
    ```t
    docker images
    ```
- check all running containers
    ```t
    docker ps 
    ```
- Check all containers
    ```t
    docker ps -a
    ```
- Download an image 
    ```t
    docker pull <imagename>
    docker pull nginx
    ```
- How to create a container
    ```t
    docker run container id  (this will run a process in foreground)
    ```
 - Run a container in backgroud
    ```t
    docker run -d container id
    ```
- Login to a container
    ```t
    docker exec container id sh/bash
    ```
- Delete a container
    ```t
    docker rm containerid
    ```
- Delete container forcefully(if the container is running)
    ```t
    docker rm containerid --force
    ```
- Check the id of all containers
    ```t
    docker ps -aq
    ```
- Delete all containers
    ```t
    docker rm $(docker ps -aq) -f
    ```
- Delete all images
    ```t
    docker rmi $(docker images -aq)
    ```
### Docker Networking 

-  Expose a docker container port to access from host ip
    ```t
    docker run -d -p <hostport>:<containerport> <image name>

    ```
#### NOTE: You can not map a host port which is already in use

### Docker storage

-  How to map a  directory to container dir
    ```t
    docker run -d -v <dir at host>:<dir at container> <imagename>
    ```


### Docker Images

- How to login to docker hub
    ```t
    docker login
    ```
- How to convert a running container into an image
    ```t
    docker commit containerid newname
    ```
- How to save docker image into a tar
    ```t
    docker save <imagename>  > example.tar
    ```

- How to import from a tar file
    ```t
    docker load < example.tar
    ```
- How to rename an image
    ```t
    docker tag <oldname> <newname>
    ```

- Login to docker hub/any other container registry
    ```bash
    docker login 
    docker login <registry url if any>
    ```
- Tag the image before you push.
   ```bash
    docker tag <old image name> <new image name >
    ```
-  How to push image to dockerhub
    ```t
    docker push imagename
    ```

### Creating dockerimage from Docker file
[Dockerfile Documentation](https://docs.docker.com/engine/reference/builder/)

-  Destroy the vm once the Lab is completed from the directory of your Laptop where you have vagrantfile
    ```bash
    vagrant destroy 
    ```
## Upcoming
### Docker Capabilities
### Docker Compose
### Dockerfile


Docker containers are very similar to LXC containers, and they have similar security features.
When you start a container with docker run, behind the scenes Docker creates a set of namespaces and control groups for the container.

Docker makes heavy use of Linux kernel features. One of the fundamental aspects that containers make use of from Linux Kernel are namespaces and cgroups.

## Namespace
Namespaces provide the first and most straightforward form of isolation. Processes running within a container cannot see, and even less affect, processes running in another container, or in the host system.
Namespaces are a feature of the Linux kernel that partitions kernel resources such that one set of processes sees one set of resources and another set of processes sees a different set of resources.

### Types of Namespaces

Within the Linux kernel, there are different types of namespaces. Each namespace has its own unique properties:

- A **user** namespace has its own set of user IDs and group IDs for assignment to processes. In particular, this means that a process can have root privilege within its user namespace without having it in other user namespaces.
- A **process ID** (PID) namespace assigns a set of PIDs to processes that are independent from the set of PIDs in other namespaces. The first process created in a new namespace has PID 1 and child processes are assigned subsequent PIDs. If a child process is created with its own PID namespace, it has PID 1 in that namespace as well as its PID in the parent process’ namespace. See below for an example.
- A **network** namespace has an independent network stack: its own private routing table, set of IP addresses, socket listing, connection tracking table, firewall, and other network‑related resources.
- A **mount** namespace has an independent list of mount points seen by the processes in the namespace. This means that you can mount and unmount filesystems in a mount namespace without affecting the host filesystem.
- An interprocess communication (IPC) namespace has its own IPC resources, for example POSIX message queues.
A **UNIX Time‑Sharing** (UTS) namespace allows a single system to appear to have different host and domain names to different processes.
- Create two container and check if the they are sharing the namespaces defined above 
 ```t
  docker run -d --name c1 ubuntu sleep 1d
  docker run -d --name c2 ubuntu sleep 2d
  ```
  - Login to container and check 
  ```t 
  docker exec c2 ps aux
  ```
  - Now check on host vm
  ```t
  ps -aux
   ```
  - Now create a container with the namespace 
  ```t
  docker run -d c3 --pid=container:c1 ununtu sleep 3d
  ```
  - Now check if the pid is shared for c1 container and c3 container
  
  ### What Are cgroups?
  A control group (cgroup) is a Linux kernel feature that limits, accounts for, and isolates the resource usage (CPU, memory, disk I/O, network, and so on) of a collection of processes.

   Cgroups provide the following features:
 
- **Resource limits** – You can configure a cgroup to limit how much of a particular resource (memory or CPU, for example) a process can use.
- **Prioritization** – You can control how much of a resource (CPU, disk, or network) a process can use compared to processes in another cgroup when there is resource contention.
- **Accounting** – Resource limits are monitored and reported at the cgroup level.
- **Control** – You can change the status (frozen, stopped, or restarted) of all processes in a cgroup with a single command.
- Check the resouce stats
  ```t
  docker stats
  ```


### Introduction to capabilities

The Linux kernel is able to break down the privileges of the root user into distinct units referred to as capabilities. For example, the CAP_CHOWN capability is what allows the root use to make arbitrary changes to file UIDs and GIDs. The CAP_DAC_OVERRIDE capability allows the root user to bypass kernel permission checks on file read, write and execute operations. Almost all of the special powers associated with the Linux root user are broken down into individual capabilities.

This breaking down of root privileges into granular capabilities allows you to:

Remove individual capabilities from the root user account, making it less powerful/dangerous.
Add privileges to non-root users at a very granular level.

Capabilities apply to both files and threads. File capabilities allow users to execute programs with higher privileges. This is similar to the way the setuid bit works. Thread capabilities keep track of the current state of capabilities in running programs.

Capabilities | Definition
-------------|-----------
CHOWN        | Make arbitrary changes to file UIDs and GIDs
DAC_OVERRIDE | Discretionary access control (DAC) - Bypass file read, write, and execute permission checks
FSETID       | Don’t clear set-user-ID and set-group-ID mode bits when a file is modified; set the set-group-ID bit for a file whose GID does not match the file system or any of the supplementary GIDs of the calling process.
FOWNER       | Bypass permission checks on operations that normally require the file system UID of the process to match the UID of the file, excluding those operations covered by CAP_DAC_OVERRIDE and CAP_DAC_READ_SEARCH.
MKNOD        | MKNOD - Create special files using mknod(2)
NET_RAW      | Use RAW and PACKET sockets; bind to any address for transparent proxying.
SETGID       | Make arbitrary manipulations of process GIDs and supplementary GID list; forge GID when passing socket credentials via UNIX domain sockets; write a group ID mapping in a user namespace.
SETUID       | Make arbitrary manipulations of process UIDs; forge UID when passing socket credentials via UNIX domain sockets; write a user ID mapping in a user namespace.
SETFCAP      | Set file capabilities.
SETPCAP      | If file capabilities are not supported: grant or remove any capability in the caller’s permitted capability set to or from any other process.
NET_BIND_SERVICE | Bind a socket to Internet domain privileged ports (port numbers less than 1024).
SYS_CHROOT   | Use chroot(2) to change to a different root directory.
KILL         | Bypass permission checks for sending signals. This includes use of the ioctl(2) KDSIGACCEPT operation.
AUDIT_WRITE  | Write records to kernel auditing log.






- To drop capabilities from the root account of a container.
```sh
sudo docker run --rm -it --cap-drop $CAP alpine sh
```
 - To add capabilities to the root account of a container.
```sh
sudo docker run --rm -it --cap-add $CAP alpine sh
```
- To drop all capabilities and then explicitly add individual capabilities to the root account of a container.
```sh
sudo docker run --rm -it --cap-drop ALL --cap-add $CAP alpine sh
```
### Testing Docker capabilities
```t
docker container run --rm -it alpine chown nobody /
docker container run --rm -it --cap-drop ALL --cap-add CHOWN alpine chown nobody /
docker container run --rm -it --cap-drop CHOWN alpine chown nobody /
docker container run --rm -it --cap-add chown -u nobody alpine chown nobody /
ocker run --rm -it --cap-drop=NET_RAW alpine sh
ping 127.0.0.1 -c 2
```


- check all capabilities
```t
capsh --print
```
```sh
docker run --rm -it --privileged=true 71aa5f3f90dc bash

capsh --print
```





### Add Capabiliies for a pod
```sh

apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]

```
-  Selinux 
  ```t
  securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"

```


## Topics 
# Docker Course TOC

## Day 1: Introduction to Containerization
- Containerization
- History of Containers
- Namespaces and Cgroups
- Containers vs Virtual Machines
- Types of Containers
- Introduction to Docker
- Docker Architecture
- Container Lifecycle
- Docker CE vs Docker EE

## Day 2: The Docker Engine
- Docker Engine
- Configuring Logging Drivers
- Docker Terminology
- Port Binding
- Detached vs Foreground Mode
- Docker CLI
- Docker Exec
- Restart Policy
### Hands-On
- Setting up Docker Engine
- Upgrading Docker Engine
- Setting up logging drivers in Docker
- Port Binding
- Starting Containers in different modes
- Docker CLI Commands
- Docker Exec Commands
- Restart Policy in Docker
- Removing Containers

## Day 3: Image Management and Registry
- Dockerfile
- Dockerfile Instructions
- Build Context
- Docker Image
- Docker Registry
- Write a Dockerfile to create an Image
- Docker Image Tags
- Setting up Docker Hub
- Configuring Local Registry
- Removing Images from the Registry

## Day 4: Storage in Docker
- Docker Storage
- Types of Persistent Storage
- Volumes
- Bind Mounts
- tmpfs Mount
- Storage Drivers
- Device Mapper
- Docker Clean Up
### Hands-On
- Deploy Docker Volumes
- Deploy Bind Mounts
- Use tmpfs mounts
- Configure Device Mapper
- Docker Clean Up

## Day 5: Orchestration in Docker
- Docker Compose
- Docker Swarm
- Docker Service
- Service Placement
- Rolling Update and Rollback
- Docker Stack
### Hands-On
- Deploy a Multi-container Application using Compose
- Running Docker in Swarm mode
- Deploying a Service in Swarm
- Scale Services
- Service Placement
- Rolling Updates and Rollbacks
- Docker Stack

## Day 6: Networking and Security
- Docker Networking
- Network Drivers
- Bridge Network
- Overlay Network
- Host and Macvlan
- Docker Security
- Docker Content Trust
- Securing the Docker Daemon
### Hands-On
- Create and use a User-defined Bridge Network
- Create and use a Overlay Network
- Use Host and Macvlan Network
- Configure Docker to use External DNS
- Signing images using DCT
- Securing the Docker Daemon

## Day 7: Docker EE and Monitoring
- Docker Enterprise
- Universal Control Plane (UCP)
- UCP Architecture
- Access Control in UCP
- Docker Trusted Registry (DTR)
- Monitoring using Prometheus
- Set up Docker Enterprise Edition
