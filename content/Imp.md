---
title: "Imp Commands "
weight: 21
no_list: true
---
## Topics

### How to increase disk size if the vm is created with vagrant 
- First set the environment variable 
- **C:\Program Files\Oracle\VirtualBox** this is default path for VBoxManage command
- Run the below command
```sh
VBoxManage modifymedium disk "E:\VirtualBox VMs\Prometheus_default_1725120844484_62520\packer-rocky-84-1624559295-disk001.vmdk" --resize 100000
```
- Verify the size 
```sh
VBoxManage showmediuminfo "E:\VirtualBox VMs\Prometheus_default_1725120844484_62520\packer-rocky-84-1624559295-disk001.vmdk"

```