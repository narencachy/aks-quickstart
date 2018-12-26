https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

# az aks create -g k8s -n k8s  -c 1 -u bartr -s Standard_D2_v3 --nodepool-name west --no-wait

az group create -l westus2 -g k8s

az aks create -g k8s -n k8s  -c 1 -u bartr -s Standard_A1_v2 --nodepool-name west --no-wait


az aks get-credentials -g k8s -n k8s
kubectl get nodes

kubectl apply -f backend
kubectl get service,pods

kubectl apply -f frontend
kubectl get service,pods

k port-forward --namespace default svc/backend 6379:6379
ctl-z
bg
fg
ctl-c


kubectl apply -f fe2
kubectl get service,pods


az aks scale --no-wait -g k8s -n k8s -c 3

k get nodes,pods,service

## Instal Helm
sudo snap install helm --classic

kubectl apply -f helm/rbac-helm.yaml

helm init --service-account tiller

helm --upgrade

helm install stable/redis


az group delete -y --no-wait -g k8s
az group delete -y --no-wait -g MC_k8s_k8s_westus2
