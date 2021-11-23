---
title: neat-enhancement-idea
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

# Add/Remove custom plugin repository

This document proposes a solution to manage custom plugin repositories. The document describes the needed changes in current implementation, and all the sub-commands that are needed for this feature.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

- Do we want the user to have the control to override/remove default plugin source?

## Summary

Along with the our own default plugin-repository, we would like to give the users an option to add their own plugin-repositories from which they can discover, install, and uninstall any custom plugins. If the custom repository is defined within the constrained mentioned [here](https://github.com/konveyor/crane-plugins/blob/main/README.md) and looks like the default [crane-plugins](https://github.com/konveyor/crane-plugins) repository, then the goal is that a user should be able add such repository and crane should be able to look at this added repository same as it looks at default crane-plugins.

## Motivation

The ability to add/remove custom crane-plugin repositories is needed in a restrictive environment where the access to the repository from outside the network is revoked. In such cases, if a user can add their own custom-plugin repository then it would be beneficial for them to leverage `crane plugin-manager` command to manage plugins.

### Goals

- User should be able to add any custom plugin repository
- User should be able to remove any custom plugin repository
- User should be able to discover plugins from custom plugin repository
- User should be able to install plugins from custom plugin repository
- User should be able to uninstall plugins installed from custom plugin repository
- User should be able to list all the plugin repositories available

### Non-Goals

Not applicable

## Proposal

We will provide cli commands within `crane plugin-manager` to add and remove any custom plugin repository.

### User Stories [optional]

#### Story 1

As a user I would like to add my own custom plugin repository

#### Story 2

As a user I would like to list plugins from my custom plugin repository

#### Story 3

As a user I would like to install plugins from my custom plugin repository

#### Story 4 

As a user I would like to uninstall plugins from installed my custom plugin repository

#### Stroy 5

As a user I would like to remove any added custom plugin repository

#### Story 6

As a user I would like to list all the repos available to install the plugin from

### Risks and Mitigations

#### Risks
- If the user doesn't setup the custom plugin repository in the desired manner, the user might face some issues working with `crane plugin-manager`
- If the user overrides the setting for default plugin repo and replace it with nil, `crane plugin-manager` might become useless

#### Mitigations
- Testing the feature thoroughly to make sure that we don't break `crane plugin-manager list/add/remove` 
- Provide user with sufficient documentation on how to setup custom plugin repositories, how to develop custom plugins and how to properly add/remove custom crane-plugin repository


## Design Details

### Prerequisites

- User have their own plugins or modified version of default provided plugins developed in a similar manner as [this](https://github.com/konveyor/crane-plugin-openshift) plugin.
- User have a custom plugin repository setup in similar manner as [crane-plugin](https://github.com/konveyor/crane-plugins)

#### Solution

We are proposing two new sub commands under `crane plugin-manager` to manage custom repositories.

- To add custom crane-plugin repository

```
crane plugin-manager add-repo -h
to add custom crane-plugin repository

Usage:
  crane plugin-manager add-repo <name> <URL> [flags]

Flags:
  -h, --help   help for add-repo

```

On high level this command will look at a file placed at `home/user/.conf/crane.yaml`, and add an entry with respective name and URL passed in the command. If passed name or URL is already added on the file then notifies user of the same.

- To remove custom crane-plugin repository
```
crane plugin-manager remove-repo -h
to remove custom crane-plugin repository

Usage:
  crane plugin-manager remove-repo <name> [flags]

Flags:
  -h, --help   help for remove-repo
```
The command will remove the entry from `home/<user>/.conf/crane.yaml`, if the respective name is not present then notifies the user of the same.

- To list all crane-plugin repositories

```
crane plugin-manager list-repo -h
to list all crane-plugin repository

Usage:
  crane plugin-manager list-repo [flags]

Flags:
  -h, --help   help for list-repo

```

- Similarly `crane plugin-manager list, add` will also look at the same config file to discover plugins from, to install plugins from, when used with `--repo` flag and otherwise when there is no conflicts of plugin name. 
  
**Note:** All above command will create `crane.yaml` if it does not exists.

- The file `home/<user>/.conf/crane.yaml` would look like following:
  
```
foo: <github raw url for an index.yaml file>
bar: <github raw url for an index.yaml file>
```

## Drawbacks

By giving the flexibility of having a custom plugin repository, user have much more power to crane and have much higher chances of breaking `crane plugin-manager` because of misconfigurations on custom plugin repositories.

## Alternatives

Instead of maintaining a file we ask users to create such file with the name `crane.yaml` and place it under a specific path with appropriate entries, and we list, add and remove plugins by reading all the repos mentioned in the file `crane.yaml`. In this approach there is no need to add any commands, however the responsibility to create an appropriate config file for `crane` to consume lies on the user.