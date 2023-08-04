
Configuration

```
kubectl create namespace elastic-stack

kubectl apply -f https://raw.githubusercontent.com/LucasDiogo96/eck-helm/main/eck-elastic-default-user-secret.yaml


helm repo add elastic https://helm.elastic.co && helm repo update

helm upgrade --install elastic-operator elastic/eck-operator -n elastic-stack
helm upgrade --install eck-stack elastic/eck-stack -f https://raw.githubusercontent.com/LucasDiogo96/eck-helm/main/eck-values.yaml -n elastic-stack

kubectl annotate service -n elastic-stack kibana-kb-http service.beta.kubernetes.io/azure-load-balancer-internal="true"
kubectl annotate service -n elastic-stack elasticsearch-es-http service.beta.kubernetes.io/azure-load-balancer-internal="true"
kubectl patch service -n elastic-stack elasticsearch-es-http -p '{"spec": {"type": "LoadBalancer"}}'
```

Uninstall

```
helm uninstall eck-stack -n elastic-stack
helm uninstall elastic-operator -n elastic-stack

```


Default username: elastic

```
kubectl get secret elasticsearch-es-elastic-user -n elk -o go-template='{{.data.elastic | base64decode}}' -n elastic-stack
```
