---
title: "Part 3"
weight: 20
---
## Topics
- [Service](#service)
  - CLusterIP
  - NodePort
  - LoadBalancer
  - Headless Service
- [METALLB](#deploying-metallb)
- [DNS Troubleshooting](#dns-troubleshooting)
-  [Kubernetes IngressController](#ingresscontroller) 
- [ReplicaSet](#replicaset)
- [Deployment](#-deployment)
- [Daemonset](#daemonset)
- Type of [Selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
    - Equality based
    - Set based selector
- Job 
- CronJob
- [Metric Server](#metric-server)
- HPA(pod Autoscaling)
- [Helm Overview](./docs/miscellaneous/Helm.md)
 
##  Service
A Service is a method for exposing a network application that is running as one or more Pods in your cluster.
- A key aim of Services in Kubernetes is that you don't need to modify your existing application to use an unfamiliar service discovery mechanism.
- A Kubernetes service is a logical abstraction for a deployed group of pods in a cluster
### What are the types of Kubernetes services?
- **ClusterIP**: Exposes a service which is only accessible from within the cluster.
- **NodePort**: Exposes a service via a static port on each node’s IP. NodePorts are in the 30000-32767 range by default
- **LoadBalancer**: Exposes the service via the cloud provider’s load balancer.
- **ExternalName** Maps a service to a predefined externalName field by returning a value for the CNAME record.
#### Create a service of Type(ClusterIp)
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Service
  metadata:
    name: myapp-service
  spec:
    selector:
      app: myapp
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
EOF
```
####  Create a service of Type (NodePort)
```t
kubectl expose pod/<name> --port 80 --target-port=80 --name <Svc>
```
####  Create a type of service (LoadBalancer)
```t
kubectl expose pod/name --port=80 --type=LoadBalancer
```
- Check the endpoint 
```bash
kubectl get ep
```
- Achieve the blue green
- Delete a service 
- Exchange the service selector

##  Metallb A LoadBalancer Solution for OnPrem Kubernetes
## Deploying [Metallb](https://metallb.universe.tf/)
### Steps:
```t
git clone https://gitlab.com/container-and-kubernetes/kubernetes-2024.git
cd kubernetes-2024/
cd metallb
kubectl apply -f 01 01_metallb.yaml
kubectl apply -f 02_metallb-config.yaml

```
- Make a test after creating a service 

```bash
kubectl apply -f 03_test-load-balancer.yaml
```
- Check the service 
```bash
kubectl get svc
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

### Metric Server
[Metric Server Github Link](https://github.com/kubernetes-sigs/metrics-server)

### Steps
```bash
git clone https://gitlab.com/container-and-kubernetes/kubernetes-2024.git
cd kubernetes-2024
cd  metricserver
kubectl apply -f .
```
- wait for few mins and check if the metric server pods are up
```t
kubectl top nodes
kubectl top pods
```
## HPA (Pod Autoscaling)
### Steps
### Prerequisites
 - Resources should be defined for a pod
 - Metric Api has to be available .
```bash
git clone https://gitlab.com/container-and-kubernetes/kubernetes-2024.git
cd kubernetes-2024
cd  hpa
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply hpa.yaml
```

- check hpa
```bash
kubectl get hpa
```
## ReplicaSet

- ✅ Create a ReplicaSet  
- 📏 Scale a ReplicaSet  
- 🔍 Check the image used by a ReplicaSet  
- 🔄 Update the image for a ReplicaSet  
- 🌐 Create a Service to expose the ReplicaSet  
- 🔁 Forward a port from the ReplicaSet Pod to local machine  
- 🧪 Test if the application is accessible  

### Replicaset Overview
A ReplicaSet is used to maintain a stable set of replica Pods running at any given time. If a Pod goes down or is deleted, the ReplicaSet will automatically create a new one to replace it.
## 🔑 Key Features

- ✅ Ensures **high availability** by maintaining the desired count of Pods.
- 🔁 Provides **self-healing** by recreating missing Pods.
- 🔢 Supports **scaling**: You can increase or decrease the number of replicas.
- 🎯 Uses **label selectors** (`matchLabels`) to manage the right set of Pods.
## ⚖️ ReplicaSet vs Deployment

| Feature        | ReplicaSet | Deployment |
|----------------|------------|------------|
| Pod Management | ✔️         | ✔️         |
| Rolling Update | ❌         | ✔️         |
| Rollback       | ❌         | ✔️         |
| Recommended    | ❌ (low-level) | ✔️ (preferred for most cases) |
### ReplicasSet Yaml example
```sh
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: nginx
EOF
```
-  View Replicaset

```t
kubectl get replicasets
kubectl get rs
```
-  Scale up/down a replicaset
 ```t
kubectl scale replicaset my-replicaset --replicas=5
```
-  Create a service for Replicaset
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
EOF
```
-  configure port forward for replicaset
```bash
kubectl port-forward replicaset/my-replicaset 8080:80
```
## 🚀 Deployment

- 🆕 **Create a Deployment**  
- 🔐 **Update Secrets Used by a Deployment**  
- ⚙️ **Configure Resource Requests and Limits in a Deployment**  
- 🌱 **Set Environment Variables in a Deployment**  
- 📦 **Perform a Rollout of a Deployment**  
- 🔁 **Roll Back to the Previous Revision**  
- ⬅️ **Roll Back to a Specific Revision**  
- 📊 **Check the Maximum Number of ReplicaSets Retained for a Deployment**  
- 🛠️ **Understand and Configure Deployment Strategies**  

### 🚀  Deployment Overview

- A Deployment provides declarative updates for Pods and ReplicaSets.
- You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate.
- You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.
-  **Create a deployment**
```bash
kubectl create deployment myapp-deployment --image=nginx
```
- Deployment with Yaml file
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3  # Number of desired replicas
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
```
-  Perform Rollout
```bash
kubectl set image deployment/myapp-deployment nginx-container=nginx:1.21.4
```
- Check rollout status 
```bash
kubectl rollout status deployment myapp-deployment
```
-  Check the rollout history
```bash
kubectl rollout history deployment myapp-deployment
```
- Perform Rollback
```t
kubectl rollout undo deployment myapp-deployment
```
-  Perform scale up/down
```bash
kubectl scale deployment  myapp-deployment --replicas=5
```
- Set Environment Variables in Deployment
```bash
kubectl set env deployment/myapp-deployment KEY=VALUE
```
-  Set Resources
```t
kubectl set resources deployment/myapp-deployment --limits=cpu=200m,memory=512Mi --requests=cpu=100m,memory=256Mi
```
- Update SA for a deployment 
```t
kubectl set serviceaccount deployment/myapp-deployment my-service-account

```
- Change the revision history 
```yaml
spec:
  revisionHistoryLimit: 20
  replicas: 3
  selector:
    matchLabels:
      app: your-app
  template:
    metadata:
      labels:
        app: your-app
    spec:
      containers:
      - name: your-container
        image: your-image

```
-  How to check if the change has been accepted
```bash

kubectl get deployment <deployment-name> -o=jsonpath='{.spec.revisionHistoryLimit}
```
-  How to use record option with rollback depoloyment 
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.21
kubectl annotate deployment nginx-deployment kubernetes.io/change-cause="Updated to nginx 1.21"

```
## job

- Create a job
- Create a cronjob 
- Clean up finished jobs automatically
    -  ttlSecondsAfterFinished: 100 
- check different option in cronjob as job 
     -  backoffLimit
        ```t
        Max pod restart in case of failure
        ```
     -  completions
        ```t
         How many containers of the job are created one after another
         ```
     -  parallelism: 
        ```t
        it defines how many pods will be created at once 
        ```

### Job Creation yaml
```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: busybox
        command: ["/bin/echo", "Hello World"]
      restartPolicy: Never
  backoffLimit: 4

EOF
```
### cronjob
- Create a cronjob
```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: simple-cronjob
spec:
  schedule: "*/1 * * * *"  # Run every minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: simple-cronjob-container
            image: busy
EOF
```
- Example 2 
```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: simple-cronjob
spec:
  schedule: "*/1 * * * *"  # Run every minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: simple-cronjob-container
            image: busybox
            command: ["echo", "Hello, Kubernetes!"]
EOF
```
- Example 3
```sh
kubectl apply -f - <<EOF
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: parallel-cronjob
spec:
  schedule: "0 */6 * * *"  # Run every 6 hours
  jobTemplate:
    spec:
      completions: 2
      parallelism: 1
      template:
        spec:
          containers:
          - name: parallel-cronjob-container
            image: busybox
            command: ["echo", "Running parallel cronjob"]
EOF
```
- Example 4
```sh
kubectl apply -f - <<EOF
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob-volume-mounts
spec:
  schedule: "*/5 * * * *"  # Run every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cronjob-volume-mounts-container
            image: busybox
            command: ["/bin/sh", "-c"]
            args: ["echo Hello > /data/hello.txt"]
            volumeMounts:
            - name: data-volume
              mountPath: /data
          volumes:
          - name: data-volume
            emptyDir: {}
EOF
```
-  Cronjob/job with imperative way
```bash
kubectl create job myjob --image=busybox --command -- echo "Hello, Kubernetes!"
kubectl create cronjob mycronjob --image=busybox --schedule="*/5 * * * *" --command -- echo "Scheduled Job"

```
- Suspend an active Job:
```t
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":true}}'
```
- Resume a suspended Job:
```t
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":false}}'
```


## [Daemonset](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.

- Some typical uses of a DaemonSet are:
   - running a cluster storage daemon on every node
   - running a logs collection daemon on every node
   -  running a node monitoring daemon on every node
 -  Create a daemonset
 ```sh
 kubectl apply -f - <<EOF
 apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      # it may be desirable to set a high priority class to ensure that a DaemonSet Pod
      # preempts running Pods
      # priorityClassName: important
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
EOF
```
 -  Check the daemonset 
 ```t
 kubectl get ds
 ```
 - test if the daemonset is as expected.
 ```t
 kubectl get pods -n kube-system -o wide |grep -i fluentd-elasticsearch
 ```
 -  check the pod if they are creating on all nodes

 ### DNS Troubleshooting:
 [ Follow this for DNS Troubleshooting](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- Create a simple Pod to use as a test environment
  ```sh
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: dnsutils
    namespace: default
  spec:
    containers:
    - name: dnsutils
      image: registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3
      command:
        - sleep
        - "infinity"
      imagePullPolicy: IfNotPresent
    restartPolicy: Always
  EOF
  ```
- Run nslookup for kuberntes cluster
 ```t
 kubectl exec -i -t dnsutils -- nslookup kubernetes.default
 ```
 - Check the local DNS configuration first
 ```t
 kubectl exec -ti dnsutils -- cat /etc/resolv.conf
 ```
 - if you get some error  Check if the DNS pod is running
  ```t
  kubectl get pods --namespace=kube-system -l k8s-app=kube-dns
  ```
- Check for errors in the DNS pod
 ```t
  kubectl logs --namespace=kube-system -l k8s-app=kube-dns
```
- Is DNS service up?
```t
kubectl get svc --namespace=kube-system
```
- Are DNS endpoints exposed?
 ```t
 kubectl get endpoints kube-dns --namespace=kube-system
 ```
### DNS for Services and Pods
- Kubernetes creates DNS records for Services and Pods.
- You can contact Services with consistent DNS names instead of IP addresses.
- Services defined in the cluster are assigned DNS names. 
- By default, a client Pod's DNS search list includes the Pod's own namespace and the cluster's default domain.

