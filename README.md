# Steps to install temporary vault instance

## Create cluster group in TMC
```
export CG=shared-services
ytt --data-value clusterGroupName=$CG -f tanzu-cli/cluster-group/cg-template.yaml > generated/$CG-cluster-group.yaml
tanzu tmc clustergroup create -f generated/$CG-cluster-group.yaml
```