---
title: Manage custom plugin repository
authors:
  - "@jgabani"
reviewers:
  - "@sseago"
approvers:
  - "@sseago"
creation-date: 2021-11-23
last-updated: 2021-11-23
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
---

# Manage custom plugin repository

This document proposes a solution to manage custom plugin repositories. The document describes the needed changes in current implementation, and all the sub-commands that are needed for this feature.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- Do we want the user to have the control to override/remove default plugin source?

## Summary

Along with the our own default plugin-repository, we would like to give the users an option to add their own plugin-repositories from which they can discover, install, and uninstall any custom plugins. If the custom repository is defined within the constrained mentioned [here](https://github.com/konveyor/crane-plugins/blob/main/README.md) and looks like the default [crane-plugins](https://raw.githubusercontent.com/konveyor/crane-plugins/main/index.yml) repository, then the goal is that a user should be able add such repository and crane should be able to look at this added repository same as it looks at default crane-plugins.

## Motivation

The ability to add/remove custom crane-plugin repositories is needed in a restrictive environment where the access to the repository from outside the network is revoked, as well as for environments where custom plugins not provided in the crane-plugins repo are needed. In such cases, if a user can add their own custom-plugin repository then it would be beneficial for them to leverage `crane plugin-manager` command to manage plugins.

### Goals

- User should be able to add any plugin repository
- User should be able to remove any plugin repository
- User should be able to discover plugins from plugin repository
- User should be able to install plugins from plugin repository
- User should be able to uninstall plugins installed from added plugin repository
- User should be able to list all the plugin repositories available

### Non-Goals

Not applicable

## Proposal

We provide new subcommands, `add-repo|remove-repo|list-repos`, to `crane plugin-manager` for managing custom plugin repositories.

### User Stories

#### Story 1

As a user, I would like to manage plugin repositories (add|list|remove)

#### Story 2

As a user, I would like to manage my plugins using configured plugin repositories (install|uninstall|list)

### Risks and Mitigations

#### Risks
- If the user doesn't setup the custom plugin repository in the desired manner, the user might face some issues working with `crane plugin-manager`
- If the user overrides the setting for default plugin repo and replace it with nil, `crane plugin-manager` might become useless

#### Mitigations
- Testing the feature thoroughly to make sure that we don't break `crane plugin-manager list/add/remove` 
- Provide user with sufficient documentation on how to setup custom plugin repositories, how to develop custom plugins and how to properly add/remove custom crane-plugin repository
- A validation against a plugin repo that ensures a plugin-repository (that `crane` refers to) is valid. Meaning that added custom plugin repository have valid plugin manifests.


## Design Details

### Prerequisites

- User have their own plugins or modified version of default provided plugins developed in a similar manner as [this](https://github.com/konveyor/crane-plugin-openshift) plugin.
- User have a custom plugin repository setup in similar manner as [crane-plugin](https://raw.githubusercontent.com/konveyor/crane-plugins/main/index.yml)

#### Solution

##### Part 1

A config file `repos.yaml`, placed in `--plugin-dir`. This file contains plugin repository name as key and respective URL as value. Absence of this file in a plugin directory would mean crane would create a new repos.yaml in the plugin directory linking only to the crane-plugins repo. However, if an empty `repos.yaml` file is present means user need to add at least one plugin repository such as [crane-plugins](https://github.com/konveyor/crane-plugins). It makes sense to maintain `repos.yaml` for different `--plugin-dir` since, different `--plugin-dir` would have different set of plugins potentially installed from different sources.

- The file `repos.yaml` would look like following:
  
```
foo: <accessible location of an index.yaml file>
bar: <accessible location of an index.yaml file>
```

##### Part 2

We are proposing three new sub commands under `crane plugin-manager` to manage custom repositories (i.e `repos.yaml` file).

- To add custom crane-plugin repository

```
crane plugin-manager add-repo -h
to add custom crane-plugin repository

Usage:
  crane plugin-manager add-repo <name> <URL> [flags]

Flags:
  -h, --help   help for add-repo

Global Flags:
  -p, --plugin-dir string    The path where binary plugins are located (default "plugins")
```

On high level this command will look at `repos.yaml` located in local plugin directory, and add an entry with respective name and URL passed in the command. If passed name or URL is already added on the file then notifies user of the same.

- To remove custom crane-plugin repository
```
crane plugin-manager remove-repo -h
to remove custom crane-plugin repository

Usage:
  crane plugin-manager remove-repo <name> [flags]

Flags:
  -h, --help   help for remove-repo

Global Flags:
  -p, --plugin-dir string    The path where binary plugins are located (default "plugins")
```
The command will remove the entry from `repos.yaml` located in local plugin directory, if the respective name is not present then notifies the user of the same. Removing a plugin repository from `repos.yaml` also results in removing all the plugins installed from respective plugin repository, 

- To list all crane-plugin repositories

```
crane plugin-manager list-repos -h
to list all crane-plugin repository

Usage:
  crane plugin-manager list-repos [flags]

Flags:
  -h, --help   help for list-repos

Global Flags:
  -p, --plugin-dir string    The path where binary plugins are located (default "plugins")
```

- Similarly `crane plugin-manager list, add` will also look at the same config file to discover plugins from, to install plugins from, when used with `--repo` flag and otherwise when there is no conflicts of plugin name. 
  
**Note:** 
- All above command will create `repos.yaml` if it does not exists with a single entry that is - `crane-plugins: https://raw.githubusercontent.com/konveyor/crane-plugins/crane-cli/index.yml` a path to our crane-plugins.
- To update an entry in `repos.yaml`, first the repo needs to be removed(i.e `crane plugin-manager remove-repo foo -p bar`), and then needs to be added with updated path(i.e `crane plugin-manager add-repo foo <path> -p bar`)


## Drawbacks

By giving the flexibility of having a custom plugin repository, user have much more power to crane and have much higher chances of breaking `crane plugin-manager` because of misconfigurations on custom plugin repositories.

## Alternatives

Instead of maintaining a file in path `--plugin-dir` we ask users to create a file with the name `repos.yaml` and place it in `--plugin-dir` with appropriate entries, and we list, add and remove plugins by reading all the repos mentioned in the file `repos.yaml`. In this approach there is no need to add any commands, however the responsibility to create an appropriate config file for `crane` to consume lies on the user.