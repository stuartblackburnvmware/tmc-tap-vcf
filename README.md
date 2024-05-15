# Guide to Install k8s Vault Instance
Hashicorp Vault will be installed on a k8s cluster using a bitnami helm chart.

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
ytt --data-values-file tanzu-cli/values.yaml -f tanzu-cli/cluster-group/cg-template.yaml | tanzu tmc clustergroup create -f-
```

## Create cluster in TMC
```
export PROFILE=shared-services
ytt --data-values-file tanzu-cli/values.yaml --data-value profile=$PROFILE -f tanzu-cli/clusters/cluster-template.yaml > generated/$PROFILE-cluster.yaml
tanzu tmc cluster create -f generated/$PROFILE-cluster.yaml
```

## Enable Continuous Delivery on Cluster Group

```
export CLUSTERGROUP=<clustergroup-name>
tanzu tmc continuousdelivery enable -s clustergroup -g $CLUSTERGROUP
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
helm install vault oci://$HARBOR/bitnamicharts/vault -n vault --insecure-skip-tls-verify -f override-values.yaml
```