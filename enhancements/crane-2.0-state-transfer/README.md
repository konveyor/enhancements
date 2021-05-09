---
title: Crane 2.0 State Transfer
authors:
  - "@jmontleon"
reviewers:
  - "@shawn-hurley"
  - "@alaypatel07"
approvers:
  - "@shawn-hurley"
creation-date: 2021-05-05
last-updated: 2021-05-05
status: implementable
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane 2.0 Discover and Extract

The Crane 2.0 effort is focused on creating a library to enable rapid
development of utilities to help users backup and restore applications deployed
on Kubernetes. This document describes the goals of one component of the library
being developed to enable state transfer between clusters.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined

## Summary 

At a high level this library will
- Handle basic quiesce functionality by scaling down and rescheduling resources.
- Consist of reusable code to orchestrate transfer of PVC data between clusters
- Enable transfer between kubernetes, openshift, and variants (EKS, ROSA, etc.)
- Be capable of using different endpoints (Route, Load Balancer, Ingress.)
- Support rsync, but allow implementing alternatives (rclone, restic, etc.)
- Enable transfer without escalated privileges
- Provide functionality for repairing permissions when moving from kubernetes

## Motivation

In other tools enabling transfer of data between clusters common issues include:
- A requirement for escalated or admin privileges
- Inability to transfer from kubernetes and other variants

With recent PoC work we have proven that these requirements are artificial.

## Non-goals

This library will be focused on transfering state container within PVCs. We
will not attempt to transfer state contained in:
- ConfigMaps
- Secrets
- CustomResources

## Proposal

- Create a library with interfaces to enable transfer of data between clusters
- The current WIP progress implementation provides for three interfaces to build a connection
  - Transfer: Defines a transfer program such as rsync or rclone
  - Transport: Defines a proxy for encrypting traffic. Currently stunnel is the only option.
    - In case a transfer utility provides its own encryption a null transport is provided.
  - Endpoint: Route, Load Balancer, or another alternative for exposing a server for sync.
- The library will use a PVC as the basic unit to transfer.
  - It will be possible to transfer PVCs concurrently.
- The library will make it easy to run against all PVCs in one or more project
- The library will accept two rest configurations for client and server
- Provide a CLI tool demonstrating how to implement functionality
