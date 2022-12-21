```sh
# To install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname  -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# To install kubectl
curl -LO "https://dl.k8s.io/release/$(curl  -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"


cat << eof > cluster-fargate.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: fargate-cluster
  region: us-east-1

nodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 1

fargateProfiles:
  - name: fp-dev
    selectors:
      - namespace: dev
        labels:
          env: dev
          checks: passed
eof
eksctl create cluster -f cluster-fargate.yaml

cat << eof > log-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-app-deployment
  labels:
    app: log-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
      - name: log-generator
        image: chentex/random-logger:latest
        ports:
        - containerPort: 80
eof

```
