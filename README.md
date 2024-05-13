# CockroachDB in Kubernetes with an Encrypted Store and Pod Locality Enabled

In this repo we will deploy CockroachDB into Kubernetes with encrypted disks.

```
vm_type="Standard_D4_v2"
n_nodes=4
rg="mb-crdb-aks-encryption-region"
clus1="crdb-aks-encryption-uksouth"
clus1_vnet_address_space="10.1.0.0/16"
clus1_subnet="10.1.1.0/24"
loc1="uksouth"
```

1. [Setup Azure networking and Infrastructure.](/markdown/infra-setup.md)

2. [CockroachDB installation and configuration.](/markdown/crdb-setup.md)