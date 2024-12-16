---
title: "Part 4"
weight: 20
---

## Topics
- Kubernetes Probes
- Kubernetes Storage 
     - Secret
     - ConfigMap
     - emptyDIr
     - hostPath
-  StatefulSet
-  Kubernetes IngressController 
- Kubectl Set
- Kubectl Patch

## Probes
The kubelet uses liveness probes to know when to restart a container.<br>
Liveness probes can be a powerful way to recover from application failures, but they should be used with caution.
### Liveness Probe
- It checks whether the application running inside the container is still alive and functioning properly.
### Readiness Probe:
It determines whether the application inside the container is ready to accept traffic or requests. When a readiness probe fails, Kubernetes stops sending traffic to the container until it passes the probe. This is useful during application startup or when the application needs some time to initialize before serving traffic.
### Startup Probe (introduced in Kubernetes 1.16):
It is similar to a readiness probe, but it only runs during the initial startup of a container. The purpose of the startup probe is to differentiate between a container that's starting up and one that's in a crashed state. Once the startup probe succeeds, it is disabled, and the readiness probe takes over.

-  Create a pod with startup probe
-  Make a test and monitor the pods and logs as well
-  Create a pod with liveness prode 
-  Create a pod with readiness probe 

Probes have a number of fields that you can use to more precisely control the behavior of startup, liveness and readiness checks:

- **initialDelaySeconds:** Number of seconds after the container has started before startup, liveness or readiness probes are initiated. Defaults to 0 seconds. Minimum value is 0.
- **periodSeconds:** How often (in seconds) to perform the probe. Default to 10 seconds. The minimum value is 1.
- **timeoutSeconds:** Number of seconds after which the probe times out. Defaults to 1 second. Minimum value is 1.
- **successThreshold:** Minimum consecutive successes for the probe to be considered successful after having failed. Defaults to 1. Must be 1 for liveness and startup Probes. Minimum value is 1.
- **failureThreshold:** After a probe fails failureThreshold times in a row, 
- **terminationGracePeriodSeconds:** configure a grace period for the kubelet to wait between triggering a shut down of the failed container, and then forcing the container runtime to stop that container. The default is to inherit the Pod-level value for terminationGracePeriodSeconds (30 seconds if not specified), and the minimum value is 1. See probe-level terminationGracePeriodSeconds for more detail.

### A pod with  Startup Probe
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-startup-probe
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    startupProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 20
      periodSeconds: 5
EOF
```
### A Pod with Liveness probe
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-liveness-probe
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
EOF
```

### A Pod with Readiness probe

```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-readiness-probe
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 5
EOF
```
### Key Points about kubernetes Probes
- initialDelaySeconds 60 seconds specifies the delay before the probe starts executing after the container starts.
- periodSeconds 10 seconds means that the probe will be executed every 10 seconds.
- failureThreshold 3 indicates that the container will be marked as unhealthy after three consecutive failed probes.
## Kubernetes Storage
- Set up NFS storage 
```t      
git clone https://gitlab.com/container-and-kubernetes/kubernetes-2024.git
cd kubernetes-2024/nfs-subdir-external-provisioner
```
- Run the script on all nodes
```bash
#!/usr/bin/env bash

export DEBIAN_FRONTEND=noninteractive

readonly NFS_SHARE="/srv/nfs/kubedata"

echo "[TASK 1] apt update"
sudo apt-get update -qq >/dev/null

if [[ $HOSTNAME == "kmaster" ]]; then
  echo "[TASK 2] install nfs server"
  sudo -E apt-get install -y -qq nfs-kernel-server >/dev/null
  echo "[TASK 3] creating nfs exports"
  sudo mkdir -p $NFS_SHARE
  sudo chown nobody:nogroup $NFS_SHARE
  echo "$NFS_SHARE *(rw,sync,no_subtree_check)" | sudo tee /etc/exports >/dev/null
  sudo systemctl restart nfs-kernel-server
else
  echo "[TASK 2] install nfs common"
  sudo -E apt-get install -y -qq nfs-common >/dev/null
fi
```
- Now apply nfs on kubernetes
```t
kubectl apply -f 01-setup-nfs-provisioner.yaml
```
-  Create a pvc using 02-test-claim.yaml
```t
kubectl apply -f 02-test-claim.yaml
```
- Check newly created pvc
```t
kubectl get pvc
```
#### Storage Classes
- A StorageClass provides a way for administrators to describe the classes of storage they offer.
-  Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators.
-  You can define default storageclass
- You can also decide whether the pvc can be expanded.

Example of storage class
```t
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "false"
```
- Check available storageClass
```t
 kubectl get sc
 ```

 - Enable Expansion of pvc
 ```t
 query='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
default_sc=$(kubectl get sc -o=jsonpath="$query")

echo patching storage class "[$default_sc]"

kubectl patch storageclass $default_sc -p '{"allowVolumeExpansion": true}'
```
#### A PersistentVolume

- A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes.
-  It is a resource in the cluster just like a node is a cluster resource.

#### A PersistentVolumeClaim (PVC)
- A PersistentVolumeClaim (PVC) is a request for storage by a user.
- It is similar to a Pod. Pods consume node resources and PVCs consume PV resources.


#### Lifecycle of a volume and claim

- PVs are resources in the cluster. 
- PVCs are requests for those resources and also act as claim checks to the resource. The interaction between PVs and PVCs follows this lifecycle:

#### Provisioning 
 There are two ways PVs may be provisioned: statically or dynamically.

##### Static
A cluster administrator creates a number of PVs. They carry the details of the real storage, which is available for use by cluster users. They exist in the Kubernetes API and are available for consumption.

##### Dynamic 
- When none of the static PVs the administrator created match a user's PersistentVolumeClaim, the cluster may try to dynamically provision a volume specially for the PVC.
- This provisioning is based on StorageClasses: the PVC must request a storage class
- The administrator must have created and configured that class for dynamic provisioning to occur

#### Access Modes
##### The access modes are:

##### ReadWriteOnce
the volume can be mounted as read-write by a single node. ReadWriteOnce access mode still can allow multiple pods to access the volume when the pods are running on the same node. For single pod access, please see ReadWriteOncePod.
##### ReadOnlyMany
the volume can be mounted as read-only by many nodes.
##### ReadWriteMany
the volume can be mounted as read-write by many nodes.
##### ReadWriteOncePod
the volume can be mounted as read-write by a single Pod. Use ReadWriteOncePod access mode if you want to ensure that only one pod across the whole cluster can read that PVC or write to it.

#### In the CLI, the access modes are abbreviated to:

- RWO - ReadWriteOnce
- ROX - ReadOnlyMany
- RWX - ReadWriteMany
- RWOP - ReadWriteOncePod

- **Example of a pvc**
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
EOF
```
- Create a pod and use the pvc there
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```
- Test pod and pvc
- Also check the storage class 
```t
kubectl get sc
```
### emptyDir
- emptyDir are volumes that get created empty when a Pod is created.
- While a Pod is running its emptyDir exists. If a container in a Pod crashes the emptyDir content is unaffected. Deleting a Pod deletes all its emptyDirs.
An emptyDir volume is first created when a Pod is assigned to a Node and initially its empty
- A Volume of type emptyDir that lasts for the life of the Pod, even if the Container terminates and restarts.
- If a container in a Pod crashes the emptyDir content is unaffected.
- All containers in a Pod share use of the emptyDir volume .
- Each container can independently mount the emptyDir at the same / or different path.
- Using emptyDir, The Kubelet will create the directory in the container, but not mount any storage.
- Containers in the Pod can all read/write the same files in the emptyDir volume, though that volume can be mounted at the same or different paths in each Container.
- When a Pod is removed from a node for any reason, the data in the emptyDir is deleted forever along with the container.
- A Container crashing does NOT remove a Pod from a node, so the data in an emptyDir volume is safe across Container crashes.
- By default, emptyDir volumes are stored on whatever medium is backing the node – that might be disk or SSD or network storage.
- You can set the emptyDir.medium field to “Memory” to tell Kubernetes to mount a tmpfs (RAM-backed filesystem) for you instead.
- The location should of emptyDir should be in /var/lib/kubelet/pods/{podid}/volumes/kubernetes.io~empty-dir/ on the given node where your pod is running.
- Example:
```t
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  - image: redis
    name: redis
    volumeMounts:
    - mountPath: /data
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```
### hostPath:
- The hostPath volume mounts a resource from the host node filesystem. the resources could be directory, file socket, character, or block device. These resources mu
- A hostPath volume mounts a file or directory from the host node’s filesystem into your pod.
- A hostPath PersistentVolume must be used only in a single-node cluster. Kubernetes does not support hostPath on a multi-node cluster currently.
- The directories created on the underlying hosts are only writable by root. You either need to run your process as root in a privileged container or modify the file permissions on the host to be able to write to a hostPath volume
- Example:
```t
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: DirectoryOrCreate
  ```
### PersistentVolumeClaim
https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/
## Kubernetes Objects
### Secret
-  Creating a secret 
-  Checking the value of a secret
-  Use jsonpath to fetch the value
-  Use secret in pod an environment value
-  Use secret as a file in pod 
-  Creating a secret of type tls


### Configmap
-  Create a config map
-  use cm as a file in pod
#### Secret
-  To create a secret, you can use the `kubectl create secret` command.

```bash
kubectl create secret generic my-secret --from-literal=MYSQL_ROOT_PASSWORD=root 
```
- To create secret of type TLS
    
 ```bash
   kubectl create secret tls my-tls-secret --cert=path/to/tls.crt --key=path/to/tls.key

  ```
 -  Check secret
  ```bash
     kubectl get secrets
  ```
- use the secret with pod as an environment variable
```bash
apiVersion: v1
kind: Pod
metadata:
 name: mypod
spec:
 containers:
 - name: mycontainer
   image: mysql
   env:
   - name: MYSQL_ROOT_PASSWORD
     valueFrom:
      secretKeyRef:
        name: my-secret
        key: MYSQL_ROOT_PASSWORD   
```   
-  Use secret as a file
```bash
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: myimage
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
```
-  How to extract the value from a secret 
```bash
kubectl get secret my-secret -o jsonpath='{.data.username}' | base64 --decode > username.txt
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 --decode > password.txt
```
#### Configmap
- Create a configmap
```t
kubectl create configmap my-configmap --from-literal=key1=value1
```
- Create cm with yaml
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  key1: value1
  key2: value2
```
- Configmap for a config file
```t
kubectl create configmap my-configmap --from-file=config-file.txt
```

-  use configmap as a file in a pod
```bash
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: myimage
    envFrom:
    - configMapRef:
        name: my-configmap
---
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: myimage
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: my-configmap
```

## IngressController

#### Documentation Referred:

https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

https://kubernetes.github.io/ingress-nginx/deploy/


- Step 1: Install Nginx Ingress Controller:
```sh
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace
```

-  Step 2: Verify the Ingress Controller Resource
```sh
helm list --all-namespaces
kubectl get ingressclass
kubectl get service -n ingress-nginx
```
- check if the IngressClass is available
```t
kubectl get ingressClassName
```
- Add the Application dns to your hosts in case you are not using DNS service 
  ```sh
  nano /etc/hosts   for windows it is under C:\Windows\System32\drivers\etc
  ```
  ```sh
  <ADD-LOAD-BALANCER-IP> website01.example.internal website02.example.internal
  ```
- Step 1: Create two Pods 
  ```sh
  kubectl run service-pod-1 --image=nginx
  kubectl run service-pod-2 --image=nginx
  ```

- Step 2: Create Service for above created pods
  ```sh
  kubectl expose pod service-pod-1 --name service1 --port=80 --target-port=80
  kubectl expose pod service-pod-2 --name service2 --port=80 --target-port=80
  kubectl get services
  ```

- Step 3: Verify Service to POD connectivity
  ```sh
  kubectl run frontend-pod --image=ubuntu --command -- sleep 36000
  kubectl exec -it frontend-pod -- bash
  apt-get update && apt-get -y install curl nano
  curl <SERVER-1-IP>
  curl <SERVER-1-IP>
  ```
-  Check if the application is reachable
    ```sh
    curl website01.example.internal
    curl website02.example.internal
    ```

- Step 5: Change the Default Nginx Page for Each Service

  ```sh
  kubectl exec -it service-pod-1 -- bash
  cd /usr/share/nginx/html
  echo "This is Website 1" > index.html
  ```
  ```sh
  kubectl exec -it service-pod-2 -- bash
  cd /usr/share/nginx/html
  echo "This is Website 2" > index.html
  ```

- Step 6: Verification

  ```sh
  kubectl exec -it frontend-pod -- bash
  curl website01.example.internal
  curl website02.example.internal
  ```

#### Documentation Referred:
[ Lean About name based virtual Hosting](https://kubernetes.io/docs/concepts/services-networking/ingress/#name-based-virtual-hosting)

- Step 7: Create Ingress Resource
  ```sh
  kubectl apply -f - <<EOF
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: name-virtual-host-ingress
  spec:
    ingressClassName: nginx
    rules:
    - host: website01.example.internal
      http:
        paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: service1
              port:
                number: 80
    - host: website02.example.internal
      http:
        paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: service2
              port:
                number: 80
  EOF
  ```
 
- Check newlly created ingress rules
   ```t
   kubectl get ingress
   ```
- Check more information for ingress
    ```t
    kubectl describe ingress name-virtual-host-ingress created
    ```
- Now check if the application is opening in browser 


## Kubernetes Patch command

### Description

Briefly describe the purpose of the issue.

### Examples using `kubectl patch`
- Patch a Deployment (Update Replicas)

```bash
  kubectl patch deployment <deployment-name> -p '{"spec": {"replicas": 3}}'
 ```
- Patch a Pod (Update Container Image)
```t
kubectl patch pod <pod-name> -p '{"spec": {"containers": [{"name": "container-name", "image": "new-image:tag"}]}}'
```
- Update configmap
```bash
kubectl patch configmap <configmap-name> -p '{"data": {"new-key": "new-value"}}'
```
 - Patch a Service (Update Service Type)
```bash
kubectl patch service <service-name> -p '{"spec": {"type": "LoadBalancer"}}'
```
- Patch a PVC (Update Storage Size)
```t
kubectl patch pvc <pvc-name> -p '{"spec": {"resources": {"requests": {"storage": "5Gi"}}}}'
```

### StatefulSets
- StatefulSet is the workload API object used to manage stateful applications.
- Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec.
- Unlike a Deployment, a StatefulSet maintains a sticky identity for each of its Pods.
#### Using StatefulSets
##### StatefulSets are valuable for applications that require one or more of the following.

- Stable, unique network identifiers.
- Stable, persistent storage.
- Ordered, graceful deployment and scaling.
- Ordered, automated rolling updates.

#### Example of statefulset
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
  ---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  minReadySeconds: 10 # by default is 0
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class" <Change this as per your sc>
      resources:
        requests:
          storage: 1Gi
  EOF
  ```
- check statefulset
```t
kubectl  get sts
```
- Also check for service type 
```t
kubectl get svc
```
- Check for pvc
```t
kubectl get pvc
```
- Check pods
```t
kubectl get pods
```
- Test after deleting a pod if the pod is taking the same name again
