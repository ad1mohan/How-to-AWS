# DEPLOY THE OFFICIAL KUBERNETES DASHBOARD


We can deploy the dashboard with the following command:
```sh
export DASHBOARD_VERSION="v2.6.0"
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml
kubectl proxy --port=8080 --address=0.0.0.0 --disable-filter=true &
```

To the end of the URL and append
```sh
/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Get Token
```sh
aws eks get-token --cluster-name eksworkshop-eksctl | jq -r '.status.token'
```

Stop the proxy and delete the dashboard deployment
```sh
# kill proxy
pkill -f 'kubectl proxy --port=8080'

# delete dashboard
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml

unset DASHBOARD_VERSION
```
