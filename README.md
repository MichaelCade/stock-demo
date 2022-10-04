# Stock-Demo app ArgoCD demo

<p align="center">
<img src="/media/qr.png" width=400 height=400>
</p>

Original app created and forked from - https://github.com/tdewin/stock-demo 

You can find a talk where this repository and process is used by me - [Integrating backup into your Gitops CI/CD Pipelines](https://youtu.be/DcOYB4cUM9U?t=9787) - [KubeHuddle 2022](https://kubehuddle.com/2022/)

In this demo we will highlight how developers that are working with applications that have state should consider integrating backup operations into their GitOps workflow. This demo will simulate the deployment of our stock app via ArgoCD and enable us to simulate really how easy it is to manipulate your data outside of your source code and version control. 

The focus here is on using Kasten K10 with a [pre-sync phase](https://argoproj.github.io/argo-cd/user-guide/sync-waves/) in ArgoCD.

This demo will look at the following steps: 
- Deploy ArgoCD  
- Deploy Kasten K10 (Helm Chart Deployment)
- Deploy stock-demo app via ArgoCD 
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

`COMMAND FOR ARGOCD`

out of the box the service is not set to LoadBalancer if this is not changed then you can use the following command to access. 

`kubectl port-forward svc/stock-demo-svc -n stock-demo 8181:80`

## Implement change that affects our stock database (PostgreSQL) 

This is a very simple web front end stock app that our whole company use. Our company has been busy and we have some additional items to add to our stock system the below job will implement this change, but can you see the flaw. 

There are two options for implementing this change, if you are using the above repository then you can use the branch v2 within the argocd UI to issue the below change. If you have forked the repo and you would like to implement the below problem or issue into the app using git commands then please follow that instruction. 

TLDR; Using a psql command to add new products but in doing so it also clears all counters by mistake

```
#Make sure you are in the correct Kubernetes folder 
#cd kubernetes 

cat << EOF > new-stock-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: new-stock-job
  annotations: 
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "2"
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command: 
        - /bin/bash
        - -c
        - |
          #!/bin/bash
          PGPASSWORD=notsecure psql -h stockdb-postgresql -p 5432 -d stock -U root <<EOF
          select * from stock;
          INSERT INTO stock(product,unit,amount,price) VALUES ('Veeam Backup for Salesforce','User',10000.0,10000.0);
          EOF
          # Oh no !!  :(
          PGPASSWORD=notsecure psql -h stockdb-postgresql -p 5432 -d stock -U root <<EOF
          UPDATE stock SET(product,unit,amount,price) = ('Veeam VBR Socket','Socket',1000.0,0);
          EOF
        env:
            - name: POSTGRES_DB
              value: stock
            - name: POSTGRES_SERVER
              value: stockdb-postgresql
            - name: POSTGRES_USER
              value: root
            - name: POSTGRES_PORT
              value: "5432"
            - name: ADMINKEY
              value: unlock
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: stockdb-postgresql
        image: docker.io/bitnami/postgresql:14.5.0-debian-11-r14
        name: new-stock-job
      restartPolicy: Never
EOF
```
If now sync our application again this job will be ran and this can be checked with the following command. 
`kubectl get pods -n stock-demo`

To quickly see the outcome you can check the logs with this command. Changing the `new-stock-job-tsnl6` below with the value found above.
`kubectl logs new-stock-job-tsnl6 -n stock-demo`

If however we want to be sure of the mistake and you have time then you can perform the following on the StatefulSet PostgreSQL pod using the following commands to show you the updated values within the stock database. 

Connect to PostgreSQL pod 
`kubectl exec -it stockdb-postgresql-0 -n stock-demo -- bash`

Use psql to interact with your stock database 
`psql -h localhost -p 5432 -d stock -U root` 

Shows the content of our database 
`select * from stock;`

## Recovery 
Our mission-critical database is now in theory showing us wrong data, we managed to get the new item in with the correct stock levels but we accidently changed the values of stock within our database. The issue here is only found when you know that this data has been wrongly implemented or someone checks the stock levels, which could be weeks/months in the future. 

We can use Kasten K10 to get our data back from when this issue took place. This process will be the same regardless if you are using the code changes or branches.

## Resolving the code

If you are using the branch method then in ArgoCD you can change the branch to v3 to simulate this stage you do not need to delete or remove anything. 

After the recovery we will need to remove our `new-stock-job.yaml` so that we do not alter wrongly the stock value of some products. 

`rm -r new-stock-job.yaml`

```
cat << EOF > correct-stock-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: correct-stock-job
  annotations: 
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "2"
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command: 
        - /bin/bash
        - -c
        - |
          #!/bin/bash
          PGPASSWORD=notsecure psql -h stockdb-postgresql -p 5432 -d stock -U root <<EOF
          select * from stock;
          INSERT INTO stock(product,unit,amount,price) VALUES ('Veeam Backup for Salesforce','User',10000.0,10000.0);
          EOF
        env:
            - name: POSTGRES_DB
              value: stock
            - name: POSTGRES_SERVER
              value: stockdb-postgresql
            - name: POSTGRES_USER
              value: root
            - name: POSTGRES_PORT
              value: "5432"
            - name: ADMINKEY
              value: unlock
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: stockdb-postgresql
        image: docker.io/bitnami/postgresql:14.5.0-debian-11-r14
        name: correct-stock-job
      restartPolicy: Never
EOF
```

## Standard Stock-App Deployment steps 
If you are looking to deploy this Application outside of ArgoCD this is also possible, also a great demo for Kasten K10 + protecting PostgreSQL data, simulate this with buying some fake produce to maniuplate the stock numbers. 

- Deploy Application manifests under /kubernetes (disregard the ArgoCD folder if not using)
- Port-forward your application if not LoadBalancer is available `kubectl port-forward -n stock-demo service/stock-demo-svc 31602:80`
- Open browser at http://localhost:31602/ (If this is the first time you will see RED)
- You must init so that we build our database http://localhost:31602/init
- Use Kasten K10 to take your backup and export to S3
- Cause a problem to the App! Why not even zero the product stock all out! You could also use the job described above to add in some new products. 
- Restore from backup or export using Kasten K10

