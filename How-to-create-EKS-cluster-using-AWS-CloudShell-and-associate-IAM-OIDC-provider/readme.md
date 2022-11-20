# Create EKS cluster using AWS CloudShell

## Step-01: Complete pre-requisite (commands pasted below)
- Launch CloudShell
- Install eksctl and kubectl [Reference](https://github.com/ad1mohan/How-to-AWS/tree/main/How-to-install-eksctl-and-kubectl-in-AWS-CloudShell)
- Create EKS Cluster without Node Group [Reference](https://github.com/ad1mohan/How-to-AWS/tree/main/How-to-create-EKS-cluster-using-AWS-CloudShell)
```
# To install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname  -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# To install kubectl
curl -LO "https://dl.k8s.io/release/$(curl  -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# To create a cluster
eksctl create cluster --name=test-cluster-1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
```

## Step-02: Associate IAM OIDC Provider
```
# To associate IAM OIDC Provider
eksctl utils associate-iam-oidc-provider \
                                        --region us-east-1 \
                                        --cluster test-cluster-1 \
                                        --approve
```

