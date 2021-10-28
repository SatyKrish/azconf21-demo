# Azure Community Conference 2021 - Demo
AKS Demo application for Azure Community Conference 2021

## Setup

1. Set environment defaults.

    ```sh
    RESOURCE_GROUP='azconf21demogroup'
    LOCATION=eastus2
    CLUSTER_NAME='azconf21democluster'
    ```

2. Create an AKS cluster with kubernetes version 1.21, CNI network plugin and managed identity enabled. 

    ```sh
    az group create \
        --name $RESOURCE_GROUP \
        --location $LOCATION
    
    az aks create \
        --resource-group $RESOURCE_GROUP \
        --name $CLUSTER_NAME \
        --enable-managed-identity \
        --network-plugin azure \
        --kubernetes-version 1.21.2
    ```

3. Generate kubeconfig file for connecting to AKS cluster.

    ```sh
    az aks get-credentials \
        --resource-group $RESOURCE_GROUP \
        --name $CLUSTER_NAME \
        --admin
    ```

4. Verify whether `csi-driver` pods are `running`.

    ```sh
    kubectl get pods -n kube-system
    ```

## Demo 1: Azure Disk as Persistent Volume

1. Deploy `voting-app`, which includes a python frontend service and redis backend service with `Azure Disk` as persistent volume.

    ```sh
    kubectl apply -f manifests/1-voting-app-deploy.yaml
    ```

    Watch the pods are in `running` state.

    ```sh
    watch kubectl get all -n voting-app
    ```

    Open `frontend` service external-ip in a browse for `voting-app` to load. Click the poll options few times to increase the count.

2. Backup `voting-app` persistent volume using CSI `volumesnapshot` function.

    ```sh
    kubectl apply -f manifests/2-voting-app-backup.yaml
    ```

    Review the `volumesnapshot` and `volumesnapshotcontent` to ensure snapshot is `ready`.

    ```sh
    kubectl get volumesnapshot,volumesnapshotcontent -n voting-app

    kubectl describe volumesnapshot <name> -n voting-app

    kubectl describe volumesnapshotcontent <name> -n voting-app
    ```

3. Restore `voting-app` persistent volume from CSI `volumesnapshot` function.

    ```sh
    kubectl apply -f manifests/3-voting-app-restore.yaml
    ```

    Watch the pods are in `running` state.

    ```sh
    watch kubectl get all -n voting-app
    ```

    Open `frontend` service external-ip in a browse for `voting-app` to load. Verify whether the poll count matches the count in #1.

## Demo 2: Azure Files as Persistent Volume

1. Create a general-purpose storage account and a file share with `1G1` storage capacity.

    ```sh
    STORAGE_ACCOUNT=`azcconf21demostorage`
    FILE_SHARE=`azcconf21demoshare`

    az storage account create \
        --resource-group $RESOURCE_GROUP \
        --name $STORAGE_ACCOUNT \
        --location $LOCATION \
        --encryption-services file

    az storage share create  \
        --name $FILE_SHARE \
        --account-name $STORAGE_ACCOUNT \
        --quota 1
    ```

    Get storage account resource id.

    ```sh
    STORAGE_RESOURCE_ID=$(az storage account show -g ${RESOURCE_GROUP} -n ${STORAGE_ACCOUNT} --query id -o tsv)
    ```

2. Assign kubelet identity with `Storage Account Key Operator Service Role` role scoped to the storage account.

    ```sh
    KUBELET_IDENTITY=$(az aks show -g ${RESOURCE_GROUP} -n ${CLUSTER_NAME} --query identityProfile.kubeletidentity.objectId -o tsv)

    az role assignment create \
        --assignee ${KUBELET_IDENTITY} \
        --role 'Storage Account Key Operator Service Role' \
        --scope ${STORAGE_RESOURCE_ID}
    ```

3. Deploy hello-world application with `Azure Files` persistent volume mounted using managed identity.

    ```sh
    kubectl apply -f manifests/4-hello-world-app-deploy.yaml
    ```

    Watch the pods are in `running` state.

    ```sh
    watch kubectl get all -n hello-world-app
    ```

4. Verify whether the persistent volume with read-write operations on both pods. Persistent volume is mounted at `/data` path.

    ```sh
    # echo "Hello world AKS-CSI from Pod 1 !" >> csi-test
    kubectl exec -it $(kubectl get pod -n hello-world-app -l app=hello-world-app -o jsonpath='{.items[0].metadata.name}') -n hello-world-app -- sh

    # echo "Hello world AKS-CSI from Pod 2 !" >> csi-test
    kubectl exec -it $(kubectl get pod -n hello-world-app -l app=hello-world-app -o jsonpath='{.items[1].metadata.name}') -n hello-world-app -- sh
    ```

## Cleanup

1. Delete the resources created in `voting-app` namespace. 

    ```sh
    kubectl delete namespace voting-app
    ```

2. Delete the resources created in `hello-world-app` namespace. 

    ```sh
    kubectl delete namespace hello-world-app
    ```

3. Delete resource group

    ```sh
    az group delete --name $RESOURCE_GROUP
    ```