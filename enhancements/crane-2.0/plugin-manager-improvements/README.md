---
title: crane-kustomizate-support
authors:
  - jmontleo
reviewers:
  - "@shawn-hurley"
  - "@sseago"
  - "@djzager"
  - "@JaydipGabani"
approvers:
  - "@djzager"
  - "@JaydipGabani"
creation-date: 2022-07-12
last-updated: 2022-07-12
status: completed
see-also:
  - https://github.com/konveyor/crane/issues/89
  - https://github.com/konveyor/crane/pull/137
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane Plugin Manager Improvements

This document covers proposed improvements to `crane plugin-manager`.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Crane plugin manager uses the relative path to install plugins and an issue was filed to change the behavior. Preserving the existing behavior in some capacity would be beneficial for running within container

## Motivation

The current behavior works well for container installs. When a WORKINGDIR is specified for the image and the plugin is installed, then regardless of which user the image is run as the plugin is available where expected.

This is however not ideal on a users workstation where they may want to install crane and the plugin once and then traverse directories to perform migrations.

### Goals

To provide a solution that can benefit both cases.

## Proposal

- Allow looking for plugins in multiple locations
  - `./plugins` - relative path
  - `~/.local/share/crane/plugins` - Default install location when running `crane plugin-manager add`
  - `/usr/local/share/crane/plugins` - For non-packaged global installs by an admin on shared systems
  - `/usr/share/crane/plugins` - Install by a package management system (dnf, yum, apt, etc.)
- `crane plugin-manager add` would change it's behavior to install in `~/.local/share/crane/plugins`, unless `--global` is specified in which case it will install to  `/usr/local/share/crane/plugins`.
- Installation to other locations could be facilitated by placing the plugin binary in that location either manually, using the `-p` option, or by package installation.
- `-p --plugin-dir` will override the `~/.local/share/crane/plugins` path in all cases.
- The `managed/default` portion of the plugins directory structure will be removed.
  - We will only look for executable plugins in the base directory.
  - It doesn't seem well understood what the intention is and will likely lead to confusion when loading plugins from subdirectories.
  - In its current incarnation copying plugins/managed to plugins/unmanaged creates a duplicate plugin entry.

### Implementation Details/Notes/Constraints

The conflict resolution is simple and follows the `./plugins` -> `~/.local/share/crane/plugins` (or overriden path) -> `/usr/local/share/crane/plugins` -> `/usr/share/crane/plugins`.

While we could look for and use an instance of the newest version this could lead to a scenario on a shared system where an admin has done a global and/or package install with the newest version, but a user discovers a regression and wants to use a prior version. By doing it this way we empower them to do so.

### Test Plan

- Test container runs with the plugins in the pwd
- Test on a workstation install with a home dir based bath
- Test resolution of multiple plugin versions
