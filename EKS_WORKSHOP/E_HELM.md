Install the Helm CLI
```sh
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version --short
```
Download the stable repository so we have something to start with:
```sh
helm repo add stable https://charts.helm.sh/stable
helm search repo stable
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```
Deploy nginx With Helm
To update Helm’s local list of Charts, run:
```sh
# first, add the default repository, then update
helm repo add stable https://charts.helm.sh/stable
helm repo update
```
To list all Charts:
```sh
helm search repo
```
Nginx
```sh
helm search repo nginx
```
To add the Bitnami Chart repo to our local list of searchable charts:
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami
helm search repo bitnami/nginx
helm install mywebserver bitnami/nginx
kubectl get deploy,po,svc
kubectl describe deployment mywebserver
kubectl get pods -l app.kubernetes.io/name=nginx
kubectl get service mywebserver-nginx -o wide
```
CLEAN UP
```sh
helm uninstall mywebserver
```
kubectl will also demonstrate that our pods and service are no longer available:
```sh
kubectl get pods -l app.kubernetes.io/name=nginx
kubectl get service mywebserver-nginx -o wide
```
