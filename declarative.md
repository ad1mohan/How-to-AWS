## POD
```
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod-name
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
