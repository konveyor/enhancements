---
title: Release process for plugins and keeping Crane-plugins upto date with all releases
authors:
  - "@jgabani"
reviewers:
  - "@shawn-hurley"
  - "@sseago"
approvers:
  - "@shawn-hurley"
  - "@sseago"
creation-date: 2021-11-05
last-updated: 2021-11-05
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Release process for plugins and keeping Crane-plugins upto date with all releases

This document proposes a solution to setup a release cycle for all the `Crane` plugins managed by us. Alongside the approaches to keep `crane-plugins` (all the manifests files for the plugins) upto date.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

Not needed

## Summary

We are trying to define a standardized release process for plugins including ways to keep the `crane-plugins` repository upto date with all the plugins releases.

<b> Relevant repositories </b>

- [crane-plugins](https://github.com/konveyor/crane-plugins): repository where all the plugin manifest files are present
- [crane-plugin-<plugin-name>](https://github.com/konveyor/crane-plugin-openshift): a plugin repository where individual plugin code lives. (For example, openshift plugin)  

## Motivation

With the release of crane-lib and crane, there is possible need to release compatible plugins. To make the `crane plugin-manager` work properly, we need to have all the `manifest` files in the default `crane-plugin` repository  upto date. Since there are multiple repositories included (crane-plugins, repositories for each plugin) the release process becomes complicated. Goal of this document is to define a streamlined release process for plugins and a way to make sure that `crane-plugins` repository remains upto date with released plugins.  

### Goals

- We have a standardized process to release plugins
- We have a defined set of actions to update `crane-plugins` repository with the manifests files of plugins

### Non-Goals

Not applicable

## Proposal

We setup a release process for an individual plugin with github workflows to release binaries of plugins for desired os/arch to be available for `crane` user.

The `release.yml` look something like this - 

```
name: Releases
on:
  workflow_dispatch:
    inputs:
      version:
        description: Bump Version
        default: v0.0.1
        required: true
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Set up Go 1.16
        uses: actions/setup-go@v2
        with:
          go-version: 1.16

      - name: Check out source code
        uses: actions/checkout@v2
        with:
          path: main
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Tidy
        env:
          GOPROXY: "https://proxy.golang.org"
        run: cd main && go mod tidy && git diff --quiet HEAD

      - name: Build plugin linux amd64
        env:
          GOPROXY: "https://proxy.golang.org"
        run: cd main/ && GOOS=linux GOARCH=amd64 go build -o ~/main/bin/amd64-linux-<plugin>-${{ github.event.inputs.version }} .

      - name: Build plugin darwin amd64
        env:
          GOPROXY: "https://proxy.golang.org"
        run: cd main/ && GOOS=linux GOARCH=amd64 go build -o ~/main/bin/amd64-linux-<plugin>-${{ github.event.inputs.version }} .

      - name: Build plugin darwin arm64
        env:
          GOPROXY: "https://proxy.golang.org"
        run: cd main/ && GOOS=linux GOARCH=amd64 go build -o ~/main/bin/amd64-linux-<plugin>-${{ github.event.inputs.version }} .

      - name: release
        uses: ncipollo/release-action@v1
        with:
          artifacts: "~/main/bin/*"
          token: ${{ secrets.GITHUB_TOKEN }}
          tag: ${{ github.event.inputs.version }} 

      - name: Checkout crane-plugins
        uses: actions/checkout@v2
        with:
          repository: ${{ github.repository_owner }}/crane-plugins
          token: ${{ secrets.PLUGIN_RELEASE }}

      - name: Updating index file and adding manifest
        run: |
          cat << EOF >> index.yml
          <plugin>-${{ github.event.inputs.version }}: https://github.com/${{ github.repository_owner }}/crane-plugins/raw/main/plugins/<plugin>/<plugin>-${{ github.event.inputs.version }}.yml
          EOF
          mkdir -p plugins/<plugin>
          cat << EOF >> plugins/<plugin>/<plugin>-${{ github.event.inputs.version }}.yml
          name: <name>
          shortDescription: <short description>
          description: <description>
          version: ${{ github.event.inputs.version }}
          binaries:
            - os: linux
              arch: amd64
              uri: https://github.com/${{ github.repository }}/releases/download/${{ github.event.inputs.version }}/amd64-linux-<plugin>-${{ github.event.inputs.version }}
            - os: darwin
              arch: amd64
              uri: https://github.com/${{ github.repository }}/releases/download/${{ github.event.inputs.version }}/amd64-darwin-<plugin>-${{ github.event.inputs.version }}
            - os: darwin
              arch: arm64
              uri: https://github.com/${{ github.repository }}/releases/download/${{ github.event.inputs.version }}/arm64-darwin-<plugin>-${{ github.event.inputs.version }}
          optionalFields:
            - flagName: <flag name>
              help:     <help>
              example:  <example>
            - flagName: <flag name>
              help:     <help>
              example:  <example>
            - flagName: <flag name>
              help:     <help>
              example:  <example>
          EOF

      - name: Create Pull Request against crane-plugins
        uses: peter-evans/create-pull-request@v3
        with:
          token: ${{ secrets.PLUGIN_RELEASE }}
          commit-message: Updating index and adding manifest from <plugin> plugin release
          title: Adding <plugin> plugin from release ${{ github.event.inputs.version }}
          body: Update index and add manifest to include version ${{ github.event.inputs.version }} of <plugin> plugin
          branch: <plugin>
          base: main

```

The above workflow raises a PR against [crane-plugins](https://github.com/konveyor/crane-plugins) that updates the `index.yml` adding the latest release of the plugin and adds a `manifest` file for the plugin at appropriate location within `crane-plugins`.

The next step would be to review this PR and merge it to the `main` branch of `crane-plugins`.

And the final step after testing the changes of `main` branch would be to release these manifest files in to `crane-cli` branch, for that we would need to cherry pick a tested commit and change `url` of manifest files in `index.yml` (i.e changing from `main` to `crane-cli` in index.yml for every entry)


### User Stories [optional]

In this context user will mean a team member handling release for plugins.

#### Story 1

As a user, I want to release a plugin.

#### Story 2

As a user, I want to make sure that all the plugins released are available for `crane plugin-manager` cli.

### Risks and Mitigations

N/A

## Design Details

### Test Plan

N/A

### Upgrade / Downgrade Strategy

N/A

## Drawbacks

1. We need to update `release.yml` file whenever `metadata` changes for a plugin (i.e optional field, description, etc)
2. Manual merge for any PR against `crane-plugin` is required (at the time of multiple plugin release at the same time it may require additional attention to resolve conflicts amongst PRs)

## Alternatives

To overcome the drawback of the solution below are other approaches. The below are the alternatives for the process of manually raising PR against `crane-plugins` and not the alternatives of whole solution proposed. We want to have a release workflow for the plugin.

### Alternative 1

We can set up a webhook in plugin repository that sends a request to a CI (potentially configured in `crane-plugins`) after release, this event will causes CI to update the `index.yml` file and add a manifest file for the plugin in the main branch.

The flow of events would look something similar to below in this case:

1. We release a plugin (for example, we release plugin `foo`)
2. Github webhook is triggered from a plugin repository which sends a request to CI configured in `crane-plugins`. (for example, after release of plugin `foo`, the webhook configured in `foo` plugin repository would send a request to CI configured in `crane-plugins`)
3. CI configured in `crane-plugins` receives the request with payload info and updates `index.yml` file and add a manifest file for released plugin version.
