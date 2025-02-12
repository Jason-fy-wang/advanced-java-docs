---
tags:
  - kubernetes
  - k8s
  - serviceaccount
---

create a serviceaccount for developer to communicate with k8s cluster.

```shell
# 1. create namespace
kubectl create ns my-ns

# 2. create serviceaccount
kubectl create -n my-ns serviceaccount my-sa

# 3. create token for sa
token=$(kubectl create token my-sa -n my-ns)

# 4. create clusterrole
kubectl create clusterrole secret-reader --resource=secrets,pods --verb=get,list,watch 

# 5. create rolebinding
kubectl create rolebinding -n my-ns my-sa-bind --role=secret-reader --serviceaccount=my-ns:my-sa --dry-run=client -o yaml

# 6. create context
kubectl config set-credentials my-sa-user --token=$token
kubectl config set-context my-sa-context --cluster=kubernetes --namespace=my-ns --user=my-sa-user

# 7. switch context
kubectl config use-context my-sa-context

# 8. communicate with cluster
kubectl get pods -n my-ns


## check context
kubectl config get-contexts

```





