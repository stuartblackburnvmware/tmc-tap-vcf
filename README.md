# Guide to Install TAP Clusters via TMC
TAP clusters will be rolled out using TMC. This guide details the steps involved in that process.

## Prereqs
* TKGS is deployed with AVI
* TKGS namespace created to be used for the new clusters

## Tools
* Tanzu CLI
    * TMC plugin
    * Apps Plugin
* yq
* ytt

## Prep values file
Copy the values template file and prep modify the values as needed
```
cp tanzu-cli/values/values-template.yml tanzu-cli/values/values.yml
```

## Create cluster group in TMC
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cluster-group/cg-template.yml | tanzu tmc clustergroup create -f-
```

## Create cluster in TMC
Replace PROFILE with the name of the TAP cluster as needed
```
export PROFILE=site1-tap-run
ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE -f tanzu-cli/clusters/cluster-template.yml > generated/$PROFILE-cluster.yml
tanzu tmc cluster create -f generated/$PROFILE-cluster.yml
```

## Enable helm and flux on Cluster Group
Replace CLUSTERGROUP with the name of the cluster group as needed
```
export CLUSTERGROUP=tap-vcf
tanzu tmc continuousdelivery enable -s clustergroup -g $CLUSTERGROUP
tanzu tmc helm enable -g $CLUSTERGROUP -s clustergroup
```

## Bootstrap cluster with gitops
This step configures the cluster group to use this git repo as the source for flux, specifically the flux folder. The gitops setup is done at the cluster group level so we don't need to individually bootstrap every cluster.

Before creating the TMC objects, you will need to rename the folders in flux/clusters to match your cluster names. Also if your cluster group name is different than tap-vcf you will need to rename the folder in flux/clustergroups along with the paths in the flux/clustergroups/<group-name>/base.yml.

### Create the gitrepo in TMC

This step is done manually for now since the current version of TMC-SM doesn't support everything we need. TODO: Automate this

Under your cluster group->Add-ons->Repository Credentials, create an SSH key repository credential

Note: Known hosts can be retrieved by running a `ssh-keyscan <git-server>` against your git server

Under your cluster group->Add-ons->Git repositories, add a Git repository pointing to your repo. Select the repository credential created in the previous step.

Once this is automated, it could be added with a command similar to the following:
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/git-repo-template.yml > generated/gitrepo.yml
tanzu tmc continuousdelivery gitrepository create -f generated/gitrepo.yml -s clustergroup
```

### Create kustomizations
Create the base kustomization that will bootstrap the clusters and setup any initial infra. Kustomizations are split into pre and post in order to support deploying items which are dependent on other items.
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/kust-template.yml > generated/kust.yml
tanzu tmc continuousdelivery kustomization create -f generated/kust.yml -s clustergroup
```

At this point clusters should start syncing in multiple kustomizations. You can check their status using the below command. there will be some in a failed state until the TAP install is done.

```
kubectl get kustomizations -A
```