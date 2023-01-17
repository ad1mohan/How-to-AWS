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

# Create Service account (Replace test-cluster-1, 111122223333 and us-east-1)
eksctl create iamserviceaccount \
    --cluster test-cluster-1 \
    --namespace kube-system \
    --name efs-csi-controller-sa \
    --attach-policy-arn arn:aws:iam::111122223333:policy/AmazonEKS_EFS_CSI_Driver_Policy \
    --approve \
    --region us-east-1

# Install the Amazon EFS driver using helm
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update

# Install by using proper repo url of Amazon container image registries
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.region-code.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa

# Create an Amazon EFS file system
# Retrieve the VPC ID that your cluster is in and store it in a variable for use in a later step. Replace test-cluster-1 with your cluster name
vpc_id=$(aws eks describe-cluster \
    --name test-cluster-1 \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)

# Retrieve the CIDR range for your cluster's VPC and store it in a variable for use in a later step. Replace us-east-1 with the AWS Region that your cluster is in.
cidr_range=$(aws ec2 describe-vpcs \
    --vpc-ids $vpc_id \
    --query "Vpcs[].CidrBlock" \
    --output text \
    --region us-east-1)

# Create a security group with an inbound rule that allows inbound NFS traffic for your Amazon EFS mount points.
security_group_id=$(aws ec2 create-security-group \
    --group-name MyEfsSecurityGroup \
    --description "My EFS security group" \
    --vpc-id $vpc_id \
    --output text)

# Create an inbound rule that allows inbound NFS traffic from the CIDR for your cluster's VPC.
aws ec2 authorize-security-group-ingress \
    --group-id $security_group_id \
    --protocol tcp \
    --port 2049 \
    --cidr $cidr_range

# Create an Amazon EFS file system for your Amazon EKS cluster
file_system_id=$(aws efs create-file-system \
    --region us-east-1 \
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)

# Create mount targets.
# Determine the IDs of the subnets in your VPC and which Availability Zone the subnet is in.
aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id subnet-EXAMPLEe2ba886490 \
    --security-groups $security_group_id
```
https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html#:~:text=security%2Dgroups%20%24security_group_id-,Deploy%20a%20sample%20application,-You%20can%20deploy
