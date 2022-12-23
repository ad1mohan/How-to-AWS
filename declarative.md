## POD
```
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod-name
spec:
  containers:
    - args:
        - "3600"
      image: ad1mohan/alpine-sleeper
      name: simple-sleeper-container
```
