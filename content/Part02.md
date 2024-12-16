---
title: "Part 2"
weight: 20
---


## Topics:
### [NameSpace](#namespace)
- Create a namespace
- Switch from one ns to another 
- Resource Quota
- Resource Limits 
 ### [POD](#pod)
-    Single Container pod
-    MultiContainer pod
-    Login to a pod 
-    Copy to a pod and from a pod(container)
-    How to check logs from a container
-   Environment Variables of pod
-   initContainer
-    Command and Argument of a pod
-    pullSecret of a pod
-    pod restart policy
-    imagePull policy
-    How to delete a pod
-    Pod priority
-    Pod Resources
-    Pod Quality of Services(QOS)

### [Advance Scheduling of pod](#advance-scheduling-of-pod)
-  Scheduler
-  Nodename
-  nodeSelector
-  Node Affinity
-  Taints and Tolerations
- Pod affinity and anti affinity
-  Priority and PriorityClass
-  Preemption
- Disruption Budget
-  Topology and Constraints
-  Descheduler
#### Namespace
- Namespaces are a way to divide cluster resources between multiple users (via resource quota).
- namespaces provides a mechanism for isolating groups of resources within a single cluster
- Names of resources need to be unique within a namespace, but not across namespaces. 
-  To achieve multitenancy with Networkpolicy
-  To configure RBAC
-  Check all namespaces
```t
kubectl get ns
```
- Create a new namespace
```t
kubectl create ns <name>
```
-  Switch from one ns to another
```t
kubens
kubens <ns name>
```
#### To see which Kubernetes resources are and aren't in a namespace:
- In a namespace
```t
kubectl api-resources --namespaced=true
```
- Not in a namespace
```t
kubectl api-resources --namespaced=false
```
- [Learn About Resource Quota and limits ](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)

- Configure Memory and CPU Quotas for a Namespace
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
EOF
```
#### As per above Example:
- The ResourceQuota places these requirements on the quota-mem-cpu-example namespace:

- For every Pod in the namespace, each container must have a memory request, memory limit, cpu request, and cpu limit.
- The memory request total for all Pods in that namespace must not exceed 1 GiB.
- The memory limit total for all Pods in that namespace must not exceed 2 GiB.
- The CPU request total for all Pods in that namespace must not exceed 1 cpu.
- The CPU limit total for all Pods in that namespace must not exceed 2 cpu.

- **Check applied quota**
```t
kubectl get quota 
```
- **Create namespace using a yaml file**

```sh
kubectl apply -f - <<EOF
kind: Namespace
apiVersion: v1
metadata:
  name: test
  labels:
    name: test
EOF
```
#### Resource Limits
A Kubernetes cluster can be divided into namespaces. Once you have a namespace that has a default memory limit, and you then try to create a Pod with a container that does not specify its own memory limit, then the control plane assigns the default memory limit to that container.
-  Create a LimitRange
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
EOF
```
- Now create one pod with no resource and check if the default resource has been applied
```t
kubectl describe pod <podname>
```
#### POD
- A pod is the smallest deployable unit in Kubernetes that represents a single instance of an application.
- For example, if you want to run the Nginx application, you run it in a pod.
- A container is a single unit. However, a pod can contain more than one container. You can think of pods as a box that can hold one or more containers together.
- the pod gets a single unique IP address and containers running inside the pod use localhost to connect to each other on different ports.
- Each pod gets a unique IP address.
- Pods communicate with each other using the IP address.
- Containers inside a pod connect using localhost on different ports.
- Containers running inside a pod should have different port numbers to avoid port clashes.
- You can set CPU and memory resources for each container running inside the pod.
- Containers inside a pod share the same volume mount.
- All the containers inside a pod are scheduled on the same node; It cannot span multiple nodes.
  - If there is more than one container, during the pod startup all the main containers start in parallel. Whereas the init containers inside the pod run in sequence.
  - Create a pod
  ```t
  kubectl run pod --image=nginx
  ```
  - Check newly created pod
  ```t
  kubectl get pods
  ```
  - Check more info of a pod like where the pod is scheduled and ip address
  ```t
  kubectl get pods -o wide
  ```
  - check pods details like events and resources 
  ```t
  kubectl describe pod <podname>
  ```
  - Check the name of all pods
  ```t
  kubectl get pods -o name
  ```
- Create a pod using yaml file
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  labels:
    app: example
spec:
  containers:
  - name: my-container
    image: nginx:latest
EOF
```
  
  - Check Pod Logs
    ```t
    kubectl logs <pod-name>
    ```
  - Follow Pod Logs in Real-time
  ```t
    kubectl logs -f <pod-name>

  ```
  - How to check logs for a container from multiContainer pod
  ```t
  kubectl logs -c <containername> <podname>
  ```
  - Check the logs for all containers
  ```sh
  kubectl logs <podname> --all-containers=true
  - Execute Command in a Pod:
  ```t
  kubectl exec -it <pod-name> -- <command>

  ```
  -  Copy Files to/from Pod:
  ```t
  kubectl cp <local-path> <pod-name>:<pod-path>
  kubectl cp <pod-name>:<pod-path> <local-path>

  ```

  -  Delete a Pod:
  ```t
  kubectl delete pod <pod-name>

  ```
  - How to delete a pod forcefully
  ```t
  kubectl delete pod --force --grace-period=0
  ```
  -  Port Forwarding to Pod:
  ```t
  kubectl port-forward <pod-name> <local-port>:<pod-port>

  ```
  - Port forwarding on ip address not on the localhost
  ```t
   kubectl port-forward --address 0.0.0.0 pod/mypod 8888:5000
   ```
  - Get YAML Definition of a Running Pod:
  ```t
  kubectl get pod <pod-name> -o yaml
  ```
### Some useful command in realtime(Production)

  - Find out all the images of all the pods
  ```t
  kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}'
  ```
  - Get  All containers name
  ```t
  kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace} {.metadata.name}: {.spec.containers[*].name}{"\n"}{end}'
  ```
### Define Environment Variables for a Container
A Kubernetes environment variable is a dynamic value that configures some aspect of the environment in which a Kubernetes-based application runs.
  ```t
  env: 
      - name: SERVICE_PORT 
          value: "8080" 
      - name: SERVICE_IP 
          value: "192.168.100.1"
  ```
#### Problem Statement if the pod is failing becuase of environment variable
- Deploy one mysql pod and see if the pod is  failing 
```t
kubectl run mysql --image=mysql:5.6
```
-  Check the logs and fix the issue 
#### Environment Variables
- Apply required variables using below yaml file
```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-env
spec:
  containers:
  - name: my-container
    image: mysql:5.6
    env:
      - name:  MYSQL_ROOT_PASSWORD
        value: ""
EOF
``` 
## Command and Arguments with Kubernetes Pod
Here's a table summarizing the field names used by Docker and Kubernetes:

| Description                                     | Docker field name | Kubernetes field name |
|-------------------------------------------------|-------------------|------------------------|
| The command run by the container                | Entrypoint        | command                |
| The arguments passed to the command             | Cmd               | args   
 
 When you override the default Entrypoint and Cmd, these rules apply:
- If you do not supply command or args for a Container, the defaults defined in the Docker image are used.
- If you supply a command but no args for a Container, only the supplied command is used. The default EntryPoint and the default Cmd defined in the Docker image are ignored.
- If you supply only args for a Container, the default Entrypoint defined in the Docker image is run with the args that you supplied.
- If you supply a command and args, the default Entrypoint and the default Cmd defined in the Docker image are ignored. Your command is run with your args.
### Example 1: Command Override
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-command
spec:
  containers:
  - name: my-container
    image: nginx:latest
    command: ["echo"]
    args: ["Hello, Kubernetes!"]
EOF
```
### Example 2: Command and Arguments
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-command-args
spec:
  containers:
  - name: my-container
    image: busybox:latest
    command: ["sh", "-c"]
    args: ["echo Hello from Kubernetes! && sleep 3600"]
EOF
```
#### Example 3: Passing Environment Variables to Commands
```bash

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-env-vars
spec:
  containers:
  - name: my-container
    image: alpine:latest
    command: ["/bin/sh", "-c"]
    args: ["echo \$GREETING"]
    env:
    - name: GREETING
      value: "Hello, Kubernetes!"
EOF
```

### Example 4: Passing Arguments to Docker Entrypoint
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-entrypoint-args
spec:
  containers:
  - name: my-container
    image: ubuntu:latest
    command: ["/bin/echo"]
    args: ["Hello", "Kubernetes!"]
EOF
```

### MultiContainer pod
### Use Cases for Multi-Container Pods
- Pods that run multiple containers that need to work together.
- A Pod can encapsulate an application composed of multiple co-located containers that are tightly coupled and need to share resources. 
- These co-located containers form a single cohesive unit.
![MultiContainerpod](/images/pod.svg)

- Here is an example for multiple container.
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-redis-pod
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
  - name: redis-container
    image: redis:latest
    ports:
    - containerPort: 6379
EOF
```
#### InitContainer
- A Pod can have multiple containers running apps within it, but it can also have one or more init containers, which are run before the app containers are started.
Init containers are exactly like regular containers, except:<br>

 - Init containers always run to completion.
 - Each init container must complete successfully before the next one starts.
- If a Pod's init container fails, the kubelet repeatedly restarts that init container until it succeeds.
- Regular init containers (in other words: excluding sidecar containers) do not support the lifecycle, livenessProbe, readinessProbe, or startupProbe fields.
- Init containers must run to completion before the Pod can be ready
- If you specify multiple init containers for a Pod, kubelet runs each init container sequentially
- Each init container must succeed before the next can run.
Here is an example for initContainer:
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
 name: init-container-pod
spec:
 containers:
 - name: main-container
   image: nginx:latest
   ports:
   - containerPort: 80
 initContainers:
 - name: init-wait-redis
   image: busybox:latest
   command: ["sh", "-c", "until nc -zv nginx 80; do echo 'Waiting for Redis to be ready'; sleep 1; done"]
EOF
```
-  Make a test by creating a pod and service 
  ```t
  kubectl run nginx --image=nginx
  kubectl expose pod/nginx --port 80
  ```
- Now check the pod status initContainer should be successful
- Also check the logs for initContainer
#### Sidecar container
- A Sidecar container extends and enhances the functionality of a preexisting container without changing it.
- This pattern is one of the fundamental container patterns that
allows single-purpose containers to cooperate closely together.
#### Adapter Container
- The Adapter pattern takes a heterogeneous containerized system and makes it conform to a consistent, unified interface with a standardized and normalized format that can be consumed by the outside world.
-  The Adapter pattern inherits all its characteristics
from the Sidecar,
### Resources definition in pod

#### Burstable QOS 
- Kubernetes assigns the Burstable class to a Pod when a container in the pod has more resource limit than the request value. 
- A pod in this category will have the following characteristics:
  - The Pod has not met the criteria for Guaranteed QoS class.</br>
  - A container in the Pod has an unequal memory or CPU request or limit
 An example is given below
  ```t
  resources:
        limits:
          memory: "300Mi"
          cpu: "800m"
        requests:
          memory: "100Mi"
          cpu: "600m"
    ```
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-with-resources
  spec:
    containers:
    - name: my-container
      image: nginx:latest
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
  EOF
  ```

- Check the pods if this is running 
- check the QOS of the pod
```t
kubectl describe pod | grep -i qos
```
#### Guranteed QOS 
- Kubernetes considers Pods classified as Guaranteed as a top priority. It won’t evict them until they exceed their limits. 
- A Pod with a Guaranteed class has the following characteristics:

    - All containers in the Pod have a memory limit and request.
    - All containers in the Pod have a memory limit equal to the memory request.
    - All containers in the Pod have a CPU limit and a CPU request.
    - All containers in the Pod have a CPU limit equal to the CPU request.
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-resources
spec:
  containers:
  - name: my-container
    image: nginx:latest
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "64Mi"
        cpu: "250m"
EOF
```
#### Best Efforts
- Kubernetes assigns the Burstable class to a Pod when a container in the pod has more resource limit than the request value. 
- A pod in this category will have the following characteristics:
  - The Pod has not met the criteria for Guaranteed QoS class.
  - A container in the Pod has an unequal memory or CPU request or limit
  - Pod without resourcs section are considered best efforts
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-resources
spec:
  containers:
  - name: my-container
    image: nginx:latest
   
EOF
```
#### imagepullSecret
 - Pod that uses a Secret to pull an image from a private container image registry or repository. 
 - There are many private registries in use. 
 - This task uses Docker Hub as an example registry.
##### Steps to Use a Pull Secret in a Pod YAML:

 - **Create Docker Config JSON File:**
   - Create a Docker configuration file (`~/.docker/config.json`) with the credentials for your private registry. 
   - You can use the `docker login` command to generate this file.

```bash
   docker login
```
### Create a Kubernetes Secret:
``` bash
kubectl create secret generic my-pull-secret --from-file=.dockerconfigjson=$HOME/.docker/config.json --type=kubernetes.io/dockerconfigjson
```
-  use the yaml to create pod with private registry
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pull-secret
spec:
  containers:
  - name: my-container
    image: myregistry.example.com/my-image:latest
  imagePullSecrets:
  - name: my-pull-secret
EOF
```

- Give permission to serviceAccount to pull image so that you will not have to mention in pod Spec
```t
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "mySecret"}]}'

```
[Learn about ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
## imagePullPolicy Examples

### 1. IfNotPresent

- **Description**: Pull the image only if it does not already exist locally.
- **Usage**: Use this policy when you want to minimize image pulls and rely on locally cached images.
- **Example**:
  ```yaml
  spec:
    containers:
    - name: my-container
      image: my-image:latest
      imagePullPolicy: IfNotPresent

### 1. IfNotPresent

- **Description**: Pull the image only if it does not already exist locally.
- **Usage**: Use this policy when you want to minimize image pulls and rely on locally cached images.
- **Example**:
  ```yaml
  spec:
    containers:
    - name: my-container
      image: my-image:latest
      imagePullPolicy: IfNotPresent

### 2. Always

  - **Description** : Always pull the latest version of the image, even if it already exists locally.
  -  **Usage**: Use this policy when you want to ensure that the container runs the latest version of the image.
  - **Example**:
  ```yaml
  spec:
  containers:
  - name: my-container
    image: my-image:latest
    imagePullPolicy: Always
```
### 3. Never

 - **Description**: Never pull the image, and only use the locally cached version if available.

 - **Usage** : Use this policy when you want to prevent Kubernetes from pulling the image, even if it does not exist locally.
  - **Example**:
   ```yaml
   spec:
  containers:
  - name: my-container
    image: my-image:latest
    imagePullPolicy: Never
```
### 4. IfPresent

   - **Description**: Pull the image only if it already exists locally. If the image does not exist locally, do not pull it.
   - **Usage**: Use this policy when you want to pull the image only if it is already available locally, otherwise use a cached version.
- **Example**
```yaml
spec:
  containers:
  - name: my-container
    image: my-image:latest
    imagePullPolicy: IfPresent
```

### 5. Default (IfNotPresent)

  - Description: Kubernetes default behavior if imagePullPolicy is not explicitly set. 
  - Pull the image only if it does not already exist locally.
  - Usage: This is the default behavior, and it is suitable for many use cases where you want to minimize image pulls.
  **Example** :
  ```yaml
  spec:
  containers:
  - name: my-container
    image: my-image:latest

```
### Container RestartPolicy in Pod block
### 1. Always

- **Description**: Always restart the container regardless of the exit status or reason.
- **Usage**: Use this policy for critical services that should always be running.
- **Example**:
```yaml
  spec:
    restartPolicy: Always
```
### 2. OnFailure

- Description: Restart the container only if it exits with a non-zero status.
 - Usage: Use this policy for jobs or batch processes that should be retried on failure.

  Example:
  ```yaml
  spec:
    restartPolicy: OnFailure
 ```
### 3. Never

- Description: Never restart the container, regardless of the exit status or reason.
- Usage: Use this policy for containers that are expected to run to completion and not be restarted.

Example:
  ```yaml
  spec:
   restartPolicy: Never
```
### 4. Default (Always)

 - Description: Kubernetes default behavior if restartPolicy is not explicitly set. Always restart the container.
 - Usage: This is the default behavior and is suitable for many long-running services.

Example
  ```yaml
  spec:
    restartPolicy: Always (default behavior)
  ```
## Advance Scheduling of  pod 
#### Using nodeName
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-scheduled-node
spec:
  nodeName: <mentionhere nodename>
  containers:
  - name: my-container
    image: nginx:latest
EOF
```

#### Steps to Use nodeSelector:

-  **Label the node**
  ```bash
  kubectl label nodes specific-node-name diskType=ssd
  ```
- **Use this label to schedule the pod** 

  ```bash
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-with-node-selector
  spec:
    containers:
    - name: my-container
      image: nginx:latest
    nodeSelector:
      diskType: ssd
  ```
- **Check where this pod has been schduled**

  ```t
  kubectl get pods -o wide
  ```


#### Taint and Toleration
- A taint allows a node to refuse pod to be scheduled unless that pod has a matching toleration.
- You apply taints to a node through the node specification (NodeSpec) and apply tolerations to a pod through the pod specification (PodSpec). A taint on a node instructs the node to repel all pods that do not tolerate the taint.
- Taints and tolerations consist of a key, value, and effect. An operator allows you to leave one of these parameters empty.

Taint and Toleration key points 
| Parameter | Description                                                                                           |
|-----------|-------------------------------------------------------------------------------------------------------|
| key       | Any string, up to 253 characters. Must begin with a letter or number, and may contain letters, numbers, hyphens, dots, and underscores. |
| value     | Any string, up to 63 characters. Must begin with a letter or number, and may contain letters, numbers, hyphens, dots, and underscores.   |
| effect    | One of: NoSchedule, PreferNoSchedule, NoExecute                                                        |
| operator  | One of: Equal, Exists                                                                                |

- **How to apply taint on a node**

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>

kubectl taint nodes specific-node-name disktype=ssd:NoSchedule

```
-  **Now use this in pod yaml**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  containers:
  - name: my-container
    image: nginx:latest
  tolerations:
  - key: disktype
    operator: Equal
    value: ssd
    effect: NoSchedule
EOF
```

#### Example : NotEqual Operator
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: toleration-not-equal-pod
spec:
  containers:
  - name: my-container
    image: nginx:latest
  tolerations:
  - key: disktype
    operator: NotEqual
    value: ssd
    effect: NoSchedule
EOF
```
#### Example Exists Operator with Key
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: toleration-exists-key-pod
spec:
  containers:
  - name: my-container
    image: nginx:latest
  tolerations:
  - key: disktype
    operator: Exists
    effect: NoSchedule
EOF
```

-  Check the pods where are they schedules

```t
kubectl get pod -o wide
```


### Node Affinity 

-   Label Nodes
```bash
kubectl label nodes kworker1 example-label=value1
kubectl label nodes kworker2 example-label=value2
```

- Yaml for pod creation
```bash
cat <<EOF > pod-with-node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-node-affinity
spec:
  containers:
  - name: nginx-container
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: example-label
            operator: In
            values:
            - value1
            - value2
EOF
```
### Example 2 with operator notin
```bash
cat <<EOF > pod-with-node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-node-affinity
spec:
  containers:
  - name: nginx-container
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: example-label
            operator: NotIn
            values:
            - value3
            - value4
EOF
```
#### Example Using Exists Operator
```bash
cat <<EOF > pod-with-node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-node-affinity
spec:
  containers:
  - name: nginx-container
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: example-label
            operator: Exists
EOF
```


- Check the pods status

### Example with Preferred Affinity
```bash
cat <<EOF > pod-with-preferred-node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-preferred-node-affinity
spec:
  containers:
  - name: nginx-container
    image: nginx
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: example-label
            operator: Exists
EOF
```


####  Pod Affinity and Anti Affinity

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pod-affinity
  labels:
    app: web
spec:
  containers:
  - name: nginx-container
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pod-affinity-rule
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web
        topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx-container
    image: nginx
EOF

```
#### Another Example with weight
```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-preferred-pod-affinity
  labels:
    app: database
spec:
  containers:
  - name: postgres-container
    image: postgres
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-preferred-pod-affinity-rule
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - database
          topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx-container
    image: nginx
EOF
```

#### Pod antiAffinity
```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-anti-affinity
  labels:
    app: web
spec:
  containers:
  - name: nginx-container
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-anti-affinity-rule
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web
        topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx-container
    image: nginx
EOF
```

#### Soft AntiAffinity  
```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-preferred-anti-affinity
  labels:
    app: database
spec:
  containers:
  - name: postgres-container
    image: postgres
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-preferred-anti-affinity-rule
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - database
          topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx-container
    image: nginx
EOF

```

### [Pod Priority](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)

#### Pod priority classes
- You can assign pods a priority class, which is a non-namespaced object that defines a mapping from a name to the integer value of the priority. The higher the value, the higher the priority.

- A priority class object can take any 32-bit integer value smaller than or equal to 1000000000 (one billion). Reserve numbers larger than one billion for critical pods that should not be preempted or evicted. By default, OpenShift Container Platform has two reserved priority classes for critical system pods to have guaranteed scheduling.

- System-node-critical - This priority class has a value of 2000001000 and is used for all pods that should never be evicted from a node. Examples of pods that have this priority class are sdn-ovs, sdn, and so forth.

- System-cluster-critical - This priority class has a value of 2000000000 (two billion) and is used with pods that are important for the cluster. Pods with this priority class can be evicted from a node in certain circumstances. For example, pods configured with the system-node-critical priority class can take priority. However, this priority class does ensure guaranteed scheduling. Examples of pods that can have this priority class are fluentd, add-on components like descheduler, and so forth.

- Sample priority class object
```sh
kubectl apply -f - <<EOF
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority 
value: 1000000 
globalDefault: false 
description: "This priority class should be used for XYZ service pods only." 
EOF
```
- Sample pod specification with priority class name
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
EOF 
```
## Pod Disruptions Budget:
- [Learn more about Disruptions Budget ]( https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
```sh
kubectl apply -f - <<EOF
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: example-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: example-app
EOF
```

### Use of pdb with Deployment 
```sh
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: example-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: example-app
EOF
```
### Pod Preemption and Pod Distruption Budget
- Pod preemption and other scheduler settings
If you enable pod priority and preemption, consider your other scheduler settings:
#### Pod priority and pod disruption budget
- A pod disruption budget specifies the minimum number or percentage of replicas that must be up at a time. If you specify pod disruption budgets, OpenShift Container Platform respects them when preempting pods at a best effort level. The scheduler attempts to preempt pods without violating the pod disruption budget. If no such pods are found, lower-priority pods might be preempted despite their pod disruption budget requirements.

#### Pod priority and pod affinity
Pod affinity requires a new pod to be scheduled on the same node as other pods with the same label.


  #### Pointer Regarding Pod Scheduling 
  -  Preemption removes existing Pods from a cluster under resource pressure to make room for higher priority pending Pods
  - The default priority for all pods is zero ( 0 )
  - Supported Operators for Affinity
    - The operator represents the relationship between the label on the node and the set of values in the matchExpression parameters in the pod specification.
    - This value can be below:

      - In,
      - NotIn, 
      - Exists
      - DoesNotExist
      - Lt
      - Gt

  - Specify a weight for the node, 1-100. The node that with highest weight is preferred.
  - A taint on a node instructs the node to repel all pods that do not tolerate the taint.
  -  Taint Effects are given below:
     

#### The effect is one of the following:

##### NoSchedule
	
  New pods that do not match the taint are not scheduled onto that node.
  Existing pods on the node remain.

##### PreferNoSchedule

  New pods that do not match the taint might be scheduled onto that node, but the scheduler tries not to.

  Existing pods on the node remain.

#### NoExecute
	

New pods that do not match the taint cannot be scheduled onto that node.

Existing pods on the node that do not have a matching toleration are removed.



### operator
#### Equal
The key/value/effect parameters must match. This is the default.
#### Exists
The key/effect parameters must match. You must leave a blank value parameter, which matches any.
- [Descheduler](https://github.com/kubernetes-sigs/descheduler)