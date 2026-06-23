---
title: hub-snapshots
authors:
  - "@mansam"
  - "@jortel"
reviewers:
  - "@jortel"
  - "@shawn-hurley"
  - "@aufi"
  - "@dymurray"
approvers:
  - TBD
creation-date: 2024-08-26
last-updated: 2024-08-26
status: implementable
see-also:
 - https://github.com/konveyor/tackle2-hub/issues/565
---

# Hub Snapshots

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions


## Summary

Introduce a way to take a snapshot of a running Konveyor Hub instance, including
its database and bucket contents, and package it in a portable way. These snapshots are stored on a volume
accessible to the Hub, and can be restored to the same Hub, or downloaded and imported into another
Hub instance-- even one that has a newer database version than the source Hub. Restoring a snapshot
should be as easy as choosing one out of a list and pressing "deploy".

## Motivation

This system fills a number of related purposes. First, it provides a general backup
and restore solution, in the form of portable snapshots that can be deployed with all
database and artifact relationships in place. Second, it provides a tool that can
be used by support personnel to examine a nearly exact replica of an end-user's Hub
instance. Third, it provides a way to prime a Hub instance for demos and workshops, including
quickly switching between various pre-prepared scenarios (different snapshots) and rolling back
to retry a workshop.

### Goals

* Allow administrative users to create and manage snapshots of a running Hub instance from the API
and UI.
* Allow administrative users to see a list of stored snapshots with details about creation
date, size, database version, etc.
* Allow administrative users to export snapshots encrypted by a passphrase.
* Allow administrative users to import external snapshots, that should appear on the list of
stored snapshots.
* Allow administrative users to delete snapshots.
* Allow administrative users to rapidly deploy a snapshot from the list of available snapshots.
* A snapshot should be importable into any newer Hub version, whereupon deployment the database
should be automatically migrated to match the current version.
* The Hub should recognize a malformed snapshot manifest or an incompatible snapshot version and
refuse to restore it.
* The snapshot API should be operational even if the rest of the Hub is unable to come up due to
database damage or some other problem.

### Non-Goals

* Scheduled snapshots.
* Automatic cleanup of old snapshots.
* Importing snapshots into Hubs that are older than the snapshot version.
* Integration with any external tools.

All non-goals other than backporting snapshots could be explored at a later date after
the fundamentals of the feature are accepted and implemented.

## Proposal


### User Stories

* As an administrator, I want to backup a running Hub instance.
* As an administrator, I want to restore a specific snapshot of the Hub.
* As an administrator, I want to list, create and delete snapshots.
* As an administrator, I want to import and export snapshots that have been
protected by passphrases.
* As an administrator, I want to demo using a (choice of) snapshots of the Hub.

### Implementation Details

#### Quiescing the Hub

The Hub consists of a main thread serving the API, as well as a number of background
goroutines that handle periodic tasks. The majority of these read from and write to
the Hub DB and must be quiesced before a copy of the database can be made.

A new subsystem will be added to manage the lifecycle of the API server and all of
the other background tasks. When a snapshot is to be created or deployed, the lifecycle
manager will quiesce the API (other than the snapshot API) and the rest of the background
tasks and close the database handle before permitting the snapshot process to proceed. If
they cannot be shutdown properly, snapshotting will be aborted. 

After creating the snapshot
or deploying the snapshot has completed, the lifecycle manager will restart the API server
and the rest of the background tasks so that normal Hub operations can resume.

#### Snapshot Anatomy (gzipped tarball):

    /snapshot.yaml - manifest YAML file about the snapshot.
        name: The name of the snapshot.
        key: (optional) The encryption key (encrypted using a password).
    /hub.db - The sqlite3 DB file.
    /bucket - The bucket tree.

#### Workflows:

List

```
route: GET /snapshots
returns: A list of created or uploaded snapshots. (Listing of name: found in manifests).
```

Export

```
route: POST /snapshots/export
body: A password used to encrypt the encryption key.
returns: streamed snapshot (tarball).
```

Create

```
route: POST /snapshots/:name
actions:

    Create /snapshot/uuid/snapshot.tar.gz
```

Import

```
route: POST /snapshots/import/:name
body: multi-part file upload of a tarball and the password used to encrypt the encryption key.
action:
    Store the uploaded in snapshot (tarball) in /snapshot/uuid
    Extract /snapshot.yaml and decrypt the key using the password.
    Encrypt key in /snapshot.yaml using the (local) encryption key and update the extracted file.
```

Deploy

```
route: POST /snapshot/deploy/:name
action: Create a symlink /snapshot/deploy => the snapshot (directory) to be deployed.
```

### Security, Risks, and Mitigations

The Hub database contains sensitive credentials that are encrypted by a key unique to that
instance of Konveyor. This means that the snapshots will contain these encrypted credentials,
and they will only be able to be unencrypted on a Hub using the same key.

In order to export the snapshot to another Hub, the encryption key needs to come with it. When exporting a snapshot
for use in another Hub, a passphrase will be required to encrypt the key so that it is not exposed.
On import, the passphrase will be required to decrypt the key so that the snapshot can be applied.
(The encryption key from that snapshot would then replace the one previously used by the Hub).

## Design Details

### Test Plan

Integration tests will be necessary to ensure that the process of quiescing background tasks,
taking snapshots, restoring snapshots, and restarting background tasks work as expected.


### Upgrade / Downgrade Strategy

Snapshots are versioned with the database version of the Hub they were created from. Downgrading
to an older Konveyor version will make the snapshots created with the newer version usuable with that Hub,
but they could still be deployed elsewhere.

Old snapshots can be deployed even into a newer hub, and the contents of the snapshot are automatically
upgraded via the existing database migration mechanism when they are deployed. The snapshot itself
is not altered.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

* It is not possible to perform a partial export of resources from the Hub with this method. A snapshot
is all or nothing.

* It is not easy to alter the contents of a snapshot on disk, for instance to prepare a demo or a test case.
The snapshot needs to be loaded into a Hub, then any changes for the demo scenario can be made and a new snapshot can be exported.

## Alternatives

Expanding the range of the Tackle import tool could cover a fairly large amount of this use case,
but it is limited to API-compatible resource versions and would require a substantial amount of
maintenance to keep it up to date with changing Hub versions compared to taking
database snapshots.
