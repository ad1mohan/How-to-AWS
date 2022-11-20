
# To install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# To install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"


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

# To create a private Node Group with key-pair (kube-kp) and instnace type 't3.meduim'.
eksctl create nodegroup \
                --cluster=test-cluster-1 \
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

# To download IAM Policy File
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# To Create IAM Service Account using same policy created above
# This command will also create a policy named AWSLoadBalancerControllerIAMPolicy in your account
eksctl create iamserviceaccount \
                --cluster=test-cluster-1 \
                --namespace=kube-system \
                --name=aws-load-balancer-controller \
                --attach-policy-arn=$(aws iam create-policy \
                                                    --policy-name AWSLoadBalancerControllerIAMPolicy \
                                                    --policy-document file://iam_policy_latest.json \
                                                    --query "Policy.Arn" \
                                                    --output text) \
                --override-existing-serviceaccounts \
                --approve

# To check is SA is being created
kubectl get sa aws-load-balancer-controller -n kube-system

# Install Open SSL
sudo yum install openssl -y

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh

# Add the eks-charts repository.
helm repo add eks https://aws.github.io/eks-charts

# Update your local repo to make sure that you have the most recent charts.
helm repo update

# Install the AWS Load Balancer Controller using Helm V3
## Replace Cluster Name, Region Code, VPC ID, Image Repo Account ID and Region Code  
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=test-cluster-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-02a0aef6f180d1752 \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller

# Treate Fargate Profile
eksctl create fargateprofile --cluster test-cluster-1 \
                             --name fp-test \
                             --namespace fp-test-dev

# To create a folder to keep yamls
mkdir ./yamls

# To create yamls
cat <<EOF > ./yamls/fargate_depolyment.yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: fp-dev
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-nginx-deployment
  labels:
    app: app1-nginx
  namespace: fp-test-dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1-nginx
  template:
    metadata:
      labels:
        app: app1-nginx
    spec:
      containers:
        - name: app1-nginx
          image: stacksimplify/kube-nginxapp1:1.0.0
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "500m"
            limits:
              memory: "500Mi"
              cpu: "1000m"                         
---
apiVersion: v1
kind: Service
metadata:
  name: app1-nginx-nodeport-service
  labels:
    app: app1-nginx
  namespace: fp-test-dev 
  annotations:
    #Important Note:  Need to add health check path annotations in service level if we are planning to use multiple targets in a load balancer    
    alb.ingress.kubernetes.io/healthcheck-path: /app1/index.html
spec:
  type: NodePort
  selector:
    app: app1-nginx
  ports:
    - port: 80
      targetPort: 80
---
# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app1-ingress-service
  namespace: fp-test-dev    
  annotations:
    # Load Balancer Name
    alb.ingress.kubernetes.io/load-balancer-name: ingress-fargatedemo
    # Ingress Core Settings  
    #kubernetes.io/ingress.class: "alb" (OLD INGRESS CLASS NOTATION - STILL WORKS BUT RECOMMENDED TO USE IngressClass Resource)
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Health Check Settings
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP 
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    #Important Note:  Need to add health check path annotations in service level if we are planning to use multiple targets in a load balancer    
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/success-codes: '200'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'  
    # For Fargate
    alb.ingress.kubernetes.io/target-type: ip    
spec:
  rules:
    - http:
        paths:      
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-nodeport-service
                port: 
                  number: 80                                   
# Important Note-1: In path based routing order is very important, if we are going to use  "/*", try to use it at the end of all rules.
EOF

kubectl apply -f ./yamls/