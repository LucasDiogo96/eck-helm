
Configuration

1- Create namespace if not exist
```
kubectl create namespace elastic-stack
```
2 - Update charts
```
helm repo add elastic https://helm.elastic.co && helm repo update
```
3 - Install eck stack & operator
```
helm upgrade --install elastic-operator elastic/eck-operator -n elastic-stack
helm upgrade --install eck-stack elastic/eck-stack -f https://raw.githubusercontent.com/LucasDiogo96/eck-helm/main/values.yaml -n elastic-stack
```
4 - Install eck stack & operator
```
kubectl annotate service -n elastic-stack eck-stack-eck-kibana-kb-http service.beta.kubernetes.io/azure-load-balancer-internal="true"
```

Extra commands: Uninstall

```
helm uninstall eck-stack -n elastic-stack
helm uninstall elastic-operator -n elastic-stack

```


Default username: elastic

```
kubectl get secret elasticsearch-es-elastic-user -n elk -o go-template='{{.data.elastic | base64decode}}' -n elastic-stack
```
