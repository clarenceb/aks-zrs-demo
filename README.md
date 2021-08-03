AKS with ZRS disks
==================

ZRS managed disks is currently in preview and supported only in the West US 2, West Europe, North Europe, and France Central regions.

Demo
----

```sh
LOCATION=westus2
RGROUP=aks-storage
CLUSTER=cluster
VERSION=1.21.2
USER_POOL=npapps1

az feature register --namespace Microsoft.Compute --name SsdZrsManagedDisks
az feature show --namespace Microsoft.Compute --name SsdZrsManagedDisks
az provider register --namespace Microsoft.Compute

az group create --name $RGROUP --location $LOCATION

az aks create \
    --resource-group $RGROUP \
    --name $CLUSTER \
    --kubernetes-version $VERSION \
    --node-count 2 \
    --node-zones 1 2 \
    --node-vm-size Standard_DS2_v2 \
    --enable-addons monitoring \
    --generate-ssh-keys

az aks nodepool add \
    --resource-group $RGROUP \
    --name $USER_POOL \
    --cluster-name $CLUSTER \
    --node-count 2 \
    --node-zones 1 2 \
    --node-vm-size Standard_DS2_v2 \
    --mode User

az aks get-credentials --resource-group $RGROUP --name $CLUSTER --overwrite-existing
az aks nodepool list --resource-group $RGROUP --cluster-name $CLUSTER -o table
```

Check nodes are spread across AZs:

```sh
kubectl get node -o json | grep -e "hostname" -e "topology.kubernetes.io/zone"
```

Deploy test app:

```sh
kubectl apply -f azure-zone-sc-csi.yaml
kubectl get sc

kubectl apply -f pvc-zone-disk-csi.yaml

kubectl get pvc
#NAME                 STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS       AGE
#pvc-zone-azuredisk   Pending                                      managed-zone-csi   29s

kubectl apply -f workload.yaml

kubectl get pvc
#NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
#pvc-zone-azuredisk   Bound    pvc-ecd71a4f-9024-4820-8b87-9b939748ef37   10Gi       RWO            managed-zone-csi   4m53s

kubectl get pod -o wide
#NAME                            READY   STATUS    RESTARTS   AGE   IP           NODE                              NOMINATED NODE   READINESS GATES
#stateful-app-6b79894dfd-tm7bx   1/1     Running   0          88s   10.244.5.3   aks-npapps1-27520834-vmss000001   <none>           <none>
```

Check state is being writing to the disk:

```sh
POD_NAME="$(kubectl get pod --selector name=stateful-app -o name)"
kubectl exec -ti $POD_NAME -- /bin/bash
wc -l < /mnt/azuredisk/outfile
tail -f /mnt/azuredisk/outfile
# CTRL+C
exit
```

Continuously poll for file line count in another terminal:

```sh
POD_NAME="$(kubectl get pod --selector name=stateful-app -o name)"
watch kubectl exec -ti $POD_NAME -- wc -l /mnt/azuredisk/outfile
```


Delete the node with the pod running on it:

```sh
NODE_NAME="$(kubectl get pod --selector name=stateful-app -o jsonpath='{.items[0].spec.nodeName}')"
VMSS_NAME="$(az vmss list --query [].name -o tsv | grep $USER_POOL)"
NODE_RGROUP="$(az aks show -n $CLUSTER -g $RGROUP --query nodeResourceGroup -o tsv | tr '[a-z]' '[A-Z]')"
POD_INSTANCE_ID="$(kubectl get pod --selector name=stateful-app -o jsonpath='{.items[0].spec.nodeName}' | rev | cut -c 1-6 | rev | bc)"

echo $NODE_NAME, $VMSS_NAME $NODE_RGROUP, $POD_INSTANCE_ID

kubectl delete node $NODE_NAME
az vmss list-instances --name $VMSS_NAME --resource-group $NODE_RGROUP -o table
az vmss delete-instances --name $VMSS_NAME --resource-group $NODE_RGROUP --instance-ids $POD_INSTANCE_ID
```

Check that the pod moved to the other zone and the zrs disk is attached with existing state:

```sh
kubectl get node -o wide
kubectl get pod -o wide
POD_NAME="$(kubectl get pod --selector name=stateful-app -o name)"
kubectl describe $POD_NAME
```

Check state is being writing to the disk:

```sh
kubectl exec -ti $POD_NAME -- /bin/bash
wc -l < /mnt/azuredisk/outfile
tail -f /mnt/azuredisk/outfile
# CTRL+C
exit

Reset Demo
----------

```sh
az aks nodepool scale --name npapps1 --cluster-name $CLUSTER --resource-group $RGROUP --node-count 2
kubectl delete -f workloads.yaml
kubectl delete -f pvc-zone-disk-csi.yaml
```

Cleanup
-------

```sh
az group create --name $RGROUP
```

Issues
------

* Currently the process takes 8-9 minutes to move a pod from one node to another in case of unexpected node shutdown: https://github.com/kubernetes-sigs/azuredisk-csi-driver/tree/master/docs/known-issues/node-shutdown-recovery
* When pod is recreated after node deletion, the NODE name shows as `...-vmss000000` even if the actual node instance is some other ID, e.g. `...-vmss000003` (unconfirmed issue)
