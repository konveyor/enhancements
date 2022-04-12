---
title: Release process for plugins and keeping Crane-plugins upto date with all releases
authors:
  - "@jgabani"
reviewers:
  - "@shawn-hurley"
  - "@sseago"
  - "@djzager"
approvers:
  - "@shawn-hurley"
  - "@sseago"
  - "@djzager"
creation-date: 2021-11-05
last-updated: 2022-04-04
status: implementable
see-also:
  - "N/A" 
replaces:
  - [Initial approach](README.md)
superseded-by:
  - "N/A"
---

# Release process for plugins and keeping Crane-plugins upto date with all releases

This document proposes a redesign of release process as well as re-structure of `crane-plugins` a little to accommodate simplified process of plugin release. The motivation behind achieving this simplicity is to make release process simpler for the plugin author.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

Not needed

## Summary

We are trying to make release process simpler and decouple the need of updating index file with individual releases of each plugin so the changes needed in `crane-plugins` to include new releases of any plugins could be limited to respective plugin's manifest file. Alongside making the process simpler to accommodate this simplicity, we need to restructure `crane-plugins`.

<b> Relevant repositories </b>

- [crane-plugins](https://github.com/konveyor/crane-plugins): repository where all the plugin manifest files are present
- [crane-plugin-<plugin-nam>]: a plugin repository where individual plugin code lives. (For example, [openshift]((https://github.com/konveyor/crane-plugin-openshift)) plugin)  

## Motivation

As of now to release a plugin we need to have at least two PRs in `crane-plugins` one against `main` branch and another one against `crane-cli` branch, and needs to modify two files, `index.yaml` and respective manifest of each plugin. This document aims to reduce the number of steps needed to release a plugin. 

### Goals

- We have a standardized process to release plugins 
- Simplify current process of releasing plugins
- Define changes in the repository for each plugin and `crane-plugins` in order to achieve the simplification.

## Proposal

The `index.yaml` in `crane-plugins` will only include mapping of a plugin with respective location of manifest (url/file path).

```
apiVersion: crane.konveyor.io/v1alpha1
kind: PluginIndex

plugins:
- name: OpenShiftPlugin
  path: https://github.com/konveyor/crane-plugins/raw/main/plugins/OpenShift/index.yaml

```

There will be a single manifest file for each plugin. Each of the manifest file of respective plugin will include all the information of every release of the plugin. Reformated manifest file would look something like,

```
apiVersion: crane.konveyor.io/v1alpha1
kind: Plugin

versions:
- name: OpenShiftPlugin
  shortDescription: OpenshiftPlugin
  description: this is OpenshiftPlugin
  version: v0.0.1
  binaries:
    - os: linux
      arch: amd64
      uri: https://github.com/konveyor/crane-plugin-openshift/releases/download/v0.0.1/amd64-linux-openshiftplugin-v0.0.1
    - ...
  optionalFields:
    - flagName: "strip-default-pull-secrets"
      help:     "Whether to strip Pod and BuildConfig default pull secrets (beginning with builder/default/deployer-dockercfg-) that aren't replaced by the map param pull-secret-replacement"
      example:  "true"
    - ..
- name: OpenShiftPlugin
  shortDescription: OpenshiftPlugin
  description: this is OpenshiftPlugin
  version: v0.0.2
  binaries:
    - os: linux
      arch: amd64
      uri: https://github.com/konveyor/crane-plugin-openshift/releases/download/v0.0.1/amd64-linux-openshiftplugin-v0.0.2
    - ...
  optionalFields:
    - flagName: "strip-default-pull-secrets"
      help:     "Whether to strip Pod and BuildConfig default pull secrets (beginning with builder/default/deployer-dockercfg-) that aren't replaced by the map param pull-secret-replacement"
      example:  "true"
    - ..
```


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

      - name: Releasing artifacts
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
          path: crane-plugins

      - name: setup python
        uses: actions/setup-python@v2
        with:
          python-version: 3.8

      - name: Updating index file and adding manifest
        shell: bash
        run: |
          pip install pyyaml
          cd crane-plugins
          python ~/work/${{ github.event.repository.name }}/${{ github.event.repository.name }}/main/.github/workflows/script.py "${{ github.event.inputs.version }}" "${{ github.repository_owner }}" "${{ github.repository }}"
     
      - name: Create Pull Request against crane-plugins
        uses: peter-evans/create-pull-request@v3
        with:
          token: ${{ secrets.PLUGIN_RELEASE }}
          commit-message: Updating index and adding manifest from openshift plugin release
          title: Adding openshift plugin from release ${{ github.event.inputs.version }}
          body: Update index and add manifest to include version ${{ github.event.inputs.version }} of openshift plugin
          branch: OpenShiftPlugin
          base: main
          path: crane-plugins
```

The above workflow raises a PR against [crane-plugins>](https://github.com/konveyor/crane-plugins) that updates the `\<plugin-name>\<yaml>` adding the latest released version in `versions`. 

The `script.py` used to modify yaml files would look something like this - 

```python
import os
import yaml
import sys

os.makedirs('plugins/OpenShift', exist_ok=True)

if os.path.exists("index.yml"):
    file = open("index.yml","r")
    not_present = 1
    index = yaml.safe_load(file)
    for plugin in index['plugins']:
        if plugin['name'] == 'OpenShiftPlugin':
            not_present = 0
            break
    if not_present:
        index['plugins'].append({"name": "OpenShiftPlugin", "path": "https://github.com/%s/crane-plugins/raw/main/plugins/OpenShift/index.yaml"%sys.argv[1]})
    file = open("index.yml","w")
    yaml.dump(index, file)
    file.close()

else:
    file = open("index.yml","a+")

    index = yaml.safe_load(file)

    index = {}
    index['kind'] = 'PluginIndex'
    index['apiServer'] = 'crane.konveyor.io/v1alpha1'
    index['plugins'] = []

    index['plugins'].append({"name": "OpenShiftPlugin", "path": "https://github.com/%s/crane-plugins/raw/main/plugins/OpenShift/index.yaml"%sys.argv[2]})

    yaml.dump(index, file)
    file.close()

# create or append in plugin index
if os.path.exists('plugins/OpenShift/index.yml'):

    file = open("plugins/OpenShift/index.yml","r")

    index = yaml.safe_load(file)

    index['versions'].append({})
    index['versions'][-1] = {
        'name': 'OpenShiftPlugin',
        'shortDescription': 'OpenShiftPlugin',
        'description': 'this is OpenShiftPlugin',
        'version': sys.argv[1],
        'binaries': [
            {
                'os': 'linux',
                'arch': 'amd64',
                'uri': "https://github.com/%s/releases/download/%s/amd64-linux-openshiftplugin-%s"%(sys.argv[3], sys.argv[1],sys.argv[1]),
            },
            {
                'os': 'darwin',
                'arch': 'amd64',
                'uri': "https://github.com/%s/releases/download/%s/amd64-darwin-openshiftplugin-%s"%(sys.argv[3], sys.argv[1],sys.argv[1]),
            },
            {
                'os': 'darwin',
                'arch': 'arm64',
                'uri': "https://github.com/%s/releases/download/%s/arm64-darwin-openshiftplugin-%s"%(sys.argv[3], sys.argv[1],sys.argv[1]),
            },
        ],
        'optionalFields': [
            {...},
            {...},
            {...}
        ]
    }

    file = open("plugins/OpenShift/index.yml","w")

    yaml.dump(index, file)
    file.close()
    
else:
    file = open("plugins/OpenShift/index.yml","a+")

    index = yaml.safe_load(file)

    index = {}
    index['kind'] = 'Plugin'
    index['apiServer'] = 'crane.konveyor.io/v1alpha1'
    index['versions'] = []

    index['versions'].append({})
    index['versions'][0] = {
        'name': 'OpenShiftPlugin',
        'shortDescription': 'OpenShiftPlugin',
        'description': 'this is OpenShiftPlugin',
        'version': sys.argv[1],
        'binaries': [
            {
                'os': 'linux',
                'arch': 'amd64',
                'uri': "https://github.com/%s/releases/download/%s/amd64-linux-openshiftplugin-%s"%(sys.argv[3], sys.argv[1],sys.argv[1]),
            },
            {
                'os': 'darwin',
                'arch': 'amd64',
                'uri': "https://github.com/%s/releases/download/%s/amd64-darwin-openshiftplugin-%s"%(sys.argv[3], sys.argv[1],sys.argv[1]),
            },
            {
                'os': 'darwin',
                'arch': 'arm64',
                'uri': "https://github.com/%s/releases/download/%s/arm64-darwin-openshiftplugin-%s"%(sys.argv[3], sys.argv[1],sys.argv[1]),
            },
        ],
        'optionalFields': [
            {...},
            {...},
            {...}
        ],
    }
    
    yaml.dump(index, file)
    file.close()
```

The next step would be to review this PR and merge it to the `main` branch of `crane-plugins`.



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

1. Plugin author has to make sure that there are no bugs and gaps in the new plugin release that is intended.
2. This redesign is not backward compatible.

## Alternatives

To overcome the drawback of the solution below are other approaches.

### Alternative 1

[This](README.md) implementation of crane plugins allow us to test all the plugin while they are not GA but released through `main` branch and then after making everything sure we make them GA through`crane-lib` branch

### Alternative 2

Introduce ci to include tests that make sure that any incoming change in respective plugin through PRs are tested thoroughly.