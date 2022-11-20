# Create EKS cluster using AWS CloudShell

## Step-01: Install eksctl and kubectl
- Paste below command in CloudShell
```
# To install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname  -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# To install kubectl
curl -LO "https://dl.k8s.io/release/$(curl  -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

```

## Step-02: Create EKS cluster without node group using below command
```
# To create a cluster
eksctl create cluster --name=test-cluster-1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
```

