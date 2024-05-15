# Steps to install temporary vault instance

## Create cluster group in TMC
```
export CG=shared-services
ytt --data-value clusterGroupName=$CG -f ../tanzu-cli/cluster-group/cg-template.yml > generated/$CG-cluster-group.yaml
tanzu tmc cluster create -f generated/$CG-cluster-group.yaml
```