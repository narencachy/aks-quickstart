# Time Saving

### Start some long running tasks in advance

Don't worry if you don't have a clue what these do. We'll get there ...

```

# You can copy and paste all of these at once

kubectl apply -f app-gw/svc.yaml

MCVNET=`az network vnet list -g $MCRG --query '[0].[name]' -o tsv` && echo $MCVNET
az network vnet subnet create --name app-gw-subnet --resource-group $MCRG --vnet-name $MCVNET --address-prefix 10.0.0.0/24

```

### Create Azure Container Registry

```

az acr create -g $ACRRG -n $ACR_NAME --sku Basic

```

### Finish some prep work

```

# wait for IP to be valid before next section
IP=`kubectl get svc app-gw -o json | jq .status.loadBalancer.ingress[0].ip` && echo $IP

```

```

# run all of these at once
cd ~/aks/app-gw/setup

# update app-gw.json
sed s/{{vnet}}/$MCVNET/ app-gw.json > v.json
sed s/{{ip}}/$IP/ v.json > app-gw-final.json
rm v.json

# Deploy app-gw.json
az group deployment create --name app-gw-deployment -g $MCRG --template-file app-gw-final.json --no-wait

```

```

# now run these
cd ~/aks
kubectl apply -f acrgoweb/svc.yaml
kubectl apply -f votes/svc.yaml
kubectl apply -f webapp/svc.yaml

```
