```
# To install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname  -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# To install kubectl
curl -LO "https://dl.k8s.io/release/$(curl  -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# To create a cluster
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
aws ec2 create-key-pair --key-name kube-kp --query=KeyMaterial --output=text > kube-kp.pem

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

# To test is there's any IAM service account associated with this cluster
eksctl get iamserviceaccount --cluster=test-cluster-1

# To download IAM Policy File
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# To Create IAM Service Account using same policy created above
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

# Install the AWS Load Balancer Controller using Helm V3
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

vpc_id=$(aws eks describe-cluster --name test-cluster-1 --query "cluster.resourcesVpcConfig.vpcId" --output text)


# To install aws lb controller
## Replace Cluster Name, Region Code, VPC ID, Image Repo Account ID and Region Code  
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=test-cluster-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$vpc_id \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller



Below are files used to create deployments and LBs

# To create a folder to keep yamls
mkdir ./yamls

# To create an app1 nginx pod
cat <<EOF > ./yamls/app1-nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-nginx-deployment
  labels:
    app: app1-nginx
spec:
  replicas: 1
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
          image: ad1mohan/interactive-application:v1.0
          ports:
            - containerPort: 80
EOF

# To create an NP service
cat <<EOF > ./yamls/app1-nginx-nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-nginx-nodeport-service
  labels:
    app: app1-nginx
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /app1/index.html
spec:
  type: NodePort
  selector:
    app: app1-nginx
  ports:
    - port: 80
      targetPort: 80  
EOF

# To create an app1 nginx pod
cat <<EOF > ./yamls/app2-nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-nginx-deployment
  labels:
    app: app2-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2-nginx
  template:
    metadata:
      labels:
        app: app2-nginx
    spec:
      containers:
        - name: app2-nginx
          image: ad1mohan/interactive-application-1:v1.0
          ports:
            - containerPort: 80
EOF

# To create an NP service
cat <<EOF > ./yamls/app2-nginx-nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app2-nginx-nodeport-service
  labels:
    app: app2-nginx
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /app2/index.html
spec:
  type: NodePort
  selector:
    app: app2-nginx
  ports:
    - port: 80
      targetPort: 80  
EOF

# To create an app1 nginx pod
cat <<EOF > ./yamls/app3-nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3-nginx-deployment
  labels:
    app: app3-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app3-nginx
  template:
    metadata:
      labels:
        app: app3-nginx
    spec:
      containers:
        - name: app3-nginx
          image: ad1mohan/interactive-application-2:v1.0
          ports:
            - containerPort: 80
EOF

# To create an NP service
cat <<EOF > ./yamls/app3-nginx-nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app3-nginx-nodeport-service
  labels:
    app: app3-nginx
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /index.html
spec:
  type: NodePort
  selector:
    app: app3-nginx
  ports:
    - port: 80
      targetPort: 80  
EOF

# To create a IngressClass
cat <<EOF > ./yamls/IngressClass.yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: my-aws-ingress-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb
EOF

# To create ingress file for path based routing
cat <<EOF > ./yamls/IngressFile.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-cpr-demo
  annotations:
    # Load Balancer Name
    alb.ingress.kubernetes.io/load-balancer-name: cpr-ingress
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
spec:
  ingressClassName: my-aws-ingress-class   # Ingress Class                  
  rules:
    - http:
        paths:      
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app2-nginx-nodeport-service
                port: 
                  number: 80
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app3-nginx-nodeport-service
                port: 
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-nodeport-service
                port: 
                  number: 80              
EOF

kubectl apply -f ./yamls/
```
