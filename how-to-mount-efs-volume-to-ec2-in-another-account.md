# Access EFS in closs account
https://repost.aws/knowledge-center/access-efs-across-accounts

# Perform a VPC Peering in the two accounts

# You cont use the DNS name for mounting, you need to use mount command with IP
```
sudo mkdir /mnt/efs
sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 10.0.2.68:/ /mnt/efs
ls /mnt/efs
```