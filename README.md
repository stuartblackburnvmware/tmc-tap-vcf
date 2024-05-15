# Steps to install temporary vault instance
Hashicorp Vault will be installed on a k8s cluster using a bitnami helm chart.

## Create cluster group in TMC
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cluster-group/cg-template.yml | tanzu tmc clustergroup create -f-
```

## Create cluster in TMC
```
export CLUSTER=shared-services
ytt --data-value cluster