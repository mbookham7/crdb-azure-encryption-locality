# AKS - Multi Region cockroachDB

Description: Setting up and configuring a multi region cockroach cluster on Azure AKS
Tags: Azure

- Create a Resource Group for the project

    ```bash
    az group create --name $rg --location $loc1
    ```

- Networking configuration

    In order to enable VPC peering between the regions, the CIDR blocks of the VPCs must not overlap. This value cannot change once the cluster has been created, so be sure that your IP ranges do not overlap.

    - Create vnets for all Regions

        ```bash
        az network vnet create -g $rg -n crdb-$loc1 --location $loc1 --address-prefix $clus1_vnet_address_space \
            --subnet-name crdb-$loc1-sub1 --subnet-prefix $clus1_subnet
        ```


- Create the Kubernetes clusters in each region
    - To get SubnetID

        ```bash
        loc1subid=$(az network vnet subnet list --resource-group $rg --vnet-name crdb-$loc1 | jq -r '.[].id')
        ```

    - Create K8s Clusters in each region

        ```bash
        az aks create \
        --name $clus1 \
        --resource-group $rg \
        --network-plugin azure \
        --zones 1 2 3 \
        --vnet-subnet-id $loc1subid \
        --node-vm-size $vm_type \
        --node-count $n_nodes \
        --location $loc1
        ```

    - To Configure Kubectl context use

        ```bash
        az aks get-credentials --name $clus1 --resource-group $rg
        ```

[home](/README.md)