# Creating an Amazon EFS Volume and Mounting it to an EC2 Instance
## Step 1: Creating an VPC
```#!/bin/

# Step 1: Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyVPC}]' --query 'Vpc.VpcId' --output text)
echo "VPC created with ID: $VPC_ID"

# Enable DNS support and DNS hostname for the VPC
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
echo "DNS support and DNS hostname enabled for the VPC"

# Step 2: Create Public Subnets
PUBLIC_SUBNET_1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.0.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet1}]' --query 'Subnet.SubnetId' --output text)
PUBLIC_SUBNET_2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet2}]' --query 'Subnet.SubnetId' --output text)
echo "Public subnets created with IDs: $PUBLIC_SUBNET_1, $PUBLIC_SUBNET_2"

# Step 3: Create Private Subnets
PRIVATE_SUBNET_1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet1}]' --query 'Subnet.SubnetId' --output text)
PRIVATE_SUBNET_2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet2}]' --query 'Subnet.SubnetId' --output text)
echo "Private subnets created with IDs: $PRIVATE_SUBNET_1, $PRIVATE_SUBNET_2"

# Step 4: Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]' --query 'InternetGateway.InternetGatewayId' --output text)
echo "Internet Gateway created with ID: $IGW_ID"

# Step 5: Attach Internet Gateway to VPC
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
echo "Internet Gateway attached to VPC"

# Step 6: Create Route Table for Public Subnets
PUBLIC_RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRouteTable}]' --query 'RouteTable.RouteTableId' --output text)
echo "Public Route Table created with ID: $PUBLIC_RT_ID"

# Step 7: Create Route for Internet Gateway in Public Route Table
aws ec2 create-route --route-table-id $PUBLIC_RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
echo "Route added to Public Route Table for Internet Gateway"

# Step 8: Associate Public Subnets with Public Route Table
aws ec2 associate-route-table --subnet-id $PUBLIC_SUBNET_1 --route-table-id $PUBLIC_RT_ID
aws ec2 associate-route-table --subnet-id $PUBLIC_SUBNET_2 --route-table-id $PUBLIC_RT_ID
echo "Public Subnets associated with Public Route Table"

# Step 9: Create Security Group for NAT Gateway
NAT_SG_ID=$(aws ec2 create-security-group --group-name NAT-Security-Group --description "Security group for NAT gateway" --vpc-id $VPC_ID --query 'GroupId' --output text)
echo "NAT Security Group created with ID: $NAT_SG_ID"

# Step 10: Allocate Elastic IP for NAT Gateway
EIP_ALLOCATION_ID=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
echo "Elastic IP allocated for NAT Gateway with ID: $EIP_ALLOCATION_ID"

# Step 11: Create NAT Gateway
NAT_GATEWAY_ID=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_SUBNET_1 --allocation-id $EIP_ALLOCATION_ID --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=MyNATGateway}]' --query 'NatGateway.NatGatewayId' --output text)
echo "NAT Gateway creation in progress..."

# Wait until NAT Gateway is in the "available" state
while true; do
    NAT_GATEWAY_STATE=$(aws ec2 describe-nat-gateways --nat-gateway-ids $NAT_GATEWAY_ID --query 'NatGateways[0].State' --output text)
    if [ "$NAT_GATEWAY_STATE" == "available" ]; then
        break
    fi
    echo "Waiting for NAT Gateway to become available..."
    sleep 10
done

echo "NAT Gateway created with ID: $NAT_GATEWAY_ID"


# Step 12: Create Route Table for Private Subnets
PRIVATE_RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRouteTable}]' --query 'RouteTable.RouteTableId' --output text)
echo "Private Route Table created with ID: $PRIVATE_RT_ID"

# Step 13: Create Route for NAT Gateway in Private Route Table
aws ec2 create-route --route-table-id $PRIVATE_RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $NAT_GATEWAY_ID
echo "Route added to Private Route Table for NAT Gateway"

# Step 14: Associate Private Subnets with Private Route Table
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET_1 --route-table-id $PRIVATE_RT_ID
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET_2 --route-table-id $PRIVATE_RT_ID
echo "Private Subnets associated with Private Route Table"

echo "Script execution completed."
```

## Step 2: Create an EFS File System
1. Navigate to the AWS Management Console.
2. Open the Amazon EFS console.
3. Click on the "Create file system" button.
4. Configure the file system settings, such as the VPC, availability zones, and security groups.
5. Click on the "Next" button, review your settings, and click on the "Create file system" button.
6. Note the DNS name of the file system, as you'll need it later.

```
#!/bin/

# Check if required environment variables are set
if [ -z "$VPC_ID" ] || [ -z "$PRIVATE_SUBNET_1" ] || [ -z "$PRIVATE_SUBNET_2" ]; then
    echo "Please set the required environment variables before running this script."
    exit 1
fi

# Step 1: Create Security Group for EFS
EFS_SG_ID=$(aws ec2 create-security-group --group-name EFS-Security-Group --description "Security group for EFS" --vpc-id $VPC_ID --query 'GroupId' --output text)
echo "EFS Security Group created with ID: $EFS_SG_ID"

# Step 2: Authorize Ingress for EFS Security Group (allowing traffic from all instances in the VPC)
aws ec2 authorize-security-group-ingress --group-id $EFS_SG_ID --protocol tcp --port 2049 --source-security-group $EFS_SG_ID

# Step 3: Create Amazon EFS
EFS_ID=$(aws efs create-file-system --creation-token MyEFS --performance-mode generalPurpose --tags Key=Name,Value=MyEFS --query 'FileSystemId' --output text)
echo "Amazon EFS creation in progress..."

# Wait until EFS is in the "available" state
while true; do
    EFS_STATE=$(aws efs describe-file-systems --file-system-id $EFS_ID --query 'FileSystems[0].LifeCycleState' --output text)
    if [ "$EFS_STATE" == "available" ]; then
        break
    fi
    echo "Waiting for Amazon EFS to become available..."
    sleep 10
done

echo "Amazon EFS created with ID: $EFS_ID"

# Step 4: Create Mount Targets for Private Subnets
MOUNT_TARGET_1=$(aws efs create-mount-target --file-system-id $EFS_ID --subnet-id $PRIVATE_SUBNET_1 --security-group $EFS_SG_ID --query 'MountTargetId' --output text)
MOUNT_TARGET_2=$(aws efs create-mount-target --file-system-id $EFS_ID --subnet-id $PRIVATE_SUBNET_2 --security-group $EFS_SG_ID --query 'MountTargetId' --output text)
echo "Mount Targets created with IDs: $MOUNT_TARGET_1, $MOUNT_TARGET_2"

# Wait until Mount Targets are in the "available" state
while true; do
    MOUNT_TARGET_STATE_1=$(aws efs describe-mount-targets --mount-target-id $MOUNT_TARGET_1 --query 'MountTargets[0].LifeCycleState' --output text)
    MOUNT_TARGET_STATE_2=$(aws efs describe-mount-targets --mount-target-id $MOUNT_TARGET_2 --query 'MountTargets[0].LifeCycleState' --output text)

    if [ "$MOUNT_TARGET_STATE_1" == "available" ] && [ "$MOUNT_TARGET_STATE_2" == "available" ]; then
        break
    fi
    echo "Waiting for Mount Targets to become available..."
    sleep 10
done

echo "Script execution completed."
```

## Step 3: Create EC2 instance and modify the NFS SG such that it accepts traffic from EC2 instance on port 2049

## Step 4: Mounting Amazon EFS to EC2 Instance

```
# SSH into your EC2 Instance:
# Use your terminal or an SSH client to connect to your EC2 instance.
ssh -i path/to/your/keypair.pem ec2-user@your-ec2-instance-ip

# Update the EC2 Instance:
# Ensure your instance's package repositories are up to date.
sudo yum update -y

# Install the NFS Client:
# Install the NFS client on your EC2 instance.
sudo yum install -y nfs-utils

# Create a Mount Point:
# Create a directory on your EC2 instance to serve as the mount point for your EFS volume.
sudo mkdir /mnt/efs

# Mount the EFS Volume:
# Obtain the DNS name of your EFS file system from the AWS Management Console.
# Mount the EFS volume to the specified directory on your EC2 instance. Replace <Your-EFS-DNS-Name> with your EFS DNS name.
sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 fs-03a4e139e8850c08e.efs.us-east-1.amazonaws.com:/ /mnt/efs



<!-- Automatically Mount EFS on Boot (Optional):
If you want the EFS volume to be mounted automatically each time the EC2 instance is booted, you can add an entry to the /etc/fstab file.
echo "<Your-EFS-DNS-Name>:/ /mnt/efs nfs4 defaults 0 0" | sudo tee -a /etc/fstab -->

# Verify the Mount:
# Check if the EFS volume is successfully mounted.
df -h
# You should see the EFS volume mounted at /mnt/efs.
```

