```
# To test is there's any IAM service account associated with this cluster
eksctl get iamserviceaccount --cluster=test-cluster-1

# To download IAM Policy File
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
 
# Create IAM Policy using policy downloaded 
aws iam create-policy \
     --policy-name AWSLoadBalancerControllerIAMPolicy \
     --policy-document file://iam_policy_latest.json
```