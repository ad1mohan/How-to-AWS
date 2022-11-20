# Create EKS cluster with a private managed node group with two nodes


## Step-01: Create a Key Pair using AWS CLI
- Use below command to create a key pair using AWS CLI
```
# To create a keypair named kube-kp
aws ec2 create-key-pair --key-name my-key-pair-name
```