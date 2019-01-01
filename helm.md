# Helm Walkthrough (optional)

## Prerequisites

This walk through assumes you have completed the main walk through.

You will need to install Helm if not installed.

```
# Install Helm - Mac
brew install kubernetes-helm

# Install Helm - Ubuntu
sudo snap install helm --classic
```

## Scenario
In this walkthrough we will install Helm onto the cluster and use a Helm Chart to deploy a Redis cluster. We will modify the web app to use the new cluster and delete the original Redis deployment.

This walk through takes about 20 minutes.

### Intialize Helm on cluster

```
# Add RBAC role for Tiller
kubectl apply -f helm/rbac-helm.yaml

# Initialize Helm
helm init --service-account tiller --upgrade

# Wait for tiller to start
watch kubectl -n kube-system get deploy
```

### Install redis from Helm

```
helm install --values helm/redis.yaml --name backend stable/redis 

# Wait for pod to be ready
watch kubectl get pods

# Start port forarding in background
kubectl port-forward svc/backend-redis-master 6379:6379 &

# Run some Redis commands
bin/redis-cli

set Dogs 100
set Cats 2

# Stop Port Forwarding
fg

# Press <ctl> c
```

### Change frontend to use Helm deployed service

```
nano frontend/deploy.yaml

# Change "backend" to "backend-redis-master"
# Save

kubectl apply -f frontend/deploy.yaml

# revert the file
git checkout frontend/deploy.yaml

# Wait for pods
watch kubectl get pods
```

### Node Resource Group

Notice that AKS added a disk image to the node Resource Group. The following YAML sets up Redis Persistance in helm/redis.yaml

```
persistence:
    enabled: true
    path: /data
    subPath: ""
    accessModes:
    - ReadWriteOnce
    size: 8Gi
```

![screenshot](images/aks-node-helm.png)

### Delete original backend service

```
kubectl delete -f backend

# Wait for service to be deleted
watch kubectl get svc,pods
```
