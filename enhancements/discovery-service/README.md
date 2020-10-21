---
title: Discovery Service
authors:
  - "@jortel"
reviewers:
  - "@djwhatle"
  - "@eriknelson"
approvers:
  - "@djwhatle"
  - "@eriknelson"
creation-date: 2019-12-04
last-updated: 2020-10-21
status: implemented

---


# Discovery Service

## Background

The CAM application controller reconciler watches MIG CRs and ensures that source and destination clusters are as declared in the Spec. In addition, the controller discovers information that is used by the controller and provided to the user. This (currently) includes:

- MigCluster (performed: every reconcile)
  - Storage Classes
- MigPlan (performed: each reconcole when open, not-suspended)
  - PVs
  - Unhealthy Pods (source)
- MigMigration (performed: phase=Verification)
  - Unhealthy Pods (destination)

The discovery is performed by querying resources on each cluster as needed and the data is stored on the CR. This approach is reasonable for relatively small data sets. Further, the PVs are merged to the MigPlan spec to provide the user with a _fill in the blank_ experience. The user selection is needed & preserved by the controller. For large data sets (Eg: 100k namespaces), this approach has the following pros & cons:

Pros:

- The discovery is performed during reconcile and data stored on the CR which makes it available to user/UI using standard kubernetes mechanisms and paradigms.

Cons:

- The discovery is performed during reconcile which may be triggered by changes to watched MIG and standard kube resources. As a result, the discovery frequency and interval is excessive. Further, since the reconciler does not know which watched resource has changed, the controller queries all of the resources to be discovered. This has significant performance and cluster load drawbacks.
- The data is stored on the CR.
  - This _feels_ like a misuse of etcd though has not been benchmarked or confirmed.
  - Bloats the size of CR document.
  - Does not support pagination.

## Use Cases

Immediate:

- As the UI, I want to get a paginated list of all namespaces on a cluster without configuring CORS to query the cluster directly.

Future (possibilities):

- As the plan controller, I want to get a list of namespaces to support validations without querying all of the namespaces on every reconcile.
- As the plan controller, I want to get a list of all Pods, PVs and PVCs within specified namespaces to support PV discovery without querying the cluster.
- As the plan controller, I want to get a list Storage class on cluster to support validation and PV supported choices without querying the cluster in every MigCluster reconcile.
- As the plan controller, I want to get a list of unhealthy namespaces on the source cluster to set/clear a warning condition.

## Proposal

The proposed solution is a separate discovery service/controller that maintains an inventory of resources discovered on clusters represented by MigCluster. When a new MigCluster is created, the discovery controller will discover (query) relevant resources and add them to the DB. Then, setup remote watches on those resources for incremental changes. A REST API will provide access to _canned_ reports.

## Architecture

Discovery controller that is either part of the _migration-controller_ manager Pod/container or as a separate Pod and/or container. Discovered resources are stored in a _sqlite3_ DB that is optionally backed by an EmptyDir volume. The goal of backing with a volume is to preserve the inventory across Pod restart which will minimize startup time. An embedded (gin) REST API provides reporting on warehoused data. The API supports CORS and pagination.


## Design Concepts (abstractions):

- Container - The _container_, contains and manages a set of DataSources. When a MigCluster is created, updated or deleted, a corresponding data-source is created, updated or deleted.
- DataSource - A _data-source_ is responsible for managing the inventory for a cluster. It has a corresponding _controller-runtime_Manager that is used for remote watches. Each data-source contains and manages a set of Collections.
- Collection - A _collection_ corresponds (loosely) to a k8s resource Kind. It performs the actual discovery and maintains the inventory of that kind in the DB. An initial discovery (reconcile) is performed upon creation. Then, it registers watches with its DataSource. Collections implement the Predicate interface and apply real-time changes to the DB.
- WebServer - The web server provides a REST API. Each request is delegated to a RequestHandler.
- RequestHandler - Each handler supports GET requests for s specific REST resource.
- Model - The _model_ is a very lightweight abstraction of the DB.

## API

### Concepts

Cluster scoped - All data is scoped to a cluster (MigCluster).

Pagination - All GET requests support pagination by passing the offset and limit parameters.

Availability - To protect against returning partial datasets, each collection must be initially/completely reconciled prior to REST API fulling a GET request on the collection. A 206 _RequestTimeout_ is returned when data is not yet available.

Eventual Consistency - After a collection is initially reconciled (available), remote watches are used to add/update/delete individual resources. This has been tuned to be very fast but the kubernetes informer subsystem has some inherent latency. As a result, a GET on a collection will represent a snapshot in time but will eventually become consistent with the inventory on the cluster.

Standard parameters:

- offset - The offset of the first item to be included (default: 0).
- limit - The number of items returned (default: unlimited).

### Headers:

- Authorization - Contains the bearer token.

### Resources:

Endpoints:

- /schema - Show schema.
- /namespaces/ - List top level namespaces.
- /namespaces/<name>/clusters/<name>/ - Get clusters by _name_ else all clusters.
  - namespaces/ - All namespaces on a cluster.
    - Params: none
    - Returns: Collection with resources of []Cluster.
  - /namespaces/<name>/pods/<name> Get pods by _name_ else all pods.
    - Params: none
    - Returns: Pod
  - /namespaces/<name>/pods/<name>/logs - Get pod logs.
    - Params:
      - tail - (bool) Get logs from the _tail_. (defail: false)
      - container - A container name.
    - Returns: []string
  - persistentvolumes/<name> - Get PV by _name_ else all on the cluster.
    - Params: none
    - Returns: Collection with PV of []Namespace
- /namespaces/<name>/plans/<name>/ - Get Migration plan by _name_ else all plans.
  - pods/ - Get the plan pod report.
    - Params: none
    - Returns: _PlanPods_

Types:

The overall pattern is for each discovery service resource (type) to provide both the raw k8s resource (object) and value-add fields. The object field is the standard field for the k8s type/CR. To support use cases whereby only the value-add fields are needed by the caller, the object field could be omitted using a query parameter making the payload lighter.

The Collection will be returned for collection endpoints instead of a json Array.

- Collection (List):
  - count (int64)
  - resources ([]Resource)
- Namespace:
  - name (string)
  - bject (v1.Namespace)
  - serviceCount: (int)
  - podCount (int)
  - pvcCount (int)
- Pod:
  - namespace (string)
  - name (string)
  - bject (v1.Pod)
- Service:
  - namespace (string)
  - name (string)
  - bject (v1.Service)
- PV:
  - names (string)
  - bject (v1.PersistentVolume)
- PVC:
  - namespace (string)
  - name (string)
  - bject (v1.PersistentVolumeClaim)
- Plan:
  - namespace (string)
  - name (string)
  - bject (MigPlan)
- Cluster:
  - namespace (string)
  - name (string)
  - bject (MigCluster)
- PlanPods:
  - _Container <type>_:
    - name (string)
    - log: (string) URI to get logs.
  - _Pod <type>_:
    - namespace (string)
    - name (string)
    - containers ([]PlanPods.Container)
  - namespace (string)
  - name (string)
  - controller: ([]PlanPods.Pod]
  - source: ([]PlanPods.Pod]
  - destination: ([]PlanPods.Pod]


## Authorization

Requests are authorized using the `Authorization: Bearer` token request header. The environment variable `AUTH_OPTIONAL=1` can be used for easier development. When set, the token is not required but will be used when provided.


## Resources

Resource hierarchy (scoping) follows kubernetes with cluster at the top. So, namespaces are nested under cluster, pods are nested under namespace etc … The _owner_ relationship will also be reflected in the inventory.

Immediate:

- PersistentVolumes
- Pods
- Pods/log

Future (possibilities):

- Storage classes
- Services
- Pods nested under _owner_ resources (Eg: Deployment). This includes health status.