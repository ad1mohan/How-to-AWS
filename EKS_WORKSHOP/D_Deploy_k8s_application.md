Let’s bring up the NodeJS Backend API!
Copy/Paste the following commands into your Cloud9 workspace:
```sh
cd ~/environment/ecsdemo-nodejs
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```
We can watch the progress by looking at the deployment status:
```sh
kubectl get deployment ecsdemo-nodejs
```
Let’s bring up the Crystal Backend API!
Copy/Paste the following commands into your Cloud9 workspace:
```sh
cd ~/environment/ecsdemo-crystal
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```
We can watch the progress by looking at the deployment status:
```sh
kubectl get deployment ecsdemo-crystal
```
In AWS accounts that have never created a load balancer before, it’s possible that the service role for ELB might not exist yet.
We can check for the role, and create it if it’s missing.
```sh
aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" || aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
```
Let’s bring up the Ruby Frontend!
```sh
cd ~/environment/ecsdemo-frontend
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```
We can watch the progress by looking at the deployment status:
```sh
kubectl get deployment ecsdemo-frontend
```
FIND THE SERVICE ADDRESS
```sh
kubectl get service ecsdemo-frontend -o wide
ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')
curl -m3 -v $ELB
```
Now let’s scale up the backend services:
```sh
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
```
Let’s also scale our frontend service!
```sh
kubectl get deployments
kubectl scale deployment ecsdemo-frontend --replicas=3
kubectl get deployments
```
Undeploy the applications:
```sh
cd ~/environment/ecsdemo-frontend
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/environment/ecsdemo-crystal
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/environment/ecsdemo-nodejs
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
```
