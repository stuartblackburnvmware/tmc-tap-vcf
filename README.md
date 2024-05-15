# Steps to install temporary vault instance
Hashicorp Vault will be installed on a k8s cluster using a bitnami helm chart.

## Prereqs
* TKGS is deployed with AVI
* TKGS namespace created to be used for this new cluster

## Tools
* Tanzu CLI
    * TMC plugin
    * Apps Plugin
* yq
* ytt

## Create cluster group in TMC
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cluster-group/cg-template.yaml | tanzu tmc clustergroup create -f-
```

## Create cluster in TMC
```
export PROFILE=shared-services
ytt --data-values-file tanzu-cli/values.yaml --data-value profile=$PROFILE -f tanzu-cli/clusters/cluster-template.yaml > generated/$PROFILE-cluster.yaml
tanzu tmc cluster create -f generated/$PROFILE-cluster.yaml
```