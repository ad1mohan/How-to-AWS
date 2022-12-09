## Install in cli
```sh
# Install jq, envsubst (from GNU gettext utilities) and bash-completion
sudo yum -y install jq gettext bash-completion moreutils

# Install yq for yaml processing
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc


# To install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname  -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
# EKSCTL Bash Completion
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion


# To install kubectl
curl -LO "https://dl.k8s.io/release/$(curl  -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
# Enable kubectl bash_completion
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion

# eksctl version
eksctl version

# kubectl version
kubectl version

```
## Create cluster
```sh
# To create a cluster
eksctl create cluster --name=test-cluster-1 \
            --region=us-east-1 \
            --zones=us-east-1a,us-east-1b \
            --without-nodegroup 

# To associate IAM OIDC Provider 
eksctl utils associate-iam-oidc-provider \
   --region us-east-1 \
   --cluster test-cluster-1 \
   --approve

# To create a keypair named kube-kp
aws ec2 create-key-pair --key-name kube-kp

# To create a private Node Group with key-pair (kube-kp) and instnace type 't2.micro'.
eksctl create nodegroup --cluster=test-cluster-1 \
             --region=us-east-1 \
             --name=test-cluster-1-ng-private-1 \
             --node-type=t3.medium \
             --nodes=2 \
             --nodes-min=1 \
             --nodes-max=2 \
             --node-volume-size=10 \
             --ssh-access \
             --ssh-public-key=kube-kp \
             --managed \
             --asg-access \
             --external-dns-access \
             --full-ecr-access \
             --appmesh-access \
             --alb-ingress-access
             
```
## Install KUBE-OPS-VIEW (Deprecated)
```sh
# Install Open SSL
sudo yum install openssl -y

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
# Bash completion for the helm
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
helm version --short

# install k-ops
helm repo add stable https://charts.helm.sh/stable
helm install kube-ops-view \
stable/kube-ops-view \
--set service.type=LoadBalancer \
--set rbac.create=True

helm list

kubectl get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'
```
## CONFIGURE HORIZONTAL POD AUTOSCALER
```sh
# We will deploy the metrics server using Kubernetes Metrics Server.
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml

# Lets' verify the status of the metrics-server APIService
kubectl get apiservice v1beta1.metrics.k8s.io -o json | jq '.status'

# Deploy a Sample App
# The application is a custom-built image based on the php-apache image. The index.php page performs calculations to generate CPU load.
kubectl create deployment php-apache --image=us.gcr.io/k8s-artifacts-prod/hpa-example
kubectl set resources deploy php-apache --requests=cpu=200m
kubectl expose deploy php-apache --port 80

kubectl get pod -l app=php-apache

# This HPA scales up when CPU exceeds 50% of the allocated container resource.
kubectl autoscale deployment php-apache `#The target average CPU utilization` \
    --cpu-percent=50 \
    --min=1 `#The lower limit for the number of pods that can be set by the autoscaler` \
    --max=10 `#The upper limit for the number of pods that can be set by the autoscaler`

# 
