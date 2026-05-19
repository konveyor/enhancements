---
title: crane-kustomizate-support
authors:
  - jmontleo
reviewers:
  - "@shawn-hurley"
  - "@sseago"
  - "@djzager"
approvers:
creation-date: 2022-07-12
last-updated: 2022-07-12
status: completed
see-also:
  - https://github.com/konveyor/crane/issues/128
  - https://github.com/konveyor/crane/issues/102
  - https://issues.redhat.com/browse/MTRHO-90
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane Export Improvements

This document covers proposed improvements to `crane export`.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

There are some cases we are aware of where resources are not exported using the
best resource group.

Cases include:
- Deployments exported from OpenShift 3.11 clusters are exported as `Deployment.extensions/v1beta1`, which is incompatible with newer versions.
- Deployments may be exported using the compatible `Deployment.appsv1` API.
- RoleBindings are exported as `RoleBinding.authorization.openshift.io` instead of `RoleBinding.rbac.authorization.k8s.io`.

## Motivation

The typical use case for crane will involve migrating from older to newer
versions of Kubernetes or OpenShift. From time to time resources have been
represented in in one API group and moved to another, with the first
becoming deprecated and later removed. Often times in older versions it is
possible to export the resource using the new API, as described in the Summary
point referencing Deployments. Whenever possible we should make the task of
modernizing easier.

### Goals

- Deal with known situations of cohabitating resources in a way that better
facilitates moving to newer clusters.

### Non-Goals

- While preferred versions may also come into play with the modernization, this enhancement is focused on contending with resources represented by multiple API groups.

## Proposal

### Phase I

- Export all API groups by enabling the feature in Velero Discovery
```
diff --git a/cmd/export/export.go b/cmd/export/export.go
index 4cd9c06..a91ec27 100644
--- a/cmd/export/export.go
+++ b/cmd/export/export.go
@@ -9,6 +9,7 @@ import (
        "github.com/konveyor/crane/internal/flags"
        "github.com/spf13/cobra"
        "github.com/spf13/viper"
+       velerov1api "github.com/vmware-tanzu/velero/pkg/apis/velero/v1"
        "github.com/vmware-tanzu/velero/pkg/discovery"
        "github.com/vmware-tanzu/velero/pkg/features"
        errorsutil "k8s.io/apimachinery/pkg/util/errors"
@@ -115,6 +116,7 @@ func (o *ExportOptions) Run() error {

        features.NewFeatureFlagSet()

+       features.Enable(velerov1api.APIGroupVersionsFeatureFlag)
        discoveryHelper, err := discovery.NewHelper(discoveryClient, log)
        if err != nil {
                log.Errorf("cannot create discovery helper: %#v", err)
```

- Adjust the file names for export, transform, and apply, for example:
```
--- a/cmd/export/discover.go
+++ b/cmd/export/discover.go
@@ -121,7 +121,7 @@ func getFilePath(obj unstructured.Unstructured) string {
        if namespace == "" {
                namespace = "clusterscoped"
        }
-       return strings.Join([]string{obj.GetKind(), namespace, obj.GetName()}, "_") + ".yaml"
+       return strings.Join([]string{obj.GetKind(), obj.GetObjectKind().GroupVersionKind().GroupKind().Group, namespace, obj.GetName()}, "_") + ".yaml"
 }
```

- Filter out versions that don't come from the groups preferred version

- The plugins will be responsible for whiting out undesirable duplicate resources from groups deprecated groups, such as `extensions` and `authorization.openshift.io`

### Phase II

- Filter resources based upon a label selector. The primary motivation for this functionality is to enable resources related to a specific application for GitOps Primer

### Phase III

- Allow passing in an OpenAPI spec when exporting so we can more intelligently select the best version based on versions available for each group within each cluster.

### Implementation Details/Notes/Constraints

- It is possible always preferring the newer versions could hinder a downgrade scenario.

### Test Plan

- Test migrations between versions of OpenShift known not to work, for instance
OpenShift 3.11 to 4.11.
- Test known other combinations to ensure they continue to work as expected.
