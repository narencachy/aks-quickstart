# AKS Walkthrough

## Scenario
In this walkthrough of AKS basics, we're going to create an AKS cluster and deploy services to the cluster. Much of the code is based on <https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough> but additional details and scenarios are covered. Note that this is a high level overview, so a lot of topics are not covered and those that are covered are high level coverage. This is designed to be the first few steps on the AKS journey.

All deployments are done using the "declarative" approach - read more here: <https://kubernetes.io/docs/concepts/overview/object-management-kubectl/declarative-config/>

After creating the cluster,  we will deploy a simple Go Web app as a DaemonSet. This will create an Azure Public IP address and and an Azure Load Balancer. We will also use kubectl exec to execute commands on a running container, including an interactive bash shell.

Next, we will deploy a backend service based on the standard Redis Docker container, demonstrating how to connect to containers in the cluster via port forwarding. 

Next we will deploy the web front end (frontend) which uses the Redis backend to count votes. This will automatically create an Azure Public IP address and reconfigure the Azure Load Balancer.

Next we will scale the cluster from one node to three nodes. This will cause the Go Web App (webapp) to create 3 instances as one instance per node is the default for DaemonSets. The other services will not change. This step is optional.

Next we will install Helm onto the cluster and use a Helm Chart to deploy a Redis cluster. We will modify the web app to use the new cluster and delete the original backend deployment.

Lastly, we'll delete everything.

The walk through takes about 45 minutes.

## Prerequisites

This walk through has been tested using Azure Cloud Shell (bash), Mac terminal and Ubuntu (bash).

Everything you need is installed in Azure Cloud Shell. If you're using Mac or Ubuntu, you will need to install the Azure CLI first. <https://docs.microsoft.com/en-us/cli/azure/install-azure-cli>

You will also need to install kubectl (the Kubernetes CLI) and Helm.

```
# Install kubectl
az aks install-cli

#Mac
brew install kubernetes-helm

#Ubuntu
sudo snap install helm --classic
```

## Let's get started

AKS is not currently available in all regions, so we'll need to pick a region where it is available. The default in the script is Central US. We also need to verify that whatever size VMs we use for nodes are available in that region as well.

### List of regions where AKS is available

<https://docs.microsoft.com/en-us/azure/aks/container-service-quotas>

### List of VM Sizes in a region

```
# change centralus to a different region if desired
az vm list-sizes -l centralus -o table | grep s_v3
```

### Set environment variables

The script uses these environment variables for convenience.

```
# Change these if desired
AKSSUB=k8s
AKSLOC=centralus
AKSSIZE=Standard_D2s_v3

AKSRG=aks
AKSNAME=aks

MCRG=MC_${AKSRG}_${AKSNAME}_${AKSLOC}

# setup an alias (optional)
alias k=kubectl
```

### Login and select your Azure subscription

```
az login
az account set -s $AKSSUB
```

### Create a resource group and AKS Cluster

If you add AKS to an existing Resource Group, *DO NOT* delete the Resource Group in the cleanup stage!

```
az group create -l $AKSLOC -g $AKSRG

# this takes a while
az aks create -g $AKSRG -n $AKSNAME -c 1 -s $AKSSIZE
```

### What this did
If you check the Azure portal, you will see that this command created two resource groups - AKSRG and MC_AKSRG_AKSNAME_AKSLOC. The AKS resource group contains the AKS service (Controller). The other resource group contains the k8s nodes. Here is a screen shot of what is created in the nodes subscription. You will notice that there is one Node (VM).

![Initial node resource group](images/aks-node-initial.png)

### Get the k8s credentials and save in ~/.kube/config

```
az aks get-credentials -g $AKSRG -n $AKSNAME

# Note: if you run this quickstart repeatedly, you will need to edit / remove ~/.kube/control before running this command
```

### Wait for the node(s) to be ready

```
watch kubectl get nodes
```

### Deploy a simple go web app

This is a simple web app without any dependencies
```
kubectl apply -f webapp

# Wait for service to start
watch kubectl get svc,pods

# Browse to public IP
```

### Execute commands on a running container

Like Docker, kubectl lets you execute commands on a running container, including an interactive shell.

```
# get the full pod name for the webapp container
kubectl get pod

#replace webapp-xxxxx with the full pod name

# cat the log file
kubectl exec webapp-xxxxx -- cat logs/app.log

# run an interactive bash shell
kubectl exec -it webapp-xxxxx -- bash

```

### Node Resource Group

Notice that AKS added a Public IP and a Load Balancer after you deployed the webapp service. The YAML in webapp/svc.yaml specifies "LoadBalancer" as the type of service.

![screenshot](images/aks-node-fe2.png)

### Create and deploy backend service (Redis)

This uses the "declarative" approach based on the YAML files in the backend directory. The files are split into a services file and a deployment file but can also be combined. You can also use the -R option to recursively process the directory tree for more complex services.

```
kubectl apply -f backend

# Wait for the app to start
watch kubectl get deploy,pods
```

## Connect to the Redis container

```
# Start port forarding in background
kubectl port-forward svc/backend 6379:6379 &

# wait for port to be forwarded
# need to press enter to get back to prompt

# Run some Redis commands
bin/redis-cli

#Redis prompt
set Dogs 10
set Cats 1

get Dogs
get Cats

exit

# bash prompt
# Stop Port Forwarding
fg
# press <ctl> c
```

### Create and deploy frontend web app

```
kubectl apply -f frontend

# Get the public IP from the service output
watch kubectl get svc frontend

# browse to public IP to test app
```

### Node Resource Group

Notice that AKS added a Public IP and a Load Balancer after you deployed the frontend service. The YAML in frontend/svc.yaml specifies "LoadBalancer" as the type of service.

![screenshot](images/aks-node-frontend.png)


### Scale the k8s cluster to 3 nodes

```
# this step is optional and takes a while
# the Helm deployment will work without doing this step
az aks scale --no-wait -g $AKSRG -n $AKSNAME -c 3

# Wait for nodes to deploy
watch kubectl get nodes,pods
```

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

### Setting up Azure Applicaiton Gateway

This will setup an Azure Application Gateway with HTTPS support that points to a new app-gw service. This section is optional and takes about 30 minutes.

```
cd app-gw

# create the app
kubectl apply -f svc.yaml
kubectl apply -f daemon.yaml

# wait for service / pod to start
kubectl get svc,pods

# setup port forwarding for the web app
kubectl port-forward svc/app-gw 8080:80 &

# test the web app
curl localhost:8080

# end port forwarding
fg
# press <ctl> c

# Set MC RG Name
MCRG=MC_${AKSRG}_${AKSNAME}_${AKSLOC}

# Get the MC_VNet name
MCVNET=`az network vnet list -g $MCRG --query '[0].[name]' -o tsv`
echo $MCVNET

# Create the app gateway subnet
az network vnet subnet create --name app-gw-subnet --resource-group $MCRG --vnet-name $MCVNET --address-prefix 10.0.0.0/24

# Create the app gateway public IP
az network public-ip create --resource-group $MCRG --name app-gw-ip 

# Create the app gateway with a self-signed certificate
# This will take several minutes
az network application-gateway create -g $MCRG \
--name app-gw \
--vnet-name $MCVNET \
--subnet app-gw-subnet \
--capacity 2 --sku Standard_Small \
--http-settings-cookie-based-affinity Disabled \
--frontend-port 443 \
--http-settings-port 80 \
--http-settings-protocol Http \
--public-ip-address app-gw-ip \
--cert-file 4co.pfx \
--cert-password "wmtatx"

# get the external ip for the service
# Note that "external" applies to the k8s cluster
#      the IP address is a private IP to the VNet
MCADDR=`k get svc app-gw -o json | jq .status.loadBalancer.ingress[0].ip`
MCADDR=${MCADDR//\"}
echo $MCADDR

# update the backend pool to point to the k8s IP
az network application-gateway address-pool update -g $MCRG --gateway-name app-gw -n appGatewayBackendPool --servers $MCADDR

# Show the public IP
az network public-ip show -g $MCRG --name app-gw-ip --query [ipAddress] --output tsv

# Browse to https://app-gw-public-ip/

# Setup http to https redirection
# This takes a while

# create a front end port on 80
az network application-gateway frontend-port create -g $MCRG --gateway-name app-gw --port 80 --name httpPort

# create a redirect config 
az network application-gateway redirect-config create -g $MCRG --gateway-name app-gw \
--name httpToHttps --type Permanent \
--target-listener appGatewayHttpListener \
--include-path true --include-query-string true

# create an http listener on 80
az network application-gateway http-listener create -g $MCRG --gateway-name app-gw \
--frontend-ip appGatewayFrontendIP --frontend-port httpPort \
--name forceHttpsListener

# Apply the redirect config to the listener  
az network application-gateway rule create -g $MCRG --gateway-name app-gw \
--name rule2 --http-listener forceHttpsListener \
--rule-type Basic --redirect-config httpToHttps
  
```


### Clean up

```
# Delete resource groups
az group delete -y --no-wait -g $AKSRG
az group delete -y --no-wait -g MC_${AKSRG}_${AKSNAME}_${AKSLOC}
az group list -o table

# Only remove this file if this is the only k8s cluster in the file. Otherwise, edit the file and remove the key information

rm ~/.kube/config

```
