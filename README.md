# Guide to Install Shared Services Cluster
A k8s cluster will be installed to run shared services such as Hashicorp Vault, Harbor, etc.

## Prereqs
* TKGS is deployed with AVI
* TKGS namespace created to be used for this new cluster
* bitnamicharts project created on Harbor instance

## Tools
* Tanzu CLI
    * TMC plugin
    * Apps Plugin
* yq
* ytt

## Create cluster group in TMC
```
ytt --data-values-file tanzu-cli/values.yml -f tanzu-cli/cluster-group/cg-template.yml | tanzu tmc clustergroup create -f-
```

## Create cluster in TMC
```
export PROFILE=shared-services2
ytt --data-values-file tanzu-cli/values.yml --data-value profile=$PROFILE -f tanzu-cli/clusters/cluster-template.yml > generated/$PROFILE-cluster.yml
tanzu tmc cluster create -f generated/$PROFILE-cluster.yml
```

## Enable helm and flux on Cluster Group

```
export CLUSTERGROUP=<clustergroup-name>
tanzu tmc continuousdelivery enable -s clustergroup -g $CLUSTERGROUP
tanzu tmc helm enable -g $CLUSTERGROUP -s clustergroup
```

## Bootstrap cluster with gitops
This step configures the cluster group to use this git repo as the source for flux, specifically the flux folder. The gitops setup is done at the cluster group level so we don't need to individually bootstrap every cluster.

Before creating the TMC objects, you will need to rename the folders in flux/clusters to match your cluster names. Also if your cluster group name is different than shared-services you will need to rename the folder in flux/clustergroups along with the paths in the flux/clustergroups/<group-name>/base.yml.

Create the gitrepo in TMC
```
ytt --data-values-file tanzu-cli/values.yml -f tanzu-cli/cd/git-repo-template.yml > generated/gitrepo.yml
tanzu tmc continuousdelivery gitrepository create -f generated/gitrepo.yml -s clustergroup
```

Create the base kustomization that will bootstrap the clusters and setup any initial infra.
```
ytt --data-values-file tanzu-cli/values.yml -f tanzu-cli/cd/kust-template.yml > generated/kust.yml
tanzu tmc continuousdelivery kustomization create -f generated/kust.yml -s clustergroup
```

At this point clusters should start syncing in multiple kustomizations. You can check their status using the below command. there will be some in a failed state until the TAP install is done.

```
kubectl get kustomizations -A
```




## Vault Installation
### Login to Newly Created Cluster
```
export CLUSTER=shared-services2
export WCP=cluster01-wcp.stuart-lab.xyz
export CLUSTER_NAMESPACE=shared-services2
kubectl vsphere login --tanzu-kubernetes-cluster-name $CLUSTER --server $WCP --tanzu-kubernetes-cluster-namespace $CLUSTER_NAMESPACE --insecure-skip-tls-verify
```

### Create vault namespace
```
kubectl create namespace vault
```

### Install vault air-gapped via helm
On machine with Internet access. Replace VAULTVERSION with the desired version. 1.2.1 used in this example.
```
export VAULTVERSION=1.2.1
helm dt wrap oci://docker.io/bitnamicharts/vault --version $VAULTVERSION
```

Copy vault-1.2.1.wrap.tgz to machine in air-gapped environment and run the following:

```
export HARBOR=<harbor-fqdn>
export VAULTVERSION=1.2.1
helm dt unwrap vault-$VAULTVERSION.wrap.tgz oci://$HARBOR/bitnamicharts --insecure --yes
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