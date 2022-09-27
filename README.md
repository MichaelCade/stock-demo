# Stock app ArgoCD demo

Original app created and forked from - https://github.com/tdewin/stock-demo 

In this demo we will highlight how developers that are working with applications that have state should consider integrating backup operations into their GitOps workflow. This demo will simulate the deployment of our stock app via ArgoCD and enable us to simulate really how easy it is to manipulate your data outside of your source code and version control. 

The focus here is on using Kasten K10 with a [pre-sync phase](https://argoproj.github.io/argo-cd/user-guide/sync-waves/) in ArgoCD.

This demo will look at the following steps: 
- Deploy ArgoCD  
- Deploy Kasten K10 (Helm Chart Deployment)
- Deploy stock-demo app via ArgoCD 
- Deploy PostgreSQL via Helm within ArgoCD 
- Implement change that affects our stock database (PostgreSQL) 

## PreReqs 

Deploy minikube cluster

```
minikube start --addons volumesnapshots,csi-hostpath-driver --apiserver-port=6443 --container-runtime=containerd -p nicconf-demo --kubernetes-version=1.21.2
```
## Deploy ArgoCD  

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
When pods are in ready state then you can forward using the following command. 

`kubectl port-forward svc/argocd-server -n argocd 9090:443`

Username is admin and password can be obtained with this command.

``` 
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## Deploy Kasten K10 (Helm Chart Deployment)

```
kubectl annotate volumesnapshotclass csi-hostpath-snapclass \
    k10.kasten.io/is-snapshot-class=true

helm install k10 kasten/k10 --namespace=kasten-io --create-namespace --set auth.tokenAuth.enabled=true --set injectKanisterSidecar.enabled=true --set-string injectKanisterSidecar.namespaceSelector.matchLabels.k10/injectKanisterSidecar=true

kubectl patch storageclass csi-hostpath-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

TOKEN_NAME=$(kubectl get secret --namespace kasten-io|grep k10-k10-token | cut -d " " -f 1)
TOKEN=$(kubectl get secret --namespace kasten-io $TOKEN_NAME -o jsonpath="{.data.token}" | base64 --decode)

echo "Token value: "
echo $TOKEN

kubectl --namespace kasten-io port-forward service/gateway 8080:8000

```

## Deploy stock-demo app via ArgoCD 
First let us confirm that we do not have a namespace called stock-demo as this will be created within ArgoCD

`kubectl get ns`

This app is deployed with Argo CD and is made of : 
* A stock-demo deployment
* A stockdb-postgresql statefulset 
* A Persistent Volume Claim 
* A secret 
* A service to stockdb-postgresql

At this stage we need a number of screen shots to highlight the adding of your application to ArgoCD

https://github.com/MichaelCade/stock-demo.git 

There is an option also to use the ArgoCD CLI to create (TBD)

## Implement change that affects our stock database (PostgreSQL) 





## Standard Stock-App Deployment steps 
Buy some fake produce. Mainly to manage some stateful data in postgres DB

First create a namespace
```
kubectl create ns stock-demo
```

Then install postgres via helm for example
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install -n stock-demo --set global.postgresql.auth.username=root --set global.postgresql.auth.password=notsecure --set global.postgresql.auth.database=stock stockdb bitnami/postgresql
```

Then deploy the app. Service deploys as LoadBalancer
```
kubectl -n stock-demo apply -f https://raw.githubusercontent.com/tdewin/stock-demo/main/kubernetes/deployment.yaml
kubectl -n stock-demo apply -f https://raw.githubusercontent.com/tdewin/stock-demo/main/kubernetes/svc.yaml
```
After deployment, use /init to create the basic table and insert some fake produce