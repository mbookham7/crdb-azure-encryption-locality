# Step: Deploy CockroachDB

Description:

## Prepare Kubernetes

Retrieve the IP addresses of the LB in each region and add these to the config maps.

```
kubectl create namespace $loc1 --context $clus1
```

## CockroachDB Deployment

Kubernetes is all prepared now for our deployment of CockroachDB. Now we must complete all the steps required to setup CockroachDB.
First we create two new folders for our certificates.

```
mkdir certs my-safe-directory keys
```

All communication between the node and clients is secure. The simplest way to achieve this is with the built in certificate authority. We can use any machine with the cockroach binary installed to create all the required certificates. Create the CA certificate and key pair:

```
cockroach cert create-ca \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

Create a client certificate and key pair for the root user:

```
cockroach cert create-client \
root \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

For all 3 regions, upload the client certificate and key to the Kubernetes cluster as a secret.

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--context $clus1 \
--namespace $loc1
```

Create the certificate and key pair for your CockroachDB nodes in one region:

```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$loc1 \
cockroachdb-public.$loc1.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$loc1" \
"*.cockroachdb.$loc1.svc.cluster.local" \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

Upload the node certificate and key to the Kubernetes cluster as a secret, specifying the appropriate context and namespace:

```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--context $clus1 \
--namespace $loc1
```

Encryption
```
cockroach gen encryption-key -s 128 keys/aes-128.key
```

```
kubectl create secret \
generic cockroachdb.key \
--from-file=keys \
--context $clus1 \
--namespace $loc1
```


Apply the provided Kubernetes manifests into each AKS cluster. These contain several different resources including the statefulSet. In this file you will need to make a couple of edits. The first of these is to add the correct namespace name for each region to the StatefulSet config for each region. See the example below:

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cockroachdb
  # TODO: Use this field to specify a namespace other than "default" in which to deploy CockroachDB (e.g., us-east-1).
  namespace: northeurope
spec:
```

Next is to update the join command that is ran when the CockroachDB binary is started. This join command must contain the DNS name of all of the pods in the cluster. This is used when new pods join the CockroachDB cluster.
See an example below:

```
--join cockroachdb-0.cockroachdb.uksouth,cockroachdb-1.cockroachdb.uksouth,cockroachdb-2.cockroachdb.uksouth,cockroachdb-0.cockroachdb.ukwest,cockroachdb-1.cockroachdb.ukwest,cockroachdb-2.cockroachdb.ukwest,cockroachdb-0.cockroachdb.northeurpoe,cockroachdb-1.cockroachdb.northeurope,cockroachdb-2.cockroachdb.northeurope
```

Once you have updated the three files as required you can apply these to the three AKS clusters.

```
kubectl -n $loc1 apply -f ./manifest/uksouth-cockroachdb-statefulset-secure.yaml --context $clus1
```

Check to see if you pods are running. They should all be running but not ready.

```
kubectl get pods --context $clus1 --namespace $loc1
```

If they are all running we then need to initialize the cluster. This is done by accessing one of the CockroachDB pods and running `cockroach init`.

```
kubectl exec \
--context $clus1 \
--namespace $loc1 \
-it cockroachdb-0 \
-- /cockroach/cockroach init \
--certs-dir=/cockroach/cockroach-certs
```

Check the pods again, they should all now be ready.

```
kubectl get pods --context $clus1 --namespace $loc1
```

Now create a pod with a secure connection to the cluster and we can configure the cluster settings. Create the pod:

```
kubectl config use-context $clus1
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $loc1
```

Connect to the pod.

```
kubectl exec -it cockroachdb-client-secure -n $loc1 -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```

Create and user and make it an admin.

```
CREATE USER craig WITH PASSWORD 'cockroach';
GRANT admin TO craig;
```

```
SET CLUSTER SETTING cluster.organization = '';
SET CLUSTER SETTING enterprise.license = '';
```

Expose the Admin UI externally.

```
kubectl apply -f ./manifest/admin-ui.yaml --context $clus1 --namespace $loc1
```

Check the Service has an IP address and test access to the UI.

```
kubectl get svc --context $clus1 --namespace $loc1
```

Congratulations! You should now have a working cluster.....


## CleanUp

Delete you your certs and Azure resources.

```
rm -R certs my-safe-directory keys
az group delete --name $rg
```

[home](/README.md)