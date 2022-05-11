---
title: tekton-ui-integration
authors:
  - "@djzager"
reviewers:
  - "@alaypatel107"
  - "@mturley"
  - "@shawn-hurley"
approvers:
  - "@mturley"
  - "@shawn-hurley"
creation-date: 2022-01-18
last-updated: 2022-05-06
status: implementable
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane UI Integration with Tekton for Excution of Crane Workflows

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. How will the Crane Workflows, specifically the TaskRuns responsible for
   applying the users manifests onto the destination cluster, impersonate the
   user making the request? 
   * **ANSWER** This will be covered in a follow-up enhancement. See [MIG-1076](https://issues.redhat.com/browse/MIG-1076).
       Until superceded, the UI will create a Secret for the credentials to
       access the destination cluster and these will be consumed the same way
       source cluster credentials are consumed.
   * Research into how Vault could help us handle user auth will be researched
       in [MIG-1083](https://issues.redhat.com/browse/MIG-1083).
1. Getting the operator to override the Crane Runner container image specified
   in ClusterTask Steps. Right now ClusterTasks all reference a
   `quay.io/konveyor/crane-runner:latest` image.
   * **ANSWER** In the same way operators use environment variables defined on
       operator deployments in ClusterServiceVersions (or overridden via
       Subscription Configs), the operator will override the image specified in
       the deployed ClusterTasks. There is precedence for this in
       [mig-operator](https://github.com/konveyor/mig-operator/blob/master/deploy/olm-catalog/bundle/manifests/crane-operator.v99.0.0.clusterserviceversion.yaml#L673-L724)
       and should allow us to handle network restricted scenarios without
       additional effort.
   * Labelling container images with build information sufficient for debugging
       purposes is a task covered in https://github.com/konveyor/crane-runner/issues/33
1. Do we want to to execute the `crane-transfer-pvc` ClusterTask for each PVC to
   be migrated or should `crane transfer-pvc` support handling all the PVCs in a
   namespace?
   * **ANSWER** We are going to run the `crane-transfer-pvc` for each PVC to be
       migrated.
   * While, fixing a bug in `crane` will allow the UI to generate Pipelines that
       parallelize `transfer-pvc`, handling of: throttling, quotas, retries,
       etc. will be researched in [MIG-1082](https://issues.redhat.com/browse/MIG-1082).
1. Is it possible that OpenShift's Pipelines UI will be updated to handle moving
   a PipelineRun out of "PipelineRunPending"?
   * RFE submitted to [openshift/console#11060](https://github.com/openshift/console/issues/11060)

## Glossary/References

* [Step](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#defining-steps) -
    a reference to a container image that executes a specific tool on a specific
    input and produces a specific output.
* [Task](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md) -
    a collection of steps executed in order. Tasks, when executed as a TaskRun,
    are run as a Pod in the cluster. Tasks are the smallest building blocks in
    Tekton.
* [ClusterTask](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#task-vs-clustertask) - 
    a cluster-scoped Task. It behaves identically to a Task. The only difference
    is that ClusterTasks can be referenced by (Task|Pipeline)Runs in any
    namespace.
* [TaskRun](https://github.com/tektoncd/pipeline/blob/main/docs/taskruns.md) -
    an instantiation and execution of a Task.
* [Pipeline](https://github.com/tektoncd/pipeline/blob/main/docs/pipelines.md) -
    define a collection of Tasks.
* [PipelineRun](https://github.com/tektoncd/pipeline/blob/main/docs/pipelineruns.md) -
    an instantiation and execution of a Pipeline. One important thing to know
    about PipelineRuns (this is also true for TaskRuns) is that it is possible
    to embed the Pipeline into the
    [PipelineRun directly](https://github.com/tektoncd/pipeline/blob/main/docs/pipelineruns.md#specifying-the-target-pipeline).
    PipelineRun executions can be deferred by [specifying on PipelineRun.Spec.Status "PipelineRunPending"](https://tekton.dev/docs/pipelines/pipelineruns/#pending-pipelineruns).
* Parameters - can be specified on [Tasks](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#specifying-parameters)
    or on [Pipelines](https://github.com/tektoncd/pipeline/blob/main/docs/pipelines.md#specifying-parameters).
    Flags to be set on crane (like the namespace to export) are good examples of
    parameters to be exposed. Parameters can have default values.
* [Workspaces](https://github.com/tektoncd/pipeline/blob/main/docs/workspaces.md) -
    allow Tasks to declare parts of the filesystem that need to be provided at
    runtime by TaskRuns. A TaskRun can make these parts of the filesystem
    available in many ways: using a read-only ConfigMap or Secret, an existing
    PersistentVolumeClaim shared with other Tasks, create a
    PersistentVolumeClaim from a provided VolumeClaimTemplate, or simply an
    emptyDir that is discarded when the TaskRun completes.
    Workspaces can be declared as
    [optional](https://github.com/tektoncd/pipeline/blob/main/docs/workspaces.md#optional-workspaces).

## Summary

The purpose of this enhancement is to ensure integration of the Crane UI
component with Tekton ClusterTasks supported by Crane Runner.
[Crane Runner](https://github.com/konveyor/crane-runner/tree/742883ce69861510ab80f1c6c253cc759d7091b1)
currently publishes a container image that includes the `crane` binary.
This container image is referenced
by Crane Runner's supported ClusterTasks representing different `crane`
workflows like
[crane export](https://github.com/konveyor/crane-runner/blob/742883ce69861510ab80f1c6c253cc759d7091b1/manifests/clustertasks/crane-export.yaml).

## Motivation

The UI component of "Crane 2.0" must integrate seemlessly with what is being
delievered by Crane Runner. Once a user completes Crane UI's migration wizard,
we want to hand off the workflow to Tekton and the Pipelines UI to allow the
user to run the workflow at their discretion. This work will allow a seemless
experience for Crane UI and command-line users alike.

### Goals

* The UI component outputs a Tekton Pipeline that can be executed at the users'
    discretion.
* The Tekton Pipeline can be modified and re-used by the end user.
* The ClusterTasks provided by Crane Runner should still be consumable via
    Pipelines and PipelineRuns that are not generated via the UI.
* Avoid reimplementing functionality that Tekton already provides. Tekton
    provides a robust foundation on which we can build with Tasks and TaskRuns,
    Pipeline and PipelineRuns, Parameters, Workspaces, and even a UI for
    managing Pipelines/PipelineRuns in OpenShift. This enhancement is deliberate
    in it's intent to rely on work already done in Tekton.

### Non-Goals

* User impersonation. It is desirable for the ClusterTasks that interact with
    the destination cluster to do so as the user who submitted the request. This
    is a difficult problem that requires it's own enhancement. See [Open Questions](##open-questions).

## Proposal

### Crane Runner Container Image

Crane Runner will publish and support a container image including the following
binaries:

* crane - for executing crane commands
* oc & kubectl - for interacting with cluster resources and/or managing cluster
    configuration.
* kustomize - for customizing the manifests after `crane apply`. We require this
    binary in order to "initialize" a kustomize base layer from YAML. This
    allows us to perform some basic "destination" cluster preparation like
    changing the namespace where resources will be installed, adding labels, or
    configuring resource prefix + suffix.

### Crane Runner ClusterTasks

Crane Runner will support, at a minimum, the ClusterTasks described in this
section. These are the primary integration between Crane's UI component and Tekton.
The UI must know about each of these ClusterTasks, the parameters and workspaces
they require, and how they are to be ordered in Pipelines. This is to facilitate
the UI generating Pipelines and PipelineRuns (more details provided in
[UI Integration with ClusterTasks](###ui-integration-with-clustertasks))
allowing a seamless transition from Crane's UI to the Pipeline's UI.

#### crane-kubeconfig-generator

The `crane-kubeconfig-generator` ClusterTask is responsible for taking a secret
holding an API Server URL + Token and generating a kubeconfig. Saving the
kubeconfig in a workspace will allow it to be shared with follow-up tasks.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: crane-kubeconfig-generator
spec:
  params:
    - name: cluster-secret
      type: string
      description: |
        The name of the secret holding cluster API Server URL and Token.
    - name: context-name
      type: string
      description: |
        The name to give the context.
  steps:
    - name: crane-kubeconfig-generator
      image: quay.io/konveyor/crane-runner:latest
      script: |
        export KUBECONFIG=$(workspaces.kubeconfig.path)/kubeconfig

        set +x
        oc login --server=$CLUSTER_URL --token=$CLUSTER_TOKEN
        set -x

        kubectl config rename-context "$(kubectl config current-context)" "$(params.context-name)"
      env:
      - name: CLUSTER_URL
        valueFrom:
          secretKeyRef:
            name: $(params.cluster-secret)
            key: url
      - name: CLUSTER_TOKEN
        valueFrom:
          secretKeyRef:
            name: $(params.cluster-secret)
            key: token 
  workspaces:
    - name: kubeconfig
      readOnly: false
      description: |
        Where the generated kubeconfig will be saved.
```

#### crane-export

The `crane-export` is responsible for exporting resources from the source
cluster.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: crane-export
spec:
  params:
    - name: context
      type: string
      description: |
        The name of the context from kubeconfig representing the source
        cluster.

        You can get this information in your current environment using
        `kubectl config get-contexts` to describe your one or many
        contexts.
      default: ""
    - name: namespace
      type: string
      description: |
        The namespace in the specified cluster, via kubeconfig workspace and/or
        context parameter, from which to export resources.
      default: ""
  steps:
    - name: crane-export
      image: quay.io/konveyor/crane-runner:latest
      script: |
        /crane export \
          --context="$(params.context)" \
          --namespace="$(params.namespace)" \
          --export-dir="$(workspaces.export.path)"

        find $(workspaces.export.path)
      env:
        - name: KUBECONFIG
          value: $(workspaces.kubeconfig.path)/kubeconfig
  workspaces:
    - name: export
      description: |
        Directory where results of crane export will be stored for future use
        in other tasks.
      mountPath: /var/crane/export
    - name: kubeconfig
      description: |
        The kubeconfig for accessing the cluster.
```

#### crane-transform

The `crane-transform` ClusterTask is responsible for generating JSON patches.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: crane-transform
spec:
  params:
    - name: opitonal-flags
      type: string
      description: |
        Comma separated list of `flag-name=value` pairs. These flags with values
        will be passed into all pluins that are executed in the transform
        operation.
      default: ""
  steps:
    - name: crane-transform
      image: quay.io/konveyor/crane-runner:latest
      script: |
        /crane transform \
          --ignored-patches-dir="$(workspaces.ignored-patches.path)" \
          --flags-file="$(workspaces.craneconfig.path)" \
          --optional-flags="$(params.optional-flags)" \
          --export-dir="$(workspaces.export.path)" \
          --transform-dir=$(workspaces.transform.path)

        find $(workspaces.transform.path)
        if [ "$(workspaces.ignored-patches.bound)" == "true" ]; then
          find $(workspaces.ignored-patches.path)
        fi
  # https://github.com/tektoncd/pipeline/blob/main/docs/workspaces.md#using-workspaces-in-tasks
  workspaces:
    - name: export
      description: |
        This is the folder where the results of crane export were stored.
      mountPath: /var/crane/export
    - name: transform
      description: |
        This is the folder where we will store the results of crane transform.
      mountPath: /var/crane/transform
    - name: ignored-patches
      description: |
        This is the folder where the results of crane ignored-patches were stored.
      mountPath: /var/crane/ignored-patches
      optional: true
    - name: craneconfig
      description: |
        This is where we hold the configuration file for crane.
      mountPath: /var/crane/config
      optional: true
```

#### crane-apply

The `crane-apply` ClusterTask is responsible for taking the resources form
export and applying the JSON patches generated in transform to create new
manifests ready to be installed on any cluster.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: crane-apply
spec:
  steps:
    - name: crane-apply
      image: quay.io/konveyor/crane-runner:latest
      script: |
        /crane apply \
          --export-dir=$(workspaces.export.path) \
          --transform-dir=$(workspaces.transform.path) \
          --output-dir=$(workspaces.apply.path)
        find $(workspaces.apply.path)
  workspaces:
    - name: export
      description: |
        This is the folder where the results of crane export were stored.
      mountPath: /var/crane/export
    - name: transform
      description: |
        This is the folder where we will store the results of crane transform.
      mountPath: /var/crane/transform
    - name: apply
      description: |
        This is the folder where we will store the results of crane apply.
      mountPath: /var/crane/apply
```

#### crane-transfer-pvc

The `crane-transfer-pvc` ClusterTask is responsible for syncing a single PVC
from source to destination cluster.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: crane-transfer-pvc
spec:
  params:
    - name: source-context
      type: string
      description: |
        The name of the context from kubeconfig representing the source
        cluster.

        You can get this information in your current environment using
        `kubectl config get-contexts` to describe your one or many
        contexts.
    - name: source-namespace
      type: string
      description: |
        The source cluster namespace in which pvc is synced.
    - name: source-pvc-name
      type: string
      description: |
        The name of the pvc to be synced from source cluster.
    - name: dest-context
      type: string
      description: |
        The name of the context from kubeconfig representing the destination
        cluster.

        You can get this information in your current environment using
        `kubectl config get-contexts` to describe your one or many
        contexts.
    - name: dest-pvc-name
      type: string
      description: |
        The name to give pvc in destination cluster.
      default: ""
    - name: dest-namespace
      type: string
      description: |
        The source cluster namespace in which pvc is synced.
      default: ""
    - name: dest-storage-class-name
      type: string
      description: |
        The name of the storage class to use in the destination cluster.
      default: ""
    - name: dest-pvc-capacity
      type: string
      description: |
        Size of the destination volume to create.
      default: ""
    - name: endpoint-type
      type: string
      description: |
        The name of the networking endpoint to be used for ingress traffic in the destination cluster
      default: ""
  steps:
    - name: crane-transfer-pvc
      image: quay.io/konveyor/crane-runner:latest
      script: |
        /crane transfer-pvc \
          --source-context=$(params.source-context) \
          --destination-context=$(params.dest-context) \
          --pvc-name $(params.source-pvc-name):$(params.dest-pvc-name) \
          --pvc-namespace $(params.source-namespace):$(params.dest-namespace) \
          --dest-pvc-storage-class-name $(params.dest-storage-class-name) \
          --pvc-requests-storage $(params.dest-pvc-capacity) \
          --endpoint $(params.endpoint-type)
      env:
        - name: KUBECONFIG
          value: $(workspaces.kubeconfig.path)/kubeconfig
  workspaces:
    - name: kubeconfig
      description: |
        The kubeconfig for accessing the source cluster.
```

#### crane-kubectl-scale-down

The `kubectl-scale-down` ClusterTask is responsible for scaling down the
resources in a namespace.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: kubectl-scale-down
spec:
  params:
    - name: context
      type: string
      description: |
        Context to use when scaling down resources
      default: ""
    - name: namespace
      type: string
      description: |
        Namespace to use when scaling down resources
      default: ""
    - name: resource-type
      type: string
      description: |
        The resource type to be scaled down.
  steps:
    - name: kubectl-scale-down
      image: quay.io/konveyor/crane-runner:latest
      script: |
        kubectl scale --context "$(params.context)" --namespace "$(params.namespace)" --replicas=0 "$(params.resource-type)" --all
      env:
        - name: KUBECONFIG
          value: $(workspaces.kubeconfig.path)/kubeconfig
  workspaces:
    - name: kubeconfig
      description: |
        The kubeconfig for accessing the source cluster.
```

#### crane-kustomize-init

The `crane-kustomize-init` ClusterTask is responsible for packaging the
manifests created in `crane apply` for use in Kustomize. This is necessary to
customize the namespace.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: crane-kustomize-init
spec:
  params:
    - name: source-namespace
      type: string
      description: Source namespace from export.
    - name: labels
      type: string
      description: Add one or more labels
      default: ""
    - name: name-prefix
      type: string
      description: Set the namePrefix field in the kustomization file.
      default: ""
    - name: namespace
      type: string
      description: Sets the value of the namespace field in the kustomization file.
      default: ""
    - name: name-suffix
      type: string
      description: Set the nameSuffix field in the kustomization file.
      default: ""
  steps:
    - name: kustomize-namespace
      image: quay.io/konveyor/crane-runner:latest
      script: |
        # Copy apply resources into kustomize workspace
        cp -r "$(workspaces.apply.path)/resources/$(params.source-namespace)/." "$(workspaces.kustomize.path)"

        pushd "$(workspaces.kustomize.path)"
        kustomize init --autodetect --labels "$(params.labels)" --nameprefix "$(params.name-prefix)" --namespace "$(params.namespace)" --nameSuffix "$(params.name-suffix)"
        kustomize build
        popd
        tree "$(workspaces.kustomize.path)"
  # https://github.com/tektoncd/pipeline/blob/main/docs/workspaces.md#using-workspaces-in-tasks
  workspaces:
    - name: apply
      description: |
        This is the folder where the results from crane-apply are stored.
      mountPath: /var/crane/apply
    - name: kustomize
      description: |
        This is where the kustomize related manifests will be saved.
```

#### kubectl-apply-kustomize

The `kubectl-apply-kustomize` ClusterTask is responsible for applying the
resources defined in a `kustomization.yaml` at the root of the provided
`kustomize` workspace.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: kubectl-apply-kustomize
spec:
  params:
    - name: context
      type: string
      description: The context from the kubeconfig that represents the destination cluster.
      default: ""
  steps:
    - name: kubectl-apply
      image: quay.io/konveyor/crane-runner:latest
      script: |
        if [ "$(workspaces.kubeconfig.bound)" == "true" ] ; then
          export KUBECONFIG="$(workspaces.kubeconfig.path)/kubeconfig"
        fi

        kubectl --context="$(params.context)" apply -k "$(workspaces.kustomize.path)"
  workspaces:
    - name: kustomize
      description: |
        This is the folder storing a kustomization.yaml file to be applied.
    - name: kubeconfig
      description: |
        The user's kubeconfig. Otherwise, will just rely on serviceaccount.
      optional: true
```

### UI Integration with ClusterTasks

With knowledge of the supported ClusterTasks, the UI will instantiate Pipelines and
PipelineRuns based on a user's wizard responses. The source and destination cluster's
credentials will be saved as a secret, for example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  generateName: source-cluster-
data:
  url: ... (base64-encoded)
  token: ... (base64-encoded)
type: Opaque
```

[Interacting with Pipelines using the developer perspective in OpenShift](https://docs.openshift.com/container-platform/4.9/cicd/pipelines/working-with-pipelines-using-the-developer-perspective.html#op-interacting-with-pipelines-using-the-developer-perspective_working-with-pipelines-using-the-developer-perspective)
centers around the Pipelines -- PipelineRuns that aren't associated with a
Pipeline are very difficult to discover or manage -- for this reason the
UI will **NOT** be instantiating PipelineRuns with embeded Pipeline
specifications. After the user completes the wizard, the responses will be
used to instantiate Pipeline(s) and PipelineRun(s). Examples of the different
Pipelines and PipelineRuns are presented in the [User Stories](###user-stories)
section.

When the user decides to defer execution of the generated
PipelineRun it will be given a `.spec.status` of "PipelineRunPending". Later,
when the user wants to execute the PipelineRun, all the user must do is remove
the `.spec.status` from the PipelineRun or managed via the Pipelines UI in
OpenShift. 

### User Stories

#### Stateless Application Migration

As an application owner, I want to migrate my stateless application from a
source to destination cluster.

Through completion of the wizard, the UI will store source and destination
cluster's API Server URL and Token in secrets named `source-cluster-1234` and
`destination-cluster-1234` respectively and the `guestbook` namespace was
selected for migration where no PVCs exist (or where PVCs exist but none were
chosen).

The generated Pipeline:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: guestbook-migration
spec:
  params:
    - name: source-cluster-secret
      type: string
    - name: source-namespace
      type: string
    - name: destination-cluster-secret
      type: string
    - name: optional-flags
      type: string
  workspaces:
    - name: shared-data
    - name: kubeconfig
  tasks:
    - name: generate-source-kubeconfig
      params:
        - name: cluster-secret
          value: "$(params.source-cluster-secret)"
        - name: context-name
          value: source
      taskRef:
        name: crane-kubeconfig-generator
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: generate-destination-kubeconfig
      params:
        - name: cluster-secret
          value: "$(params.source-cluster-secret)"
        - name: context-name
          value: destination
      taskRef:
        name: crane-kubeconfig-generator
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: export
      runAfter:
        - generate-source-kubeconfig
        - generate-destination-kubeconfig
      params:
        - name: context
          value: source
        - name: namespace
          value: "$(params.source-namespace)"
      taskRef:
        name: crane-export
        kind: ClusterTask
      workspaces:
        - name: export
          workspace: shared-data
          subPath: export
        - name: kubeconfig
          workspace: kubeconfig
    - name: transform
      runAfter:
        - export
      params:
        - name: optional-flags
          value: "$(params.optional-flags)"
      taskRef:
        name: crane-transform
        kind: ClusterTask
      workspaces:
        - name: export
          workspace: shared-data
          subPath: export
        - name: transform
          workspace: shared-data
          subPath: transform
    - name: apply
      runAfter:
        - transform
      taskRef:
        name: crane-apply
        kind: ClusterTask
      workspaces:
        - name: export
          workspace: shared-data
          subPath: export
        - name: transform
          workspace: shared-data
          subPath: transform
        - name: apply
          workspace: shared-data
          subPath: apply
    - name: kustomize-init
      runAfter:
        - apply
      params:
        - name: source-namespace
          value: "$(params.source-namespace)"
        - name: namespace
          value: "$(context.taskRun.namespace)"
      taskRef:
        name: crane-kustomize-init
        kind: ClusterTask
      workspaces:
        - name: apply
          workspace: shared-data
          subPath: apply
        - name: kustomize
          workspace: shared-data
    - name: kubectl-apply-kustomize
      runAfter:
        - kustomize-init
      params:
        - name: context
          value: destination
      taskRef:
        name: kubectl-apply-kustomize
        kind: ClusterTask
      workspaces:
        - name: kustomize
          workspace: shared-data
```

The generated PipelineRun:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: guestbook-migration-
spec:
  params:
    - name: source-cluster-secret
      value: source-cluster-1234
    - name: destination-cluster-secret
      value: destination-cluster-1234
    - name: source-namespace
      value: guestbook
    - name: optional-flags
      value: ""
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Mi
    - name: kubeconfig
      emptyDir: {}
  pipelineRef:
    name: guestbook-migration
```

#### Stateful Applicaiton Migration

As an application owner, I want to migrate my application with state from a
source to destination cluster.

Through completion of the wizard, the UI will store source and destination
cluster's API Server URL and Token in secrets named `source-cluster-1234` and
`destination-cluster-1234` respectively and the `guestbook` namespace where
two PVCs exist and are selected for migration named `redis-data01` and
`redis-data02`.

For this use case, the UI will generate two Pipelines and PipelineRuns:

* Stage
* Cutover

##### Stage Pipeline and PipelineRun

This Pipeline and PipelineRun is responsible for migrating the state of the
application from the source to destination cluster. It's important to note that
these don't quiesce the application before transferring state which makes it
suitable for running to limit how long the cutover will take.

The generated Pipeline:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: guestbook-stage
spec:
  params:
    - name: source-cluster-secret
      type: string
    - name: source-namespace
      type: string
    - name: destination-cluster-secret
      type: string
  workspaces:
    - name: kubeconfig
  tasks:
    - name: generate-source-kubeconfig
      params:
        - name: cluster-secret
          value: "$(params.source-cluster-secret)"
        - name: context-name
          value: source
      taskRef:
        name: crane-kubeconfig-generator
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: generate-destination-kubeconfig
      params:
        - name: cluster-secret
          value: "$(params.source-cluster-secret)"
        - name: context-name
          value: destination
      taskRef:
        name: crane-kubeconfig-generator
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: redis-data01
      runAfter:
        - generate-source-kubeconfig
        - generate-destination-kubeconfig
      params:
        - name: source-context
          value: source
        - name: source-namespace
          value: "$(params.source-namespace)"
        - name: source-pvc-name
          value: redis-data01
        - name: dest-context
          value: destination
        - name: dest-pvc-name
          value: redis-data01
        - name: dest-namespace
          value: "$(context.taskRun.namespace)"
        - name: dest-storage-class-name
          value: ""
        - name: dest-pvc-capacity
          value: 10GiB
        - name: endpoint-type
          value: route
      taskRef:
        name: crane-transfer-pvc
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: redis-data02
      runAfter:
        - generate-source-kubeconfig
        - generate-destination-kubeconfig
      params:
        - name: source-context
          value: source
        - name: source-namespace
          value: "$(params.source-namespace)"
        - name: source-pvc-name
          value: redis-data02
        - name: dest-context
          value: destination
        - name: dest-pvc-name
          value: redis-data02
        - name: dest-namespace
          value: "$(context.taskRun.namespace)"
        - name: dest-storage-class-name
          value: ""
        - name: dest-pvc-capacity
          value: 10GiB
        - name: endpoint-type
          value: route
      taskRef:
        name: crane-transfer-pvc
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
```

The generated PipelineRun:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: guestbook-stage-
spec:
  params:
    - name: source-cluster-secret
      value: source-cluster-1234
    - name: source-namespace
      value: guestbook
    - name: destination-cluster-secret
      value: destination-cluster-1234
  workspaces:
    - name: kubeconfig
      emptyDir: {}
  pipelineRef:
    name: guestbook-stage
```

##### Cutover Pipeline and PipelineRun

The purpose of this Pipeline and PipelineRun is to "completely" migrate the
application and it's state to the destination cluster from the source. An
important piece of this workflow is that it will quiesce the application on the
source cluster before performing a state transfer and migrating the application.

The generated Pipeline:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: guestbook-cutover
spec:
  params:
    - name: source-cluster-secret
      type: string
    - name: source-namespace
      type: string
    - name: destination-cluster-secret
      type: string
    - name: optional-flags
      type: string
  workspaces:
    - name: shared-data
    - name: kubeconfig
  tasks:
    - name: generate-source-kubeconfig
      params:
        - name: cluster-secret
          value: "$(params.source-cluster-secret)"
        - name: context-name
          value: source
      taskRef:
        name: crane-kubeconfig-generator
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: generate-destination-kubeconfig
      params:
        - name: cluster-secret
          value: "$(params.source-cluster-secret)"
        - name: context-name
          value: destination
      taskRef:
        name: crane-kubeconfig-generator
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: export
      runAfter:
        - generate-source-kubeconfig
        - generate-destination-kubeconfig
      params:
        - name: context
          value: source
        - name: namespace
          value: "$(params.source-namespace)"
      taskRef:
        name: crane-export
        kind: ClusterTask
      workspaces:
        - name: export
          workspace: shared-data
          subPath: export
        - name: kubeconfig
          workspace: kubeconfig
    - name: kubectl-scale-down
      runAfter:
        - export
      params:
        - name: context
          value: source
        - name: namespace
          value: "$(params.source-namespace)"
        - name: resource-type
          value: deployments,statefulesets
      taskRef:
        name: crane-kubectl-scale-down
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: redis-data01
      runAfter:
        - kubectl-scale-down
      params:
        - name: source-context
          value: source
        - name: source-namespace
          value: "$(params.source-namespace)"
        - name: source-pvc-name
          value: redis-data01
        - name: dest-context
          value: destination
        - name: dest-pvc-name
          value: redis-data02
        - name: dest-namespace
          value: "$(context.taskRun.namespace)"
        - name: dest-storage-class-name
          value: ""
        - name: dest-pvc-capacity
          value: 10GiB
        - name: endpoint-type
          value: route
      taskRef:
        name: crane-transfer-pvc
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: redis-data02
      runAfter:
        - kubectl-scale-down
      params:
        - name: source-context
          value: source
        - name: source-namespace
          value: "$(params.source-namespace)"
        - name: source-pvc-name
          value: redis-data02
        - name: dest-context
          value: destination
        - name: dest-pvc-name
          value: redis-data02
        - name: dest-namespace
          value: "$(context.taskRun.namespace)"
        - name: dest-storage-class-name
          value: ""
        - name: dest-pvc-capacity
          value: 10GiB
        - name: endpoint-type
          value: route
      taskRef:
        name: crane-transfer-pvc
        kind: ClusterTask
      workspaces:
        - name: kubeconfig
          workspace: kubeconfig
    - name: transform
      runAfter:
        - redis-data01
        - redis-data02
      params:
        - name: optional-flags
          value: "$(params.optional-flags)"
      taskRef:
        name: crane-transform
        kind: ClusterTask
      workspaces:
        - name: export
          workspace: shared-data
          subPath: export
        - name: transform
          workspace: shared-data
          subPath: transform
    - name: apply
      runAfter:
        - transform
      taskRef:
        name: crane-apply
        kind: ClusterTask
      workspaces:
        - name: export
          workspace: shared-data
          subPath: export
        - name: transform
          workspace: shared-data
          subPath: transform
        - name: apply
          workspace: shared-data
          subPath: apply
    - name: kustomize-init
      runAfter:
        - apply
      params:
        - name: source-namespace
          value: "$(params.source-namespace)"
        - name: namespace
          value: "$(context.taskRun.namespace)"
      taskRef:
        name: crane-kustomize-init
        kind: ClusterTask
      workspaces:
        - name: apply
          workspace: shared-data
          subPath: apply
        - name: kustomize
          workspace: shared-data
    - name: kubectl-apply-kustomize
      runAfter:
        - kustomize-init
      params:
        - name: context
          value: destination
      taskRef:
        name: kubectl-apply-kustomize
        kind: ClusterTask
      workspaces:
        - name: kustomize
          workspace: shared-data
```

The generated PipelineRun:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: guestbook-cutover-
spec:
  params:
    - name: cluster-secret
      value: source-cluster-creds
    - name: source-namespace
      value: guestbook
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Mi
    - name: kubeconfig
      emptyDir: {}
  pipelineRef:
    name: guestbook-cutover-migration
```

### Security, Risks, and Mitigations

The greatest security risks come from our handling of cluster access and
authorization -- both the source and destination clusters. We have ClusterTasks
that login (ie. `crane-kubeconfig-generator`), some more that rely on the
generated Kubeconfig (ie. `crane-export` and `crane-transfer-pvc`), and
others that interact with the
[mounted secrets to access Kubernetes API from inside the Pod](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/#directly-accessing-the-rest-api)
(ie. `crane-transfer-pvc` and `kubectl-apply-kustomize`).

We must be careful not to expose credentials when performing the `oc login`
using the API server URL and token. The plan is to mitigate this using `set +x`
to prevent the login line from being included in Pod logs.

For ClusterTasks that require access to the destination cluster, we will use the
`admin` serviceAccount in the namespace where the PipelineRun is instantiated.
This should sufficiently limit the scope of the TaskRuns to one namespace.

## Design Details

### Upgrade / Downgrade Strategy

It is important that we treat the ClusterTasks supported by Crane Runner **as
our API contract**, we are not able to version our ClusterTasks via OCI layering
and will need to be careful to keep the UI in sync with the ClusterTasks. The
greatest probably of API incompatibilities are from adding/removing parameters
or workspaces from ClusterTasks.

To help mitigate these risks, whatever container image specified for Crane
Runner in the operator's environment (ie. `CRANE_RUNNER_IMAGE` from the
ClusterServiceVersion on the operator's deployment) will be used to retrieve the
ClusterTasks via an [initContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
and those ClusterTasks will have the image overridden as well as have
build annotations added. This way, we will have the ability to build compatibility
matrixes of which versions of Crane Runner + Crane UI + Crane Proxy go together
and can be managed by the operator (probably encoded in the form of a
ClusterServiceVersion published to https://operatorhub.io/ ) and we'll be able
to inspect these things when debugging by checking the ClusterTasks.

## Alternatives

### Variable File

One alternative that was explored revolved around storing the results of the
Crane UI wizard in a "variable file" ConfigMap. Additionally, the UI would
generate a Pipeline that when run the first Task would consume the "variable file" ConfigMap
containing all of the relevant information (ie. source namespace, cluster
coordinates and credentials, and workspaces).

The main problem with this approach is there is no mechanism for a TaskRuns in a
Pipeline to map parameters or workspaces into subsequent TaskRuns. Pipelines
are responsible for handling parameters + workspaces and mapping those onto the
Tasks contained therein. Valid PipelineRuns must specify the parameters +
workspaces expected by the Pipeline.

### Generating Pipelines

Similar to the [Variable File](###variable-file) alternative -- both mention
creating a generic Pipeline -- the thrust of this alternative was to simply
define a Pipeline such that the User could "Run" (create a PipelineRun) later.
The goal would be to have all of the parameters defined as defaults to
facilitate the user's execution without having to provide new information.
Additionally, this would avoid some of the UI problems associated with starting
a PipelineRun.

There are several reasons why this route was not chosen but the single most
important reason was from [Tekton Docs on PipelineRuns](https://tekton.dev/docs/pipelines/pipelineruns/#pending-pipelineruns).

> A PipelineRun can be created as a “pending” PipelineRun meaning that it will not actually be started until the pending status is cleared.

Second, the UI is creating an instance of a migration and `Pipeline`s are
intentionally distinct from `PipelineRun`s in [Tekton's documentation](https://tekton.dev/docs/pipelines/pipelineruns/#overview).

> A PipelineRun allows you to instantiate and execute a Pipeline on-cluster.
> A Pipeline specifies one or more Tasks in the desired order of execution.
> A PipelineRun executes the Tasks in the Pipeline in the order they are specified until all Tasks have executed successfully or a failure occurs.

While we could potentially store all relevant information for a `PipelineRun`
on a `Pipeline`, these resources are distinct and making one look like the other
could cause us problems in the future.

Lastly, and by far the biggest technical hurdle that prevented us from seriously
considering this path was the handling Workspaces. Consider this section from
Tekton's documentation about [Specifying `Workspaces`](https://tekton.dev/docs/pipelines/pipelineruns/#specifying-workspaces)
with respect to `PipelineRun`s:

> If your Pipeline specifies one or more Workspaces, you must map those Workspaces to the corresponding physical volumes in your PipelineRun definition.

This opened a whole new set of open questions: do we create PVC(s) for the user?
do we tell the user to use a claim template? Any way we approach this, the user
would be required, at the time they decide to run the `Pipeline`, the correct
mapping of `Volumes` to `Workspaces`. The purpose of our approach was to
simplify this for users as much as possible and this was an unacceptable
trade-off.

### UI Generated PipelineRuns from Typed Config

Another alternative that was considered was developing a typed config that could
be stored as a ConfigMap and Crane UI would use these ConfigMaps to generate
PipelineRuns at the user's discretion.

The problem with this approach is that the Crane UI's focus is on the wizard and
creating a whole new user experience around generating PipelineRuns from the
Crane UI is out of scope for GA. The currently proposed solution of creating
PipelineRuns directly does not prevent us from pursuing this option in the
future.

### Releasing ClusterTasks in OCI Bundle Format

[Tekton Bundle Contract v0.1](https://tekton.dev/docs/pipelines/tekton-bundle-contracts/)

Research was done into whether or not we could leverage Tekton Bundles. The
benefit of Tekton Bundles is that the Tasks **and Pipelines** included can be
used across the cluster which could have allowed the UI to only be concerned
with referencing the correct Pipeline for a PipelineRun.

The limitations of these Tekton Bundles (only 10 Pipelines|Tasks) and the
inability to handle this in downstream pipelines made it an unsuitable option to
pursue. It is worth noting that the Tekton community appears to be pushing the
Tekton Bundle (see [Add Cluster scope pipeline support](https://github.com/tektoncd/pipeline/issues/1876)
and [Versioning on Tasks/Pipelines](https://github.com/tektoncd/pipeline/issues/1839).
