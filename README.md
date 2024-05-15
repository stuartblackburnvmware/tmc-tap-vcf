# TAP on VCF with TMC Guide
This guide's purpose is to quickly stand up TAP using TMC in a VCF environment. Multiple TAP clusters will be created for the various sites and profiles. A shared-services cluster will also be created to host services such as Vault, Harbor, etc.

## Prereqs
* TKGS is deployed with AVI
* TKGS namespaces created for each required cluster
* bitnamicharts project created on Harbor instance

## Tools
* Tanzu CLI
    * TMC plugin
    * Apps Plugin
* yq
* ytt

## Create cluster groups in TMC

This needs to be done for 2 cluster groups, each with a different cluster type
* shared-services
* tap-vcf
```
export CLUSTERGROUP=<cluster-type>
ytt --data-values-file tanzu-cli/values.yml --data-value clustergroup=$CLUSTERGROUP -f tanzu-cli/cluster-group/cg-template.yml | tanzu tmc clustergroup create -f-
```

## Create clusters
This needs to be done once for the shared-services cluster, and then one more time for each TAP profile you need
* shared-services
* run
* view
* build

```
export PROFILE=<profile-name>
ytt --data-values-file tanzu-cli/values.yml --data-value profile=$PROFILE -f tanzu-cli/clusters/cluster-template.yml > generated/$PROFILE-cluster.yaml
tanzu tmc cluster create -f generated/$PROFILE-cluster.yaml
```

## Enable helm and flux
This needs to be done once for each cluster group

The below commands enable flux at the cluster group level and install the source, helm, and kustomize controllers. These will be installed automatically on all clusters in this cluster group.

Enable at the cluster group level. This needs to be done once for each cluster group
* shared-services
* tap-vcf
```
export CLUSTERGROUP=<cluster-group>
tanzu tmc continuousdelivery enable -g $CLUSTERGROUP -s clustergroup
tanzu tmc helm enable -g $CLUSTERGROUP -s clustergroup
```

## Vault Installation
### Login to Newly Created Cluster
```
export CLUSTER=shared-services
export WCP=cluster01-wcp.stuart-lab.xyz
export CLUSTER_NAMESPACE=shared-services
kubectl vsphere login --tanzu-kubernetes-cluster-name $CLUSTER --server $WCP --tanzu-kubernetes-cluster-namespace $CLUSTER_NAMESPACE --insecure-skip-tls-verify
```

### Create vault namespace
```
kubectl create namespace vault
```

### Install vault air-gapped via helm
On machine with Internet access
```
helm dt wrap oci://docker.io/bitnamicharts/vault --version 0.4.6
```

Copy vault-0.4.6.wrap.tgz to machine in air-gapped environment and run the following:

```
export HARBOR=<harbor-fqdn>
helm dt unwrap vault-0.4.6.wrap.tgz oci://$HARBOR/bitnamicharts --insecure --yes
helm install vault oci://$HARBOR/bitnamicharts/vault -n vault --insecure-skip-tls-verify -f manifests/override-values.yaml
```

Initialize vault server. Make note of the unseal keys and initial root token.

kubectl exec --stdin=true --tty=true --namespace=vault vault-server-0 -- vault operator init


Unseal vault server. Repeat this command 3 times and using 3 different unseal keys

kubectl exec --stdin=true --tty=true --namespace=vault vault-server-0 -- vault operator unseal


Repeat the previous step for all unsealed vault-servers. This will vary depending on the number of replicas your environment contains.
Login to vault server:

export VAULT_ADDR="https://vault.h2o-2-21144.h2o.vmware.com/"
export VAULT_TOKEN="<from previous step>"
vault login -tls-skip-verify


Create secrets engine

vault secrets enable -path=tkg-secrets -tls-skip-verify kv


Add/read test secret to newly created secrets engine

vault kv put -tls-skip-verify tkg-secrets/test-secret keyname=keyvalue
vault kv list -tls-skip-verify tkg-secrets
vault kv get -tls-skip-verify tkg-secrets/test-secret