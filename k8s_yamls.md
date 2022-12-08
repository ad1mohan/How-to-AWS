# Create a cluster
```sh
cat << EOF > Create_a_cluster_with_ng.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: basic-cluster
  region: eu-north-1

nodeGroups:
  - name: ng-1
    instanceType: t2.micro
    desiredCapacity: 2
    volumeSize: 10
    ssh:
      allow: true # will use ~/.ssh/id_rsa.pub as the default ssh key
  - name: ng-2
    instanceType: t3.medium
    desiredCapacity: 2
    volumeSize: 10
    ssh:
      publicKeyPath: ~/.ssh/ec2_id_rsa.pub
EOF
eksctl create cluster -f Create_a_cluster_with_ng.yaml
```
