```
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
             --alb-ingress-access \
             --node-private-networking

# To create an IAM policy and role
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
aws iam create-policy \
    --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
    --policy-document file://iam-policy-example.json

# Create Service account (Replace my-cluster, 111122223333 and region-code)
eksctl create iamserviceaccount \
    --cluster my-cluster \
    --namespace kube-system \
    --name efs-csi-controller-sa \
    --attach-policy-arn arn:aws:iam::111122223333:policy/AmazonEKS_EFS_CSI_Driver_Policy \
    --approve \
    --region region-code

# Install the Amazon EFS driver using helm
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update

# Install by using proper repo url of Amazon container image registries
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa


```
