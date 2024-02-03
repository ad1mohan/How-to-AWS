# Refer below doc
[Mount Amazon EFS file systems cross-account from Amazon EKS](https://aws.amazon.com/blogs/storage/mount-amazon-efs-file-systems-cross-account-from-amazon-eks/)


We will create following structure
![](https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2021/12/08/Mount-Amazon-EFS-file-systems-cross-account-from-Amazon-EKS-1.jpg)


## In Account A - Create a key pair named "AWS-Account-A-EKS-kp" in EC2

## In Account A - Create an EKS Cluster
Use following eksctl commands
```
# To install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname  -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# To create a cluster
eksctl create cluster --name=test-cluster-126 \
            --region=us-east-1 \
            --zones=us-east-1a,us-east-1b \
            --version=1.26 \
            --without-nodegroup

# To associate IAM OIDC Provider 
eksctl utils associate-iam-oidc-provider \
   --region us-east-1 \
   --cluster test-cluster-126 \
   --approve

# To create a private Node Group with key-pair (kube-kp) and instnace type 't2.micro'.
eksctl create nodegroup --cluster=test-cluster-126 \
             --region=us-east-1 \
             --name=test-cluster-126-ng-private-1 \
             --node-type=t3.medium \
             --nodes=1 \
             --nodes-min=1 \
             --nodes-max=1 \
             --node-volume-size=10 \
             --ssh-access \
             --ssh-public-key=AWS-Account-A-EKS-kp \
             --managed \
             --asg-access \
             --external-dns-access \
             --full-ecr-access \
             --appmesh-access \
             --alb-ingress-access \
             --node-private-networking
```


## In Account B - Create a VPC and perform a peering connection with EKS VPC in account A

## In Account B - Create a security group to attach to EFS

## In Account B - Create a EFS in the same VPC
(OPTIONAL) Create an EC2 in Account B, Attach the EFS and create a file and test.
(OPTIONAL) Create an EC2 in Account A, Attach the EFS and create a file and test.

## In Account B - Create IAM Role for EKS to assume and mount EFS
```
## Download the IAM trust relationship policy
$ curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/cross_account_mount/iam-policy-examples/trust-relationship-example.json -o iam_trust.json

## Edit the principal to include your EKS cluster's AWS account ID `A`.
"Principal": {
   "AWS": "arn:aws:iam::<aws-account-id-A>:root"
 }
 
## Create an IAM role with cross-account trust relationship
$ aws iam create-role \
     --role-name EFSCrossAccountAccessRole \
     --assume-role-policy-document file://iam_trust.json 

## Download the IAM policy to describe mount targets
$ curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/cross_account_mount/iam-policy-examples/describe-mount-target-example.json -o describe_mt.json

## Replace the resource for describe mount target with your file system ID's ARN in describe_mt.json
"Resource" : "arn:aws:elasticfilesystem:<region>:<aws-account-id-B>:file-system/<file-system-id>"

## Create an IAM policy to describe mount targets
$ aws iam create-policy \ 
    --policy-name EFSDescribeMountTargetIAMPolicy \
    --policy-document file://describe_mt.json
    
## Attach it to the cross-account role above
$ aws iam attach-role-policy \
    --role-name EFSCrossAccountAccessRole \
    --policy-arn "arn:aws:iam::<aws-account-id-B>:policy/EFSDescribeMountTargetIAMPolicy"
```

## In Account A - create and attach an IAM policy with sts assume permissions to cross-account IAM role created in Step 1. Attach this policy to IAM role associated with service account of driver’s controller service.
```
## Download sts assume iam policy
$ curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/cross_account_mount/iam-policy-examples/cross-account-assume-policy-example.json -o iam_assume.json

## Edit the resource with cross-account role created above
"Resource": "arn:aws:iam::<aws-account-id-B>:role/EFSCrossAccountAccessRole"

## Create an IAM policy for assume role
$ aws iam create-policy --policy-name AssumeEFSRoleInAccountB --policy-document file://iam_assume.json
```

## In Account A - Install EFS CSI Driver in EKS and create an IAMRSA
[EFS CSI Installation](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)
```
export cluster_name=test-cluster-126
export role_name=AmazonEKS_EFS_CSI_DriverRole
eksctl create iamserviceaccount \
    --name efs-csi-controller-sa \
    --namespace kube-system \
    --cluster $cluster_name \
    --role-name $role_name \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
    --approve

## Attach assume permissions to service account's IAM role
$ aws iam attach-role-policy  --role-name AmazonEKS_EFS_CSI_DriverRole --policy-arn "arn:aws:iam::<aws-account-id-A>:policy/AssumeEFSRoleInAccountB"

## Install EFS CSI driver using AWS Manamgent Console

## Describe controller's service account
$ kubectl describe sa efs-csi-controller-sa -n kube-system
Name:                efs-csi-controller-sa
Namespace:           kube-system
Labels:              app.kubernetes.io/name=aws-efs-csi-driver
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::<aws-account-id>:role/eksctl-iamserviceaccount-Role-XXYYZZ112233
Image pull secrets:  <none>
Events:              <none>

## Grab the IAM role in the annotations section from above result
ROLE="eksctl-iamserviceaccount-Role-XXYYZZ112233"
```

## In Account A - Create a kubernetes secret with awsRoleArn as the key and the cross-account assume role from step 1 as the value.
```
kubectl create secret generic x-account \
        --namespace=kube-system \
        --from-literal=awsRoleArn="arn:aws:iam::<aws-account-id-B>:role/EFSCrossAccountAccessRole
```

## In Account B - Add a file system policy to your file system in AWS account B to allow mounts from AWS account A hosting the EKS cluster. 
```
cat <<eof>file-system-policy.json
{
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:ClientMount",
                "elasticfilesystem:ClientWrite"
            ],
            "Principal": {
                "AWS": "arn:aws:iam::<aws-account-id-A>:root" # Replace with AWS account ID of EKS cluster
            }
        }                                                                                                 
    ]
}
eof

## Put file system policy to your file system
$ aws efs put-file-system-policy --file-system-id fs-abcd1234 \
    --policy file://file-system-policy.json.json
```

## In Account A - Create a kubernetes service account for driver’s node daemonset.
```
## Create a Kubernetes service account 
$ eksctl create iamserviceaccount \ 
  --cluster=<cluster> \ 
  --region <AWS Region> \ 
  --namespace=kube-system \ 
  --name=efs-csi-node-sa \ 
  --override-existing-serviceaccounts \ 
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonElasticFileSystemClientFullAccess \ 
  --approve
```

## In Account B - Create following resources
```
cat <<eof>SC.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
mountOptions:
  - tls
  - iam
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0262f0b6dfe253362
  directoryPerms: "700"
  basePath: "/"
  az: us-east-1b
  csi.storage.k8s.io/provisioner-secret-name: x-account
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
eof
kubectl apply -f SC.yaml
```
```
cat <<eof>pvc-pod.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-app
spec:
  containers:
    - name: app
      image: centos
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo $(date -u) >> /data/out; sleep 5; done"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: efs-claim
eof
kubectl apply -f pvc-pod.yaml
```

## Fix1
```
[cloudshell-user@ip-10-138-162-118 ~]$ kubectl describe pvc efs-claim
Name:          efs-claim
Namespace:     default
StorageClass:  efs-sc
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-provisioner: efs.csi.aws.com
               volume.kubernetes.io/storage-provisioner: efs.csi.aws.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       efs-app
Events:
  Type     Reason                Age               From                                                                                      Message
  ----     ------                ----              ----                                                                                      -------
  Normal   ExternalProvisioning  8s (x4 over 24s)  persistentvolume-controller                                                               waiting for a volume to be created, either by external provisioner "efs.csi.aws.com" or manually created by system administrator
  Normal   Provisioning          8s (x5 over 24s)  efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  External provisioner is provisioning volume for claim "default/efs-claim"
  Warning  ProvisioningFailed    8s (x5 over 23s)  efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  failed to provision volume with StorageClass "efs-sc": error getting secret x-account in namespace kube-system: secrets "x-account" is forbidden: User "system:serviceaccount:kube-system:efs-csi-controller-sa" cannot get resource "secrets" in API group "" in the namespace "kube-system"
```
Sol:
```
kubectl edit clusterrole efs-csi-external-provisioner-role


# Add "secters" in get and save and reapply pod and pvc
```

## Fix2
```
[cloudshell-user@ip-10-138-162-118 ~]$ kubectl describe pvc efs-claim
Name:          efs-claim
Namespace:     default
StorageClass:  efs-sc
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-provisioner: efs.csi.aws.com
               volume.kubernetes.io/storage-provisioner: efs.csi.aws.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       efs-app
Events:
  Type     Reason              Age   From                                                                                      Message
  ----     ------              ----  ----                                                                                      -------
  Warning  ProvisioningFailed  12s   efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  failed to provision volume with StorageClass "efs-sc": rpc error: code = Internal desc = Failed to fetch Access Points or Describe File System: List Access Points failed: AccessDenied: User: arn:aws:sts::058264288323:assumed-role/AmazonEKS_EFS_CSI_DriverRole/1706609159791232853 is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::058264283673:role/EFSCrossAccountAccessRole
           status code: 403, request id: 42253867-7421-418b-b89a-49a22bd3fc21
  Warning  ProvisioningFailed  11s  efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  failed to provision volume with StorageClass "efs-sc": rpc error: code = Internal desc = Failed to fetch Access Points or Describe File System: List Access Points failed: AccessDenied: User: arn:aws:sts::058264288323:assumed-role/AmazonEKS_EFS_CSI_DriverRole/1706609160832056622 is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::058264283673:role/EFSCrossAccountAccessRole
           status code: 403, request id: b1adffbc-91e5-4510-b76d-ad58ef467f66
  Warning  ProvisioningFailed  9s  efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  failed to provision volume with StorageClass "efs-sc": rpc error: code = Internal desc = Failed to fetch Access Points or Describe File System: List Access Points failed: AccessDenied: User: arn:aws:sts::058264288323:assumed-role/AmazonEKS_EFS_CSI_DriverRole/1706609162865353897 is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::058264283673:role/EFSCrossAccountAccessRole
           status code: 403, request id: f8f986a3-c88d-424e-904a-50ecf5e70182
  Normal   ExternalProvisioning  7s (x2 over 12s)  persistentvolume-controller                                                               waiting for a volume to be created, either by external provisioner "efs.csi.aws.com" or manually created by system administrator
  Normal   Provisioning          5s (x4 over 12s)  efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  External provisioner is provisioning volume for claim "default/efs-claim"
  Warning  ProvisioningFailed    5s                efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  failed to provision volume with StorageClass "efs-sc": rpc error: code = Internal desc = Failed to fetch Access Points or Describe File System: List Access Points failed: AccessDenied: User: arn:aws:sts::058264288323:assumed-role/AmazonEKS_EFS_CSI_DriverRole/1706609166903567505 is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::058264283673:role/EFSCrossAccountAccessRole
           status code: 403, request id: 51c4a3dd-b79d-400e-8f56-b16753788c59
```
Sol:
```
Account ID was incorrect in policy AssumeEFSRoleInAccountB so corrected that.
```

## Fix3
```
[cloudshell-user@ip-10-138-162-118 ~]$ kubectl describe pvc efs-claim
Name:          efs-claim
Namespace:     default
StorageClass:  efs-sc
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-provisioner: efs.csi.aws.com
               volume.kubernetes.io/storage-provisioner: efs.csi.aws.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       efs-app
Events:
  Type     Reason                Age              From                                                                                      Message
  ----     ------                ----             ----                                                                                      -------
  Normal   ExternalProvisioning  6s               persistentvolume-controller                                                               waiting for a volume to be created, either by external provisioner "efs.csi.aws.com" or manually created by system administrator
  Normal   Provisioning          3s (x3 over 6s)  efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  External provisioner is provisioning volume for claim "default/efs-claim"
  Warning  ProvisioningFailed    2s (x3 over 6s)  efs.csi.aws.com_efs-csi-controller-68d6b7d884-rrdvb_33a392e4-e210-47cf-bd90-1a78a68778dc  failed to provision volume with StorageClass "efs-sc": rpc error: code = Unauthenticated desc = Access Denied. Please ensure you have the right AWS permissions: Access denied
```
Sol:
```
NewClientConnection permission missing in policy "EFSDescribeMountTargetIAMPolicy" in account b
Adding admin access helped the volume to be provisioned.
```

## Fix4
```
[cloudshell-user@ip-10-138-162-118 ~]$ kubectl describe pod
Name:             efs-app
Namespace:        default
Priority:         0
Service Account:  default
Node:             ip-192-168-100-28.ec2.internal/192.168.100.28
Start Time:       Tue, 30 Jan 2024 10:31:34 +0000
Labels:           <none>
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Containers:
  app:
    Container ID:  
    Image:         centos
    Image ID:      
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      while true; do echo Tue Jan 30 09:59:47 AM UTC 2024 >> /data/out; sleep 5; done
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /data from persistent-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-bg6d6 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  persistent-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  efs-claim
    ReadOnly:   false
  kube-api-access-bg6d6:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  2m18s               default-scheduler  0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling..
  Normal   Scheduled         2m16s               default-scheduler  Successfully assigned default/efs-app to ip-192-168-100-28.ec2.internal
  Warning  FailedMount       13s                 kubelet            Unable to attach or mount volumes: unmounted volumes=[persistent-storage], unattached volumes=[persistent-storage kube-api-access-bg6d6]: timed out waiting for the condition
  Warning  FailedMount       3s (x9 over 2m15s)  kubelet            MountVolume.SetUp failed for volume "pvc-355a60d3-1835-4de2-92c6-02ac8248ddfc" : rpc error: code = Internal desc = Could not mount "fs-0262f0b6dfe253362:/" at "/var/lib/kubelet/pods/45df82e0-3185-49ce-bac0-ac72b51ac59d/volumes/kubernetes.io~csi/pvc-355a60d3-1835-4de2-92c6-02ac8248ddfc/mount": mount failed: exit status 32
Mounting command: mount
Mounting arguments: -t efs -o mounttargetip=10.0.155.37,accesspoint=fsap-0efb07179fd41d203,tls,iam fs-0262f0b6dfe253362:/ /var/lib/kubelet/pods/45df82e0-3185-49ce-bac0-ac72b51ac59d/volumes/kubernetes.io~csi/pvc-355a60d3-1835-4de2-92c6-02ac8248ddfc/mount
Output: Could not start amazon-efs-mount-watchdog, unrecognized init system "aws-efs-csi-dri"
b'mount.nfs4: access denied by server while mounting 127.0.0.1:/'
Warning: config file does not have fips_mode_enabled item in section mount.. You should be able to find a new config file in the same folder as current config file /etc/amazon/efs/efs-utils.conf. Consider update the new config file to latest config file. Use the default value [fips_mode_enabled = False].Warning: config file does not have retry_nfs_mount_command item in section mount.. You should be able to find a new config file in the same folder as current config file /etc/amazon/efs/efs-utils.conf. Consider update the new config file to latest config file. Use the default value [retry_nfs_mount_command = True].
```
Sol:
```
Updated SC and removed File system policy

cat <<eof>SC.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0262f0b6dfe253362
  directoryPerms: "700"
  gidRangeStart: "1000" # optional
  gidRangeEnd: "2000" # optional
  basePath: "/dynamic_provisioning" # optional
  az: us-east-1a
  csi.storage.k8s.io/provisioner-secret-name: x-account
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
eof
kubectl apply -f SC.yaml

and reapplied and fixed
```


export cluster_name=test-cluster-128
export role_name=AmazonEKS_EFS_CSI_DriverRole
eksctl create iamserviceaccount   --name efs-csi-controller-sa   --namespace kube-system   --cluster $cluster_name   --role-name $role_name   --role-only   --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy   --override-existing-serviceaccounts   --approve


kubectl delete -f test.yaml
cat <<eof>SC.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
mountOptions:
  - tls
  - iam
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-03a4e139e8850c08e
  directoryPerms: "700"
  basePath: "/"
  az: us-east-1b
  csi.storage.k8s.io/provisioner-secret-name: x-account
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
eof
kubectl apply -f SC.yaml

kubectl delete -f PVC-POD.yaml
cat <<eof>PVC-POD.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: date-echo-pod
spec:
  containers:
  - name: date-echo-container
    image: amazonlinux:2
    command:
    - "/bin/sh"
    - "-c"
    - "mkdir -p /mnt/data && while true; do date >> /mnt/data/date.txt; sleep 1; done"
    
eof
kubectl apply -f PVC-POD.yaml
sleep 2
kubectl logs -f -n kube-system test-pod

    env:
    - name: AWS_ACCESS_KEY_ID
      valueFrom:echo
        secretKeyRef:
          name: aws-credentials
          key: access-key-id
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: aws-credentials
          key: secret-access-key

eksctl create iamserviceaccount  --cluster=test-cluster-128  --region=us-east-1  --namespace=kube-system  --name=efs-csi-node-sa --override-existing-serviceaccounts  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonElasticFileSystemClientFullAccess  --approve


cannot get resource "secrets" in API group "" in the namespace "kube-system"


Warning  FailedScheduling  2m47s                default-scheduler  0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling..

Normal   Scheduled         2m46s                default-scheduler  Successfully assigned default/efs-app to ip-10-1-155-124.ec2.internal

Warning  FailedMount       33s (x9 over 2m45s)  kubelet            

MountVolume.SetUp failed for volume "pvc-527c7bf4-55be-4cc2-a235-fbdb7fa0ff13" : 
rpc error: code = Internal desc = Could not mount "fs-03a4e139e8850c08e:/" at "/var/lib/kubelet/pods/a4cc18dc-3e60-460a-b84d-db625b501911/volumes/kubernetes.io~csi/pvc-527c7bf4-55be-4cc2-a235-fbdb7fa0ff13/mount": mount failed: exit status 32

Mounting command: mount
Mounting arguments: -t efs -o mounttargetip=10.0.2.68,accesspoint=fsap-08683d818f57f3cee,tls,iam fs-03a4e139e8850c08e:/ /var/lib/kubelet/pods/a4cc18dc-3e60-460a-b84d-db625b501911/volumes/kubernetes.io~csi/pvc-527c7bf4-55be-4cc2-a235-fbdb7fa0ff13/mount

Output: Could not start amazon-efs-mount-watchdog, unrecognized init system "aws-efs-csi-dri"
b'mount.nfs4: access denied by server while mounting 127.0.0.1:/'

Warning: config file does not have fips_mode_enabled item in section mount.. You should be able to find a new config file in the same folder as current config file /etc/amazon/efs/efs-utils.conf. Consider update the new config file to latest config file. Use the default value [fips_mode_enabled = False].

Warning: config file does not have retry_nfs_mount_command item in section mount.. You should be able to find a new config file in the same folder as current config file /etc/amazon/efs/efs-utils.conf. Consider update the new config file to latest config file. Use the default value [retry_nfs_mount_command = True].