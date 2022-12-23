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
      image: nginx
      name: simple-sleeper-container
```
