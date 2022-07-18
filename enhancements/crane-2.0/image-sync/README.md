---
title: crane-image-transfer
authors:
  - djzager
reviewers:
  - "@shawn-hurley"
  - "@sseago"
  - "@jmontleon"
approvers:
  - "@shawn-hurley"
  - "@sseago"
  - "@jmontleon"
creation-date: 2022-07-13
last-updated: 2022-07-13
status: implementable
see-also:
  - "N/A"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane Image Transfer

This document tells the story for how users will migrate their Images from one
cluster to another, what `crane` will support, and how it will be supported.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Currently, we have a story for how users will migrate data from one cluster to
another (`transfer-pvc`) but we do not yet have an answer for how users will get
the images stored in the internal OpenShift registry from source->destination
clusters. The purpose of this document is to explain, with respect to images
stored in the internal OpenShift registry, how users will migrate them from one
cluster to another.

The sections below describe the 4 classes of images found in OpenShift cluster
registries.

### OpenShift Internal Images

The OpenShift internal images all reside in the `openshift` namespace, can be
found with `oc images -n openshift`, and come with the OpenShift cluster
installation.  These images are handled by the OpenShift cluster installation,
we will not migrate these images. It is the responsibility of the user to either
migrate the internal image or update their application to use an image in the
destination cluster.

### Publicly Available Images

Publicly available images are those images mirrored into the internal OpenShift
cluster's registry. There are a few ways this can be accomplished:

* `oc create imagestream python` and `oc create imagestreamtag python:latest --from-image='docker.io/library/python:latest'`
* `oc image mirror docker.io/library/python:latest ${registry}/library/python`
* `skopeo sync --src docker --dest docker docker.io/library/python:latest ${registry}/${namespace}`

Images that fall into this category can be identified by looking at the ImageStream's
`.status.tags[*].items[*].dockerImageReference` or an ImageStreamTag's
`.image.dockerImageReference` which will point to the publicly available image.

These images are already handled via
[crane-plugin-imagestream](https://github.com/konveyor/crane-plugin-imagestream),
we keep the ImageStreamTags that reference publicly available images as these
can be migrated by simply creating the ImageStreamTag on the destination cluster.

### Internally Built Images

Internally built images are those images in the OpenShift Cluster's registry
that are built in the cluster via BuildConfigs + Builds.

These images are currently handled in the
[crane-plugin-openshift](https://github.com/konveyor/crane-plugin-openshift)
where we drop Builds (as an artifact in the source cluster) and keep the
BuildConfigs for new Builds to reference in the destination cluster.

### Externally Built Images

Externally built images are those that are built outside the cluster and pushed
to OpenShift's cluster registry. For example:

```
image=${registry}/${namespace}/example:latest

docker build ${image} -f Dockerfile .
docker push ${image}
```

These images must be migrated manually as there is no Kubernetes resource that
can be `export`ed and brought to the destination cluster to restore the image.

**NOTE**

These images are of most concern to this enhancement.

## Motivation

Crane must have an answer for users wanting to migrate their images along with
their data and workloads to new clusters.

### Goals

* Provide a clear story for how users can get their images from source ->
    destination clusters.

### Non-Goals

* Migrate OpenShift Internal Images
* Re-Implement direct image migrations, `skopeo sync`, or `oc image mirror`.
    These tools are readily available to handle images.

## Proposal

### Crane `skopeo-sync-gen` subcommand

Introduce a new subcommand to crane, `skopeo-sync-gen`, that will look for and
evaulate all `export`ed ImageStreams to determine if their images (and the
image's tags) should be migrated. If an image is internally or externally built
(ie. is not mirrored from another registry like quay|docker), then it will be
staged.

This command will write to standard output
[YAML file content for `skopeo sync` source](https://github.com/containers/skopeo/blob/main/docs/skopeo-sync.1.md).

This is covered by [this GitHub Issue](https://github.com/konveyor/crane/issues/138).

### Crane Runner `oc registry info` ClusterTask

Implement a new `oc-registry-info` ClusterTask that [emits
Results](https://tekton.dev/docs/pipelines/tasks/#emitting-results) for the
internal and and public URLs for the OpenShift cluster's registry.

This is covered by [this GitHub Issue](https://github.com/konveyor/crane-runner/issues/54).

### Crane Runner `crane-skopeo-sync-gen` ClusterTask

Implement a new `crane-skopeo-sync-gen` ClusterTask that executes the new
subcommand and saves the output to a file in a
[Workspace](https://tekton.dev/docs/pipelines/tasks/#emitting-results).

This is covered by [this GitHub Issue](https://github.com/konveyor/crane-runner/issues/55).

### Crane Runner `crane-skopeo-sync` ClusterTask

This new ClusterTask will be responsible for accepting a source yaml (like the
one generated from `crane skopeo-sync-gen`) and running `skopeo sync` to copy
the images to a destination registry.

This task will rely on the kubeconfig for the source and destination cluster to
log into the respective clusters (ie. `podman login -u $(oc whoami) -p $(oc
whoami -t)`).

This is covered by [this GitHub Issue](https://github.com/konveyor/crane-runner/issues/56).

**NOTE**

This command should be made as generic as possible, but must satisfy the use
case outlined here first. That is why the `crane-` prefix has been added.

### Crane UI Updates

The new `skopeo-sync-gen` command enables users to migrate using the CLI.
Updates to the UI component are required in order to tie these pieces together
into consumable user actions.

#### New Pipeline

Currently, the UI import wizard recognizes when there are PeristentVolumeClaims
(PVCs) to be migrated and asks the user for additional input for handling this
data. We will update the import wizard to also recognize wihen there are
ImageStreams to migrate. If the user wishes to migrate the images, we can
construct a separate pipeline that runs ClusterTasks (in order):

1. `crane-export` targeting source cluster
1. `oc-registry-info` targeting source and destination clusters
1. `crane-skopeo-sync-gen` using the emitted registry info and
   the export results stored in the common workspace
1. `skopeo-sync` targeting destination cluster to actually transfer the images

This is covered by [this GitHub Issue](https://github.com/konveyor/crane-ui-plugin/issues/110).

#### Updates to Existing Pipeline

With the new pipeline and existing openshift + imagestream plugins, what is left
is to update the references to the ImageStreams in the Kube resources that are
migrated. The default kubernetes plugin has a `registry-replacement` optional
flag that we must leverage to ensure user's workloads can find the images that
have been migrated. For example, if a DeploymentConfig
in the `source` namespace that referenced an ImageStream were migrated to a
`destination` namespace we would need to update the reference from
`${source_cluster_internal_registry_url}/source/example:latest` to
`${destination_cluster_internal_registry_url}/destination/example:latest`.

We must update the existing pipeline to:

1. Run the `oc-registry-info` ClusterTask and capture the internal registry
   value from the source and destination cluster
1. Pass the computed `registry-replacement` to the `crane-transform`
   ClusterTask.

This is covered by [this GitHub Issue](https://github.com/konveyor/crane-ui-plugin/issues/111).

## Drawbacks

We should be encouraging user's to move away from ImageStreams, this may be a
place where we could be helping user's modernize.
