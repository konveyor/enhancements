---
title: crd-v1beta1-deprecation-remote-install
authors:
  - "@djwhatle"
reviewers:
  - "@shurley"
  - "@sseago"
  - "@alaypatel07"
  - "@pranavgaikwad"
approvers:
  - "@shurley"
  - "@sseago"
  - "@alaypatel07"
  - "@pranavgaikwad"
creation-date: 2021-07-21
last-updated: 2021-07-21
status: provisional
---

# CRD v1beta1 API deprecation handling via remote install

## Summary

This enhancement proposes Crane handling the upcoming removal of `apiextensions/v1beta1` from OCP 4.9 by:

- Dropping support for OCP 3.x as _control clusters_ 
- Dropping support for mig-operator to run on OCP 3.x. 
- Enhancing the MigCluster controller to perform remote install of dependencies such as Velero, Restic, and mig-log-reader.
- Dynamically installing a suitably versioned set of Velero CRDs on the remote cluster 

Users currently have the choice of which cluster (OCP 3.7 - 4.7) to use as the control cluster. This proposal would require users of MTC/Crane 1.6+ to use a minimum of OCP 4.3+ as their _control cluster_ to support `apiextensions/v1`. 

_In exchange for dropping OCP 3.x control cluster capability_, we gain the following benefits:

- We can avoid maintaining an additional long term support OCP 3.x compatible version of Crane
  - Test burden on QE is not doubled
  - Backport/CVE/Bugfix burden on devs is not doubled
- Only need to maintain and test one set of Crane CRDs (v1)
- Entire class of bugs related to running mig-controller on OCP 3.x clusters is eliminated


## Motivation

In OpenShift 4.9, the `apiextensions/v1beta1` CustomResourceDefinition (CRD) API will be removed. The replacement API `apiextensions/v1` has been available starting from OpenShift 4.3, with the two versions co-existing from 4.3-4.8. We previously used CRD v1beta1 everywhere, but OpenShift 4.9 represents the first time there is no CRD API version usable across all cluster versions we must support.

This enhancement outlines a strategy to handle this CRD API removal while continuing to inter-operate with 3.7+ clusters so that Crane can serve its core goal of allowing users to migrate OpenShift workloads from 3.7+ to 4.x clusters. 

The two sets of CRDs that Crane/MTC must install are:
- Mig* CRDs (must be installed on the control cluster only)
- Velero CRDs (must be installed on all clusters serving as a source / destination)

![crd_compat_api_chart](./crd_compat_api_chart.png)

### Goals

- Support main migration scenarios we care about as OCP API evolves
- Develop a strategy that can be used in the future when further Kubernetes API deprecations inevitably happen

### Non-Goals

- Introduce high burden of backporting fixes 
- Block introduction of new features/bugfixes into Crane 1.x

## Proposal

Common migration scenarios can be handled with this strategy:

- OCP 3.x to 4.6+
  - OCP 4.6+: Install Crane 1.6+ 
  - OCP 3.x: Install Crane dependencies remotely

- OCP 4.6+ to 4.6+
  - OCP 4.6+: Install Crane 1.6+ 
  - OCP 4.6+: Install Crane dependencies remotely

- OCP 4.1 to 4.5-
  - No longer possible 

- OCP 3.x to 3.x
  - No longer possible


### Implementation Details/Notes/Constraints

#### Migration of Config that previously lived in MigrationController CR to the MigCluster CR

Config info previously in the MigrationController CR spec on the remote cluster can be moved to an equivalent ConfigMap on the control cluster that would be referenced from the MigCluster  CR.

Alternatively we could include this config info directly on the MigCluster CR. This would require us to worry about schema changes, whereas the ConfigMap approach would be schema-less similar to how the MigrationConroller spec field is today.

#### Moving all Velero (and other remote dependency) setup tasks to MigCluster controller

Even on the control cluster, we should move all Velero, ConfigMap, and mig-log-reader setup tasks to the MigCluster controller so that we only have to maintain one set of templates, and we aren't doing half of our Velero installs in mig-operator, half in mig-controller.

#### Achieving functionality equivalent to mig-operator for Velero dependency install tasks

We may be able to make use of Jinja or Go templates to achieve remote cluster dependency installs.

#### Requirement for highly-permissioned remote cluster SA token

Currently, mig-operator sets up the `migration-controller` SA that we use to perform actions on the remote cluster. We would need the cluster admin of that cluster to create an SA token for us instead of having mig-operator do this. This should be a trivial difference.

#### Generating v1 CRDs

For Crane 1.6+ in mig-controller, patch the [controller-gen command](https://github.com/konveyor/mig-controller/blob/master/Makefile#L60) to generate v1 CRDs instead of v1beta1 ([see example in Velero repo](https://github.com/vmware-tanzu/velero/pull/3614/files#diff-a3e5a2ec9e9650645fbeea6cc08d0c17b3fdc0be893979ca180b329267d12a9cR48-R56)). Copy generated YAML over to mig-operator and run this modified operator against a pair of 3.11 and 4.x clusters. Verify that migrations still run as usual.

#### Blocking cluster upgrades until mig-operator can handle v1 CRDs

The OLM team has outlined steps that we can follow to ensure that the cluster upgrade to OpenShift 4.9 won't proceed until our operator can handle removal of the CRD v1beta1 API.

- Ensure released Crane versions have the required annotation to block upgrades to OCP 4.9 when they are installed.
- Ensure Crane 1.6 is available at time of OCP 4.9 release so that users have a way to unblock their upgrade by switching to Crane 1.6 release stream (will this available Crane version upgrade be obvious to users though?)
  ```
  # Example of blocking cluster upgrade with CSV annotation
  apiVersion: operators.coreos.com/v1alpha1
  kind: ClusterServiceVersion
  metadata:
    annotations:
      # Prevent cluster upgrades to OpenShift Version 4.9 when this
      # bundle is installed on the cluster
      "olm.properties": '[{"type": "olm.maxOpenShiftVersion", "value": "4.8"}]'
  ```

We will also need to configure Crane 1.6 with a minimum install version of OCP 4.3 due to v1 CRDs.

##### Strategies if Velero 1.7+ stops supporting v1beta1 CRDs

It is expected that Velero 1.7+ will stop supporting v1beta1 CRDs, meaning we would need to either

- Stop upgrading Crane with latest Velero
- Re-insert the v1beta1 CRD generation code into our fork of Velero, then take on any associated maintenence and test burden of running a modified fork.

### Risks and Mitigations

- _Risk:_ implementing this will require significant rework of how Velero, Restic, mig-log-reader, and our ConfigMaps are set up on the remote cluster. We currently carry out these setup tasks using Ansible tasks in mig-operator. We would need to create a functional equivalent of this provisioning code in Golang and have it undergo QE regression testing.
  - _Mitigation_: Run all available automated regression tests related to mig-operator settings as a gating factor before merge of these changes.

- _Risk_:  large amounts of documentation refer to the current methods for configuring Crane settings using the _MigrationController_ CR provided by mig-operator. Since this functionality would be moving out of the _MigrationController_ CR, we'd need to change all of our documentation to reflect this. 
  - _Mitigation_: work with docs team early on our solution, letting them know where things will be shifting. 


- _Risk_: it's not clear what the upgrade path would look like for users jumping from Crane 1.5 to Crane 1.6, given that all of their previous Config data on the MigrationController CR would need to land in its new home in Crane 1.6. We would need to develop a strategy for migrating users Crane config to the new format
  - _Mitigation_: if users upgrade their OCP 4.x clusters to the latest version, we could remotely read the config out of the MigrationController CR and copy it to the new location on the MigCluster spec (or wherever we decide to store config data)




## Alternatives

The original idea proposed as an enhancement was maintaining a long term support 1.5.z release for OCP 3.x clusters that would serve mostly as a Velero installer. Check out the `crd-v1beta1-deprecation` enhancement for more details.