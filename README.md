https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

# az aks create -g k8s -n k8s  -c 1 -u bartr -s Standard_D2_v3 --nodepool-name west --no-wait

az aks create -g k8s -n k8s  -c 1 -u bartr -s Standard_A1_v2 --nodepool-name west --no-wait


az aks get-credentials -g k8s -n k8s
kubectl get nodes

kubectl apply -f backend
kubectl get service,pods

kubectl apply -f frontend
kubectl get service,pods

kubectl apply -f fe2
kubectl get service,pods


az aks scale -g k8s -n k8s -c 3

k get nodes,pods,service


