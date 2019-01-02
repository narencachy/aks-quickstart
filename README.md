# AKS Walk Through
#### 100 level

## Scenario
In this walk through of AKS basics, we're going to create an AKS cluster and deploy services to the cluster. Much of the code is based on <https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough> but additional details and scenarios are covered. Note that this is a high level overview, so a lot of topics are not covered and those that are covered are high level. This is designed to be the first few steps on the AKS journey. 

All deployments are done using the "declarative" approach - read more here: <https://kubernetes.io/docs/concepts/overview/object-management-kubectl/declarative-config/>

After creating the cluster,  we will deploy a simple Go Web app as a DaemonSet. This will create an Azure Public IP address and and a public Azure Load Balancer. We will also use kubectl exec to execute commands on a running container, including an interactive bash shell.

Next, we will deploy a backend service based on the standard Redis Docker container, demonstrating how to connect to containers in the cluster via port forwarding. 

Next we will deploy the web front end (frontend) which uses the Redis backend to count votes. This will automatically create an Azure Public IP address and reconfigure the Azure Load Balancer.

Next we will create a web app that uses a private load balancer and setup Azure Application Gateway to provide HTTPS offloading.

Lastly, we'll delete everything.

The full walk through takes about 90 minutes.

### Additional Reading
* AKS: <https://docs.microsoft.com/en-us/azure/aks/>
* Kubernetes: <https://kubernetes.io/docs/home/>

## Prerequisites

This walk through has been tested using Azure Cloud Shell (bash), Mac terminal and Ubuntu (bash).

Everything you need is installed in Azure Cloud Shell. If you're using Mac or Ubuntu, you will need to install the Azure CLI first. <https://docs.microsoft.com/en-us/cli/azure/install-azure-cli>

You will also need to install kubectl (the Kubernetes CLI).

```
# Install kubectl
az aks install-cli
```

## Let's get started

### Set environment variables

The script uses these environment variables extensively

AKS is not currently available in all regions, so we'll need to pick a region where it is available. The default in setenv is CentralUS. We also need to verify that whatever size VMs we use for nodes are available in that region as well. The default in setenv is Standard_D2s_v3. You can change these by editing setenv.

```
# Clone this repo
git clone https://github.com/bartr/aks-quickstart

# edit setenv to use your values
source setenv
```

### List of regions where AKS is available

<https://docs.microsoft.com/en-us/azure/aks/container-service-quotas>

### List of VM Sizes in your region

```
az vm list-sizes -l $AKSLOC -o table | grep s_v3
```

### Login and select your Azure subscription

```
# Optional if you're already logged in and have the subscription default set
az login
az account set -s $AKSSUB
```

### Create a resource group and AKS Cluster

If you add AKS to an existing Resource Group, *DO NOT* delete the Resource Group in the cleanup stage!

```
az group create -l $AKSLOC -g $AKSRG

# this takes a while
az aks create -g $AKSRG -n $AKSNAME -c 3 -s $AKSSIZE
```

### What this did
If you check the Azure portal, you will see that this command created two resource groups - AKSRG and MC_AKSRG_AKSNAME_AKSLOC. The AKS resource group contains the AKS service (Controller). The other resource group contains the k8s nodes. Here is a screen shot of what is created in the nodes subscription. You will notice that there are 3 Nodes (VMs).

![Initial node resource group](images/aks-node-initial.png)

### Get the k8s credentials and save in ~/.kube/config

```
az aks get-credentials -g $AKSRG -n $AKSNAME

# Note: if you run this walk through repeatedly, you will need to edit / delete ~/.kube/control before running this command
```

### Wait for the node to be ready

```
watch kubectl get nodes
```

### Deploy a simple web app

This is a simple web app without any dependencies. Because this is a DaemonSet, there will be 3 instances running (one per node). AKS will create an Azure Load Balancer and a Public IP Address. The Azure Load Balancer sends traffic to each pod. Node that Azure Load Balancer keeps the TCP connection alive, so you will connect to the same node for up to 5 minutes if you hit refresh.

The source code for the web app is available here: <https://github.com/bartr/go-web-aks>  The Application Gateway walk through uses the same web app.

```
kubectl apply -f webapp

# Wait for service to start
watch kubectl get svc,pods

# Browse to public IP
```

### Execute commands on a running container

Like Docker, kubectl lets you execute commands on a running container, including an interactive shell.

```
# get the full pod name for one of the webapp pods
kubectl get pod

#replace webapp-xxxxx with the full pod name

# cat the log file
kubectl exec webapp-xxxxx -- cat logs/app.log

# run an interactive bash shell
kubectl exec -it webapp-xxxxx -- bash

```

### Node Resource Group

Notice that AKS added a Public IP and a Load Balancer after you deployed the webapp. The YAML in webapp/svc.yaml specifies "LoadBalancer" as the type of service.

![screenshot](images/aks-node-webapp.png)

### Create and deploy backend service (Redis)

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

Notice that AKS added a Public IP and reconfigured the Load Balancer after you deployed the frontend service. The YAML in frontend/svc.yaml specifies "LoadBalancer" as the type of service.

![screenshot](images/aks-node-frontend.png)

### Setting up Azure Applicaiton Gateway

This will setup an Azure Application Gateway with HTTPS support that points to a new app-gw service. The app-gw service uses an internal load balancer so the IP is only accessible from within the VNET. The template sets up automatic redirection of http to https on the Azure Application Gateway. This section is optional and takes about 45 minutes.

```
# create the app
kubectl apply -f app-gw

# wait for service / pod to start
kubectl get svc,pods

# setup port forwarding for the web app
kubectl port-forward svc/app-gw 8080:80 &

# test the web app
curl localhost:8080

# end port forwarding
fg
# press <ctl> c

# Create the app gateway subnet
MCVNET=`az network vnet list -g $MCRG --query '[0].[name]' -o tsv`
az network vnet subnet create --name app-gw-subnet --resource-group $MCRG --vnet-name $MCVNET --address-prefix 10.0.0.0/24

# change to the setup directory
cd app-gw/setup

# Edit app-gw.json

# change the vnet to
echo $MCVNET

# change backendIpAddress to:
kubectl get svc app-gw -o json | jq .status.loadBalancer.ingress[0].ip

# change the certificate data and password if you use a different cert
cat cert.pfx | base64 -w 0

# Deploy app-gw.json
az group deployment create --name app-gw-deployment -g $MCRG --template-file app-gw.json

# Check on the deployment
# Generally takes 15-30 minutes
az network application-gateway show -g $MCRG --name app-gw -o table

# get the app gateway public IP address
az network public-ip show -g $MCRG --name app-gw-ip --query [ipAddress] --output tsv

# Browse to http://app-gw-public-ip/
# you should be automatically redirected to https
# note that cert.pfx is a self-signed cert, so you will get a warning from your browser

```

### Final Resource Group

Notice that AKS added a Public IP and reconfigured the Load Balancer after you deployed the frontend service. The YAML in frontend/svc.yaml specifies "LoadBalancer" as the type of service.

![screenshot](images/aks-node-final.png)

### Helm Walk Through

There is an optional Helm walk through here: helm.md

### Clean up

```
# Delete resource groups
az group delete -y --no-wait -g $AKSRG
az group delete -y --no-wait -g MC_${AKSRG}_${AKSNAME}_${AKSLOC}
az group list -o table

# Only remove this file if this is the only k8s cluster in the file. Otherwise, edit the file and remove the key information

rm ~/.kube/config

```
