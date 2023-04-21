AKS with multi-zone ZRS disk demo
=================================

Summary of demo:

* This demo will create an AKS cluster in 3 zones with a node in each zone
* A ZRS disk is created via a PVC and attached to a pod on a node in one zone.
* When the node is deleted (simulating a node or zone failure), the pod will be rescheduled to a node in the 2nd zone.
* The ZRS disk will be reattached correctly and the state is retained.

Note:

* ZRS managed disks is currently supported only in limited regions, refer to the [zrs disk limitations](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-deploy-zrs?#limitations) page for the latest information.
* Testing in other regions may require whitelisting your subscription

Demo
----

Create a zone-redundant cluster with a User node pool for our test workload:

```sh
LOCATION=australiaeast
RGROUP=zrs-demo
CLUSTER=zrs-cluster
AKS_LATEST_VERSION=$(az aks get-versions -l $LOCATION --query orchestrators[*].orchestratorVersion -o tsv | sort -nr | head -n1)
USER_POOL=npapps1

az feature register --namespace Microsoft.Compute --name SsdZrsManagedDisks
az feature show --namespace Microsoft.Compute --name SsdZrsManagedDisks
az provider register --namespace Microsoft.Compute

az group create --name $RGROUP --location $LOCATION

az aks create \
    --resource-group $RGROUP \
    --name $CLUSTER \
    --kubernetes-version $AKS_LATEST_VERSION \
    --node-count 2 \
    --zones 1 2 3 \
    --node-vm-size Standard_DS2_v2 \
    --enable-addons monitoring \
    --network-plugin azure \
    --generate-ssh-keys

az aks nodepool add \
    --resource-group $RGROUP \
    --name $USER_POOL \
    --cluster-name $CLUSTER \
    --node-count 3 \
    --zones 1 2 3 \
    --node-vm-size Standard_DS2_v2 \
    --mode User

az aks get-credentials --resource-group $RGROUP --name $CLUSTER --overwrite-existing
az aks nodepool list --resource-group $RGROUP --cluster-name $CLUSTER -o table
az aks install-cli

# Taint system node pool so thatuser workloads can only be scheduled on the User node pool
kubectl taint nodes -l kubernetes.azure.com/mode=system CriticalAddonsOnly=true:NoSchedule
```

Check that the nodes are spread across the chosen AZs:

```sh
kubectl get node -o json | grep -e "hostname" -e "topology.kubernetes.io/zone"
```

Sample response:

```sh
"kubernetes.io/hostname": "aks-nodepool1-79989030-vmss000000",
"topology.kubernetes.io/zone": "australiaeast-1"
"kubernetes.io/hostname": "aks-nodepool1-79989030-vmss000001",
"topology.kubernetes.io/zone": "australiaeast-2"
"kubernetes.io/hostname": "aks-npapps1-38150515-vmss000000",
"topology.kubernetes.io/zone": "australiaeast-1"
"kubernetes.io/hostname": "aks-npapps1-38150515-vmss000001",
"topology.kubernetes.io/zone": "australiaeast-2"
"kubernetes.io/hostname": "aks-npapps1-38150515-vmss000002",
"topology.kubernetes.io/zone": "australiaeast-3"
```

As expected, the 2 system nodes are spread across 2 AZs and the 3 user nodes are spread across 3 AZs.

Deploy a stateful test app:

```sh
# Create the storage class for ZRS disks
kubectl apply -f managed-zrs-csi-premium.sc.yaml
kubectl get storageclass

# Create the PVC using the new storage class
kubectl apply -f pvc-zrs-disk-csi.yaml

kubectl get pvc
# NAME                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS              AGE
# pvc-zrs-azuredisk   Pending                                      managed-zrs-csi-premium   16s

kubectl apply -f stateful-app.yaml

kubectl get pvc
# NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
# pvc-zrs-azuredisk   Bound    pvc-e051053b-1ab3-4d56-aacd-9343652841af   10Gi       RWO            managed-zrs-csi-premium   5m11s

kubectl get pod -o wide
# NAME                            READY   STATUS    RESTARTS   AGE   IP             NODE                              NOMINATED NODE   READINESS GATES
# stateful-app-7b6f769d78-pk2xr   1/1     Running   0          34s   10.224.0.124   aks-npapps1-38150515-vmss000002   <none>           <none>
```

Check that the app state is being writing to the zrs disk:

```sh
pod_name() {
  echo "$(kubectl get pod --selector name=stateful-app -o name)"
}
kubectl exec -ti $(pod_name) -- /bin/sh
wc -l < /mnt/azuredisk/outfile
tail -f /mnt/azuredisk/outfile
# CTRL+C
exit
```

(Optional - in another terminal) Continuously poll for file line count (count should continue incrementing):

```sh
pod_name() {
  echo "$(kubectl get pod --selector name=stateful-app -o name)"
}
watch kubectl exec -ti $(pod_name) -- wc -l /mnt/azuredisk/outfile
```

(Optional - in another terminal) Continuously poll the running stateful pod to see its status and node it is running on:

```sh
watch kubectl get pod -o wide
```

Delete the node with the pod running on it:

```sh
NODE_NAME="$(kubectl get pod --selector name=stateful-app -o jsonpath='{.items[0].spec.nodeName}')"
kubectl delete node $NODE_NAME
```

It will take a few mins for node and underlying VM to get deleted.
The pod will be moved one of the remaining nodes in another zone and will show status "ContainerCreating".
After a few mins , the pod status should change to "Running" when the ZRS disk has properly detached from the deleted node
and re-attached to the new node:

```sh
kubectl get node -o wide
kubectl get pod -o wide
kubectl describe $(pod_name)

# Whilst the pod is in "ContainerCreating", you can see the following "Multi-Attach error for volume" warning in the pod events:
# Events:
#   Type     Reason                  Age                  From                     Message
#   ----     ------                  ----                 ----                     -------
#   Normal   Scheduled               8m1s                 default-scheduler        Successfully assigned default/stateful-app-7b6f769d78-zcm8f to aks-npapps1-38150515-vmss000003
#   Warning  FailedAttachVolume      8m1s                 attachdetach-controller  Multi-Attach error for volume "pvc-e051053b-1ab3-4d56-aacd-9343652841af" Volume is already exclusively attached to one node and can't be attached to another
#   Warning  FailedMount             82s (x3 over 5m58s)  kubelet                  Unable to attach or mount volumes: unmounted volumes=[azuredisk01], unattached volumes=[azuredisk01 kube-api-access-dpwd4]: timed out waiting for the condition
#   Normal   SuccessfulAttachVolume  59s                  attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-e051053b-1ab3-4d56-aacd-9343652841af"
#   Normal   Pulling                 57s                  kubelet                  Pulling image "mcr.microsoft.com/cbl-mariner/base/core:2.0"
#   Normal   Pulled                  55s                  kubelet                  Successfully pulled image "mcr.microsoft.com/cbl-mariner/base/core:2.0" in 2.25098902s (2.25099482s including waiting)
#   Normal   Created                 55s                  kubelet                  Created container mariner-azuredisk
#   Normal   Started                 55s                  kubelet                  Started container mariner-azuredisk
```

Check state is being writing to the disk:

```sh
kubectl exec -ti $(pod_name) -- /bin/bash
wc -l < /mnt/azuredisk/outfile
tail -f /mnt/azuredisk/outfile
# CTRL+C
exit
```

Reset Demo
----------

```sh
az aks nodepool scale --name npapps1 --cluster-name $CLUSTER --resource-group $RGROUP --node-count 3
kubectl delete -f stateful-app.yaml
kubectl delete -f pvc-zrs-disk-csi.yaml
kubectl delete -f managed-zrs-csi-premium.sc.yaml
```

Cleanup
-------

```sh
az group delete --name $RGROUP
```

Issues
------

* Currently the process takes 8 minutes to move a pod from one node to another in case of unexpected node shutdown: https://github.com/kubernetes-sigs/azuredisk-csi-driver/tree/master/docs/known-issues/node-shutdown-recovery
  * See possible [workaround](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/docs/known-issues/node-shutdown-recovery/statefulset-azuredisk.yaml) to reduce the reconcillation time
