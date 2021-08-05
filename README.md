AKS with multi-zone ZRS disk demo
=================================

Summary of demo:

* This demo will create an AKS cluster in 2 zones with a node in each zone
* A ZRS disk is created via a PVC and attached to a pod on a node in one zone.
* When the node is deleted (simulating a node or zone failure), the pod will be rescheduled to a node in the 2nd zone.
* The ZRS disk will be reattached correctly and the state is retained.

Note: ZRS managed disks is currently in preview and supported only in the West US 2, West Europe, North Europe, and France Central regions.

Demo
----

```sh
LOCATION=westus2
RGROUP=aks-storage
CLUSTER=cluster
# Only use 1.21 if the VHD contains mcr.microsoft.com/oss/kubernetes-csi/azuredisk-csi:v1.5.0 or higher
# otherwise, self-install the CSI driver on 1.20 (see below for issue details).
# VERSION=1.21.2
VERSION=1.20.7
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

If you using AKS 1.20 or lower, you need to self-install the [Azure Disk CSI driver](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/docs/install-driver-on-aks.md) and follow the steps below:

```sh
# Install by kubectl
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/v1.5.0/deploy/install-driver.sh | bash -s v1.5.0 --
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/storageclass-azuredisk-csi.yaml

# Get the AKS Kubelet identity
objectId="$(az aks show -n cluster -g aks-storage --query identityProfile.kubeletidentity.objectId -o tsv)"

# Assign Contributor role to the AKS Kubelet identity on the MC_ (node) resource group
nodeResourceGroup="$(az aks show -n cluster -g aks-storage --query nodeResourceGroup -o tsv)"
subscriptionId="$(az account show --query id -o tsv)"
nodeResourceGroupId="/subscriptions/$subscriptionId/resourcegroups/$nodeResourceGroup"

az role assignment create --assignee $objectId --scope $nodeResourceGroupId --role "Contributor"

# Check pod status
kubectl -n kube-system get pod -o wide --watch -l app=csi-azuredisk-controller
kubectl -n kube-system get pod -o wide --watch -l app=csi-azuredisk-node
```

Deploy test app:

```sh
# Create the storage class for ZRS
kubectl apply -f azure-zone-sc-csi.yaml
kubectl get sc

# Create the PVC using the new storage class
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
kubectl exec -ti $POD_NAME -- /bin/sh
wc -l < /mnt/azuredisk/outfile
tail -f /mnt/azuredisk/outfile
# CTRL+C
exit
```

(Optional) Continuously poll for file line count in another terminal:

```sh
POD_NAME="$(kubectl get pod --selector name=stateful-app -o name)"
watch kubectl exec -ti $POD_NAME -- wc -l /mnt/azuredisk/outfile
```

Delete the node with the pod running on it:

```sh
NODE_NAME="$(kubectl get pod --selector name=stateful-app -o jsonpath='{.items[0].spec.nodeName}')"
VMSS_NAME="$(az vmss list --query [].name -o tsv | grep $USER_POOL)"
NODE_RGROUP="$(az aks show -n $CLUSTER -g $RGROUP --query nodeResourceGroup -o tsv | tr '[a-z]' '[A-Z]')"
POD_INSTANCE_ID="$(kubectl get pod --selector name=stateful-app -o jsonpath='{.items[0].spec.nodeName}' | rev | cut -c 1-6 | rev | sed 's/^0*//')"

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
```

Reset Demo
----------

```sh
az aks nodepool scale --name npapps1 --cluster-name $CLUSTER --resource-group $RGROUP --node-count 2
kubectl delete -f workload.yaml
kubectl delete -f pvc-zone-disk-csi.yaml
kubectl delete -f azure-zone-sc-csi.yaml

# If you self-installed the Azure Disk CSI driver:
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/v1.5.0/deploy/uninstall-driver.sh | bash -s v1.5.0 --
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/storageclass-azuredisk-csi.yaml
```

Cleanup
-------

```sh
az group delete --name $RGROUP
```

Issues
------

* Currently the process takes 8-9 minutes to move a pod from one node to another in case of unexpected node shutdown: https://github.com/kubernetes-sigs/azuredisk-csi-driver/tree/master/docs/known-issues/node-shutdown-recovery
* (UPDATED 2021/8/5) When pod is recreated after node deletion it stays in pending with this scheduling error:

```sh
kubectl describe pod stateful-app-67cdd6786d-q8dxx

Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  25s   default-scheduler  0/3 nodes are available: 1 node(s) had volume node affinity conflict, 2 node(s) didn't match Pod's node affinity/selector.
  Warning  FailedScheduling  23s   default-scheduler  0/3 nodes are available: 1 node(s) had volume node affinity conflict, 2 node(s) didn't match Pod's node affinity/selector.
```

The "1 node(s) had volume node affinity conflict" error appears to be related to the PV having these automatically assigned nodeAffinity nodeSelectorTerms:

```sh
kubectl get pv pvc-5fcfecce-9793-4152-a73f-c0a146875f5c -o yaml

...
nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.disk.csi.azure.com/zone
          operator: In
          values:
          - westus2-2
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - westus2-2


kubectl get pod -o wide
NAME                            READY   STATUS    RESTARTS   AGE     IP       NODE     NOMINATED NODE   READINESS GATES
stateful-app-67cdd6786d-q8dxx   0/1     Pending   0          3m25s   <none>   <none>   <none>           <none>

kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-27520834-vmss000000   Ready    agent   25h   v1.21.2
aks-nodepool1-27520834-vmss000001   Ready    agent   25h   v1.21.2
aks-npapps1-27520834-vmss000009     Ready    agent   34m   v1.21.2

kubectl describe node aks-npapps1-27520834-vmss000009

Labels:             agentpool=npapps1
...
                    failure-domain.beta.kubernetes.io/zone=westus2-1
...
```

So the node in in `westus2-1` but the PV is restricted to `westus2-2`:

```sh
Node Affinity:
  Required Terms:
    Term 0:        topology.disk.csi.azure.com/zone in [westus2-2]
                   topology.kubernetes.io/zone in [westus2-2]
```

### UPDATE 2021/8/5

This issue has been resolved in [fix: ZRS node affinity setting](https://github.com/kubernetes-sigs/azuredisk-csi-driver/pull/906).

Check which version of the azuredisk image you have:

```sh
kubectl get daemonset -n kube-system csi-azuredisk-node -o json | jq -r '.spec.template.spec.containers[] | select(.name=="azuredisk") | .image'
# ==> mcr.microsoft.com/oss/kubernetes-csi/azuredisk-csi:v1.4.0
```

If it is < 1.5.0 then check if a new [VHD has been released](https://github.com/Azure/AKS/tree/master/vhd-notes/aks-ubuntu/AKSUbuntu-1804) containing the azure disk csi 1.5.0+ then perform a [node image upgrade](https://docs.microsoft.com/en-us/azure/aks/node-image-upgrade#upgrade-a-specific-node-pool)).

Alternatively, create a AKS cluster on <=1.20.x and self-install the CSI driver as per the instructions here: https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/docs/install-driver-on-aks.md

Don't do this on AKS 1.21+ or there may be conflicts between the managed Azure Disk CSI driver and the self-installed version.
