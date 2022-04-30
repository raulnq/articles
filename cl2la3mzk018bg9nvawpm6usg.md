## Useful commands for Helm

> Helm is an open source package manager for Kubernetes. It provides the ability to provide, share, and use software built for Kubernetes.

In this post, we are going to cover the most common Helm command (and its most common parameters) from our day-to-day work and a brief review of the core concepts. You can review the official documentation [here](https://helm.sh/docs/).

# Helm core concepts
- The **Chart ** is a Helm package that contains information sufficient for installing a set of Kubernetes resources into a Kubernetes cluster.

- The **Repository ** is a simple HTTP server to store collection of charts. You can search, download and install charts from a repository.

- The **Release ** is an instance or a deployment of a chart. When you perform a helm install command, you are creating a new release of that chart on your Kubernetes cluster.

# Helm commands

## Repository management

List chart repositories:

```powershell
helm repo list
```
```powershell
helm repo list
NAME            URL
bitnami         https://charts.bitnami.com/bitnami
ingress-nginx   https://kubernetes.github.io/ingress-nginx
```
Search for charts in the local repositories:
```powershell
helm search repo [keyword]
```
```powershell
helm search repo rabbitmq
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/rabbitmq                        9.0.0           3.9.16          RabbitMQ is an open source general-purpose mess...
bitnami/rabbitmq-cluster-operator       2.6.0           1.12.1          The RabbitMQ Cluster Kubernetes Operator automa...
```
Search for charts in the [Artifact Hub](https://artifacthub.io/):
```powershell
helm search hub [keyword] -o yaml
```
```powershell
helm search hub jaeger -o yaml
- app_version: 1.17.1
  description: A Jaeger Helm chart for Kubernetes
  repository:
    name: hkube
    url: https://hkube.io/helm/
  url: https://artifacthub.io/packages/helm/hkube/jaeger
  version: 0.27.2006
- app_version: 1.30.0
  description: A Jaeger Helm chart for Kubernetes
  repository:
    name: jaegertracing
    url: https://jaegertracing.github.io/helm-charts
  url: https://artifacthub.io/packages/helm/jaegertracing/jaeger
  version: 0.56.1
```
Add a repository:
```powershell
helm repo add [repository] [repository-url]
```
```powershell
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
"jaegertracing" has been added to your repositories
```
Update information of charts in local repositories:
```powershell
helm repo update
```
```powershell
helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "jaegertracing" chart repository
```
## Chart management

### Using packages created by us

Create a directory containing the common chart files and directories:
```powershell
helm create [chart]
```
```
helm create demo-app
Creating demo-app
```
Examine a chart for possible issues:
```powershell
helm lint [chart]
```
```powershell
helm lint demo-app
==> Linting demo-app
[INFO] Chart.yaml: icon is recommended
```
Upgrade a release:
```powershell
helm upgrade [release] [chart]
```
```powershell
--namespace           namespace scope for this request
--install             if a release by this name doesn't already exist, run an install
--dry-run             simulate an upgrade
--debug               enable verbose output
--values              specify values in a YAML file or a URL (can specify multiple)
--set                 set values on the command line (can specify multiple or separate values with commas: key1=val1,key2=val2)
--create-namespace    if --install is set, create the release namespace if not present
``` 
```powershell
helm upgrade release-demo demo-app --debug --install --values values.local.yaml --set image.tag=1.0.0
```
Package a chart directory into a chart archive:
```powershell
helm package [chart] --app-version
```
```powershell
helm package demo-app --app-version 1.0.0
Successfully packaged chart and saved it to: C:\demo-app-1.0.0.tgz
```
Push a chart to a remote repository (depending on the server):

- [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-helm-repos)
- [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/push-oci-artifact.html)

### Using packages from a remote repository

Show the chart's definition:
```powershell
helm show chart [chart]
```
```powershell
helm show chart ingress-nginx/ingress-nginx
apiVersion: v2
appVersion: 1.2.0
description: Ingress controller for Kubernetes using NGINX as a reverse proxy and
  load balancer
home: https://github.com/kubernetes/ingress-nginx
icon: https://upload.wikimedia.org/wikipedia/commons/thumb/c/c5/Nginx_logo.svg/500px-Nginx_logo.svg.png
keywords:
- ingress
- nginx
kubeVersion: '>=1.19.0-0'
maintainers:
- name: rikatz
- name: strongjz
- name: tao12345666333
name: ingress-nginx
sources:
- https://github.com/kubernetes/ingress-nginx
type: application
version: 4.1.0
```
Show the chart's values:
```powershell
helm show values [chart]
```
Download a chart from a repository:
```powershell
helm pull [chart]
```
```powershell
helm pull ingress-nginx/ingress-nginx
```
Install a chart:
```powershell
helm install [release] [chart]
```
```
helm install ingress-nginx/ingress-nginx
```
## Release management
List releases:
```powershell
helm list --namespace [namespace]
```
```powershell
helm list --namespace default
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS  CHART                   APP VERSION
release-demo    default         3               2022-04-27 14:09:35.2202953 -0500 -05   failed  demo-app-1.0.0          1.16.0
```
Fetch release history:
```powershell
helm history [release]
```
```powershell
helm history release-demo
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION
1               Wed Apr 27 08:52:02 2022        superseded      demo-app-0.1.0          1.16.0          Upgrade complete
2               Wed Apr 27 12:53:03 2022        deployed        demo-app-0.1.0          1.16.0          Upgrade complete
3               Wed Apr 27 14:09:35 2022        failed          demo-app-1.0.0          1.16.0          Upgrade "demo-app" failed: pre-upgrade hooks failed: timed out waiting for the condition
```
Rollback a release to a previous revision:
```powershell
helm rollback [release] [revision]
```
```powershell
helm rollback demo-app 2
Rollback was a success! Happy Helming!
```
Uninstall a release:
```powershell
helm uninstall [release]
```
```powershell
helm uninstall demo-app
release "demo-app" uninstalled
```
