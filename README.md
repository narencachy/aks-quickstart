# AKS Walkthrough

### Login and select your Azure subscription

```
az login
az subscrition set -s k8s
```

### Select a region that AKS is available in (centralus)

<https://docs.microsoft.com/en-us/azure/aks/container-service-quotas>

### Create a resource group in centralus (or other region)

```
az group create -l centralus -g k8s
```

### Choose a VM size that is available in your chosen region (Standard_D2s_v3)

```
az vm list-sizes -l centralus -o table | grep s_v3
```

### Create the AKS cluster (make sure the VM size is available in that region)

```
az aks create -g k8s -n k8s  -c 1 -u bartr -s Standard_D2s_v3 --nodepool-name west --no-wait
```

### Get the k8s credentials and save in ~/.kube/config

```
az aks get-credentials -g k8s -n k8s
    
Note that if you run this repeatedly, you will need to edit / remove ~/.kube/control
```

### List the k8s nodes (should be 1 unless you changed -c 1)

```
kubectl get nodes
```

### Create and deploy backend (Redis) and frontend (web) services

```
kubectl apply -f backend
kubectl apply -f frontend
```

### Wait for the apps to start

```
kubectl get deploy
```

### Get the public IP for frontend

```
kubectl get svc frontend

browse to public IP - add some votes
```

## Connect to the Redis container

### Start port forarding in background

```
kubectl port-forward svc/backend 6379:6379 &
```

### Run some Redis commands

```
redis-cli
set Dogs 10
set Cats 0

get Dogs
get Cats

Refresh web page
```

### Deploy a simple go web app

```
kubectl apply -f fe2
```

### Wait for service to deploy

```
wait kubectl get svc

Browse to public IP
```

### Scale the k8s cluster to 3 nodes

```
az aks scale --no-wait -g k8s -n k8s -c 3
```

### Wait for nodes to deploy

```
watch kubectl get nodes,pods
```

### Install Helm cli

```
Ubuntu: sudo snap install helm --classic
Mac: brew install kubernetes-helm
```

### Intialize Helm on cluster

```
cd helm
kubectl apply -f helm/rbac-helm.yaml
helm init --service-account tiller --upgrade
```

### Wait for tiller to be available

```
kubectl -n kube-system get deploy

helm list

helm install --values redis.yaml --name backend stable/redis 
```

### Change frontend

```
Edit frontend/deploy.yaml
Change "backend" to "backend-redis-master"
kubectl apply -f frontend/deploy.yaml
```

### Clean up by deleting everything

```
az group delete -y --no-wait -g k8s
az group delete -y --no-wait -g MC_k8s_k8s_centralus

Only remove this file if this is the only k8s cluster you use. Otherwise, edit the file and remove the key information

rm ~/.kube/config
```
