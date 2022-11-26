**Liveness** probes are used in Kubernetes to know when a pod is alive or dead. A pod can be in a dead state for a variety of reasons; Kubernetes will kill and recreate the pod when a liveness probe does not pass.

**Readiness** probes are used in Kubernetes to know when a pod is ready to serve traffic. Only when the readiness probe passes will a pod receive traffic from the service; if a readiness probe fails traffic will not be sent to the pod.

CONFIGURE LIVENESS PROBE
Use the command below to create a directory
```sh
mkdir -p ~/environment/healthchecks
cat <<EoF > ~/environment/healthchecks/liveness-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-app
spec:
  containers:
  - name: liveness
    image: brentley/ecsdemo-nodejs
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
EoF
kubectl apply -f ~/environment/healthchecks/liveness-app.yaml
kubectl get pod liveness-app
kubectl describe pod liveness-app
```
Introduce a Failure
```sh
kubectl exec -it liveness-app -- /bin/kill -s SIGUSR1 1
kubectl get pod liveness-app
```
How can we check the status of the container health checks?
```sh
kubectl logs liveness-app
kubectl logs liveness-app --previous
```

CONFIGURE READINESS PROBE
```sh
cat <<EoF > ~/environment/healthchecks/readiness-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness-deployment
  template:
    metadata:
      labels:
        app: readiness-deployment
    spec:
      containers:
      - name: readiness-deployment
        image: alpine
        command: ["sh", "-c", "touch /tmp/healthy && sleep 86400"]
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 3
EoF
kubectl apply -f ~/environment/healthchecks/readiness-deployment.yaml
kubectl get pods -l app=readiness-deployment
kubectl describe deployment readiness-deployment | grep Replicas:
```
Introduce a Failure
```sh
kubectl exec -it <YOUR-READINESS-POD-NAME> -- rm /tmp/healthy
```
Check pods and deployment
```sh
kubectl get pods -l app=readiness-deployment
kubectl describe deployment readiness-deployment | grep Replicas:
```
CLEANUP
```sh
kubectl delete -f ~/environment/healthchecks/liveness-app.yaml
kubectl delete -f ~/environment/healthchecks/readiness-deployment.yaml

```
