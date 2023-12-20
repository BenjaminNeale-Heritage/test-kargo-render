
# Overview
This repository contains configuration to enable platform Infrastructure as Code (IaC) automation to build and configure OpenShift Container Platform 4 (OCP4) clusters.

The OCP4 clusters are managed through a highly opinionated GitOps approach.  Kustomize (https://kustomize.io/) is used to build the resource definitions which are applied to each OCP4 cluster.  Argo CD (https://argoproj.github.io/cd/) is deployed on each OCP4 cluster and monitors this repository for changes to configuration for its cluster.

The DRY (Don't Repeat Yourself) principle should be followed for GitOps configuration to minimise duplicated values.

The Rendered Manifest Pattern is used to make it very clear what is being deployed to each cluster/environment.  Changes should be made to the "main" branch following trunk based development.  The rendered manifests are kept in "cluster/{CLUSTER_NAME}" branches.  These branches are ONLY updated by GitOps tooling.

## Cluster Types

### Workload Types
* Application (app) clusters run business workloads.
* Hub clusters centralise and run management, monitoring, and security capabilities.

### Environment Types
* Production clusters run production workloads.
* Non-production environment (NPE) clusters run are used for development, testing, and support of production business workloads.
* Lab clusters are sandbox areas used by technology teams to configure and test cluster operations (e.g. provisioning, bootstrapping, configuring, upgrading, etc).

## Cluster Lifecycle

### Provision
A cluster needs to be provisioned before Argo CD can be installed to manage the cluster via the GitOps approach.  This step involves allocating all necessary infrastructure.  Provisioning is completed by the creation of a 'blank' OCP4 cluster.

### Bootstrap
A 'blank' cluster must then be configured before it is useful.  Bootstrapping gets the cluster to the point where the OpenShift GitOps Operator is installed and the cluster instance of Argo CD is running.  Once this happens then Argo CD is able to take over and complete the configuration of the cluster.  Bootstrapping is completed once a cluster is fully configured and ready to be used.

The README in the "bootstrap/overlays/_{CLUSTER_NAME}_/" directory will give details on how to bootstrap each individual cluster.

### Manage
Once a cluster has been bootstrapped then it will be managed through GitOps using Argo CD.

# GitOps Config

## Main Directory Structure
This is the directory structure under the "main" branch.

### Overview
```
cluster-config/
├─ bootstrap/
│  ├─ {CLUSTER_NAME}/
│  │  ├─ {CAPABILITY_NAME}/
│  │  └─ ...
│  └─ ...
├─ clusters/
│  └─ overlays/
│     ├─ {CLUSTER_NAME}/       <- Kustomize is run on each sub-directory in "clusters/overlays/{CLUSTER_NAME}/" directory to generate the contents of each "cluster/{CLUSTER_NAME}" branch.
│     │  ├─ {COMPONENT_NAME}/  <- The bootstrap Kustomization file in a "bootstrap/{CLUSTER_NAME}/{CAPABILITY_NAME}/" directory MUST point to one or more corresponding "clusters/overlays/{CLUSTER_NAME}/{COMPONENT_NAME}/" directories.
│     │  └─ ...
│     └─ ...
├ components/
│  ├─ {COMPONENT_NAME}/
│  │  ├─ base/                 <- The overlay Kustomization file in a "clusters/overlays/{CLUSTER_NAME}/{COMPONENT_NAME}/" directory MUST point to the corresponding "components/{COMPONENT_NAME}/base/" directory.
│  │  └─ variants/
│  │     ├─ {VARIANT_NAME}/    <- The overlay Kustomization file in a "clusters/overlays/{CLUSTER_NAME}/{COMPONENT_NAME}/" directory may point to one or more relevant "components/{COMPONENT_NAME}/variants/{VARIANT_NAME}/" directories.
│  │     └─ ...
│  └─ ...
└ namespaces/
   └─ overlays/
      ├─ {CLUSTER_NAME}/
      │  ├─ {NAMESPACE_NAME}/
      │  └─ ...
      └─ ...
```

### Explanation

#### Structure
* The "bootstrap" root directory contains configuration to bootstrap GitOps on each cluster.  This is grouped by capability which may bundle one or more components together during bootstrapping.
* The "clusters" root directory contains the configuration for all clusters after they have been provisioned and bootstrapped.  Some '__overlays__' directories are located here.
* The "components" root directory contains the configuration for each component that is needed for a cluster.  The '__base__' and '__variants__' directories are located here.
* The "namespaces" root directory contains the configuration for all __tenant__ namespaces on a cluster.  This is where you define cluster level resources (namespaces, quotas, limitranges, operators to install, etc) that are needed by the teams to do their work but typically require cluster-admin rights to provision.  Some '__overlays__' directories are located here.
* Configuration common to all clusters should be put into the "components/{COMPONENT_NAME}/base/" directory.
* Configuration common to a clearly defined group of clusters (e.g. production) but not all clusters should be put into the "components/{COMPONENT_NAME}/variants/{VARIANT_NAME}/" directory (e.g. "prod"). NOTE - All "kustomize.yml" files must be "Component" type rather than "Kustomization" type under the "variants" directory.
* Configuration unique to a single cluster should be put into the "clusters/overlays/{CLUSTER_NAME}/" directory.  Each of these sub-directories will be the starting point for rendering manifests into the respective "cluster/{CLUSTER_NAME}" branch (refer below).
* Kustomize in each overlay directory should include configuration from the base and variant directories under the associated "components/{COMPONENT_NAME}/" directory whenever possible to minimise the configuration that needs to be put into each overlay directory.  It is possible for a cluster to use more than one variant.  This means that many overlay directories will only contain a 'kustomization.yml' file because the resources and patches will be contained in the base and variant directories under the associated "components/{COMPONENT_NAME}/" directory.
* Fields in the 'base' resource file that must be patched within an overlay or variant should be included in the 'base' resource file with the value set to "patchme" (or a "patchme" comment for arrays and objects).  Optional fields that can be patched should not be put into the 'base' resource file though.  This will provide some documentation within the 'base' resource file for fields that must be set when creating new overlays/variants.

#### File Naming
* The 'base' resource definition file can just be named after the resource definition (e.g. "deployment.yml" can be the 'base' name file for a `Deployment` resource definition).  The 'base' resource definition file can also have a descriptive suffix if there are more than one of the same kind of resource definition file within the same directory (e.g. there might be two `MachineSet` resource definition files in the same directory which may be called "machineset-infra.yml" and "machineset-worker.yml").
* All files for resource definition patches must start with the name of the resource definition as the prefix and have a descriptive suffix (e.g. a patch for a `Deployment` resource definition that changes the replicas might be named "deployment-replicas.yml").
* Following the naming convention mentioned above will allow a resource definition file to be promoted through environments by moving the file from an "overlays" directory to a "variants" directory or the "base" directory.  The 'kustomization.yml' files should then be updated accordingly.

#### Example kustomization.yml File
This 'kustomization.yml' file could exist in an overlay for the S1A production cluster.
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: example-config

resources:
- ../../../../components/example-operator/base

components:
- ../../../../components/example-operator/variants/prod
- ../../../../components/example-operator/variants/s1

patches:
- path: deployment-image.yml
```

## Rendered Manifest Directory Structure
This is the directory structure under an "cluster/{CLUSTER_NAME}" branch.

### Overview
```
cluster-config/
└─ components/
   ├─ {COMPONENT_NAME}/
   └─ ...
```

### Explanation

#### Structure
* The "components" root directory is referenced by an Argo CD ApplicationSet.  Every sub-directory will be created as a separate Argo CD Application.
