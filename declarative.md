## POD
```
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod-name
  labels:
    tier: frontend
spec:
  containers:
    - commands:
        - "sleep"
        - "60"
      name: simple-sleeper-container
      image: nginx:latest
```
## REPLICASET
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replica-set-name
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
```
## DEPLOYMENT
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployments-simple-deployment-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deployments-simple-deployment-app
  template:
    metadata:
      labels:
        app: deployments-simple-deployment-app
    spec:
      containers:
        - name: busybox
          image: busybox
          command:
            - sleep
            - "3600"
```
## NAMESPACE
```
apiVersion: v1
kind: Namespace
metadata:
  name: namespace-namespace
```
