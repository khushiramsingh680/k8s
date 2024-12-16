---
title: "Helm Overview"
weight: 21
no_list: true
---

# Topics
- What is Helm ?
- What is  Helm chart ?
- What is Helm release ?
- What is Helm repo ?
## What is Helm ?
Helm is a packager for Kubernetes that bundles related manifest files and packages them into a single logical deployment unit: Chart. Simplified, for many engineers Helm makes it easy to start using Kubernetes with real applications.

## What is Helm Chart ?
A Helm chart is a package that contains all the necessary resources to deploy an application to a Kubernetes cluster. This includes YAML configuration files for deployments, services, secrets, and config maps that define the desired state of your application.

## What is Helm release ?
A Release is an instance of a chart running in a Kubernetes cluster. One chart can often be installed many times into the same cluster.
## What is Helm repo ?
At a high level, a chart repository is where packaged charts can be stored and shared.
## Helm Installation

- Download and Installation of Helm from the  [link](https://github.com/helm/helm/releases)

- Install using script 
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
- Check helm version
```t
helm version
```
- [Click here](https://helm.sh/docs/topics/version_skew/) to check the helm version and k8s version compatiblity

### Prerequisites
### The following prerequisites are required for a successful and properly secured use of Helm.

- A Kubernetes cluster
- Installed  and configured Helm.
### Initialize a Helm Chart Repository
```t
helm repo add bitnami https://charts.bitnami.com/bitnami
```
- Check newly added repo.
```t
helm repo ls 
```
- How to search package in repo
```t
helm search repo bitnami
```
- Update helm repo
```t
helm repo update  
```
- Install an Example Chart
```t
helm install bitnami/mysql --generate-name
```

- Check all the release installed

```t
helm list
```
- Uninstall a Release
```t
helm uninstall <mysql-1612624192>
```
- Check the history if the flag --keep-history is provided
```t
 helm status mysql-1612624192
```
- Customizing the Chart Before Installing
```t
helm show values bitnami/wordpress
```
- **make the changes in values.yaml file and use below command to install**
```t
helm install -f values.yaml bitnami/wordpress --generate-name
```
 - How to get the value file 
```t
 helm get values happy-panda
```
- How to rollback to previous version
```t
helm rollback happy-panda 1
```
- How to check packages in all projects
```t
helm list --all
```
- Check all repositories
```t
helm repo list
```
## Creating Your Own Charts
- Use below command 
```t
helm create deis-workflow
```
- Validate your helm chart
```t
helm lint .
```
- How to package your helm chart
```t
 helm package deis-workflow
```
- How to install charts 

```t
helm install deis-workflow ./deis-workflow-0.1.0.tgz
```
- How to download the charts to your local vm
```t
helm pull [chart]
```
 - Displaya list of a chart’s dependencies
 ```t
helm dependency list [chart]
```



