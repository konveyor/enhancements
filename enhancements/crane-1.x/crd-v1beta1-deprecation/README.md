---
title: crd-v1beta1-deprecation
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
creation-date: 2021-07-14
last-updated: 2021-07-14
status: provisional
---

# CRD v1beta1 API deprecation handling

## Summary

This enhancement proposes maintaining a long-term support MTC/Crane 1.5.z version for users with source clusters on OpenShift 3.7 - 4.2 that will remain compatible with `apiextensions/v1beta1` CRDs. This version of MTC/Crane would receive only minor updates:

- CVE patches
- Bug fixes
- Velero updates where possible, though Velero may drop v1beta1 CRD support in future
- Changes to mig-operator managed ConfigMaps etc. to support small new features introduced in MTC/Crane 1.6+

Users currently have the choice of which cluster (OCP 3.7 - 4.7) to use as the control cluster. This proposal would require users of MTC/Crane 1.6+ to use a minimum of OCP 4.3+ as their _control cluster_ to support `apiextensions/v1`.
 
 <img src="./crd_compat_api_chart.png" width="700">

## Motivation

In OpenShift 4.9, the `apiextensions/v1beta1` CustomResourceDefinition (CRD) API will be removed. The replacement API `apiextensions/v1` has been available starting from OpenShift 4.3, with the two versions co-existing from 4.3-4.8. We previously used CRD v1beta1 everywhere, but OpenShift 4.9 represents the first time there is no CRD API version usable across all cluster versions we must support.

This enhancement outlines a strategy to handle this CRD API removal while continuing to inter-operate with 3.7+ clusters so that Crane can serve its core goal of allowing users to migrate OpenShift workloads from 3.7+ to 4.x clusters. 

The two sets of CRDs that Crane/MTC must install are:
- Mig* CRDs (must be installed on the control cluster only)
- Velero CRDs (must be installed on all clusters serving as a source / destination)

### Goals

- Support main migration scenarios we care about as OCP API evolves
- Develop a strategy that can be used in the future when further Kubernetes API deprecations inevitably happen
- Avoid introducing high burden of backporting fixes 
- Avoid blocking introduction of new features/bugfixes into Crane 1.x

## Proposal

Common migration scenarios can be handled with this strategy:

- OCP 3.x to 4.x
  - Use Crane 1.5.z for OCP 3
  - Use Crane 1.6+ for OCP 4
- OCP 3.x to 3.x
  - Use Crane 1.5.z for OCP 3


The thought is that since Crane 1 is mostly feature-complete, we shouldn't have to backport major API changes to Crane 1.5.z. 

For users in the traditional OCP 3.x -> 4.x migration scenario, Crane 1.5.z will esentially just serve as a Velero and ConfigMap installer, and these operator managed resources shouldn't need changes often if Crane 1.x is in maintenence mode.

We will need to make sure Crane 1.5.z receives matching operator updates. This probably means that some PRs to mig-operator for Crane 1.6+ will also get PR'd to the 1.5.z branch.


### Implementation Details/Notes/Constraints

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
      "olm.properties": '[{"type": "olm.maxOpenShiftVersion", "value": "4.8"}\]'
  ```


### Risks and Mitigations

See the "non-goals" section above. The main risk I see with this approach is the additional effort it will require to keep mig-operator changes for Crane 1.5.z maintained in addition to Crane 1.6+. This will likely also introduce some additional test burden.

If we limit the number of new features we add to Crane 1, we should be OK. Changes should be limited to bugfixes, CVEs, and perhaps Velero updates to mitigate this additional workload.


## Alternatives

Instead of maintaining two versions of mig-operator to keep migration capabilities from OCP 3 to 4, we could instead choose to move Crane setup routines to mig-controller, where upon receiving a permissioned SA token we could install Velero and any necessary ConfigMaps on the remote cluster. This would require a significant amount of shuffling of how config values are consumed, documentation re-writes, and planning for how to migrate users from the old system to the new. 
