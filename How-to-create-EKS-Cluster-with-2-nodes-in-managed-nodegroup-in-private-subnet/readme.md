# Create EKS cluster with a private managed node group with two nodes


## Step-01: Complete pre-requisite (commands pasted below)
- Launch CloudShell
- Install eksctl and kubectl [Reference](https://github.com/ad1mohan/How-to-AWS/tree/main/How-to-install-eksctl-and-kubectl-in-AWS-CloudShell)
- Create EKS Cluster without node group [Reference](https://github.com/ad1mohan/How-to-AWS/tree/main/How-to-create-EKS-cluster-using-AWS-CloudShell)
- Associate IAM OIDC provider
- Create a keypair using AWS CLI to be used to SSH into nodes
```
# To install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname  -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# To install kubectl
curl -LO "https://dl.k8s.io/release/$(curl  -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

 # To create a cluster
 eksctl create cluster \
                        --name=test-cluster-1 \
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

```
## Step-02: Create a managed node group with two nodes in private subnets
- Paste below command in CloudShell
```
eksctl create nodegroup \
                        --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-private1 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking    
```
