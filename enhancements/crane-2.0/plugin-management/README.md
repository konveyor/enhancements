---
title: Crane 2.0 State Transfer
authors:
  - "@jgabani"
reviewers:
  - "@shawn-hurley"
  - "@sseago"
approvers:
  - "@shawn-hurley"
creation-date: 2021-09-28
last-updated: 2021-09-28
status: implementable
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane 2.0 Plugin management

This document proposes a solution to manage additional plugins to the pipeline. The document describes the needed changes in current implementation, needed directory structure to discover plugins, and all the sub-commands that are needed for plugin management.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined

## Summary 

At a high level plugin management functionality will be

- Able to list the available plugins
- Able to install/uninstall plugins
- Able to add/remove custom repositories to list the plugins from (long term goal)

## Motivation

Apart from the offerings of crane-lib, we would like to add a way to allow users to extend the functionality of the tool by installing the managed or custom plugins.

## Non-goals

The initial solution will ship with listing, installing, and uninstalling plugins from the managed repository only and will not support adding custom repository and list plugins from there. The document might include some details on how custom repository might be added, but that is a high level approach and is subjected to change.

## Proposal

There are two parts of the solutions, the first part includes solving how to list plugins from the repository. The second part includes installing/uninstalling the plugin inside `plugin/managed-plugins`.

### First Part

One of the proposed solution is to have a `plugins` directory at the root of such repositories and respective `<plugin-name.yml>` inside the said directory from where we are listing plugins. The format of the yml file should be something like this

```
version: <plugin-version>
shortDescription: <short description>
description: <description>
platforms:
- selector:
    matchLabels:
        os: <os name>
        arch: <arch>
  uri: <uri of binary>
  sha: <sha of binary>
```

Any other suggestions are welcome to improve plugin list functionality. 

### Second Part

An additional command `plugin-manager` to manage all the plugins.

```
crane plugin-manager --help

Usage:
  crane plugin-manager [command]

Available Commands:
  list              lists all the available plugins
  add-plugin        installs the desired plugin
  remove-plugin     removes the desired plugin

Flags:
  -h, --help    help for plugin-manager 

Global Flags:
  --debug   Debug the command by printing more information

```


#### To support additional repository to install plugins

<b>Solution 1</b>:

Other than default repository (managed by our team), also read from env variable `CUSTOM_PLUGINS` to read from custom plugin github repository and list plugins.

`CUSTOM_PLUGINS` variable holds the links to the github repositories separated by `;`. For example `CUSTOM_PLUGINS=github.com/konveyor/plugins;github.com/jgabani/plugins`. List command will iterate through all the custom repositories url and list all the plugins.

<b>Solution 2</b>:

Have a `source.yml` file in a specific location `/home/<name>/.crane/source.yml`, and this file includes user preferred (except name `default`) source name and github url as a map, for example

```
foo: <repo-url-1>
bar: <repo-url-2>
```

List command will iterate through all the custom repositories url from `source.yml` file and list all the plugins.

##### Possible complications

1. Conflicting plugin name within different plugin repositories that are independent. 

    Possible solutions - 

    1. Must have unique plugin name across all plugin repositories (on a big scale this is more difficult to manage). OR
    2. Provide a flag in `add-plugin` sub command to input a source from where user wants to install the plugin. For example `crane plugin-manager add <plugin-name> --source=<source-url>` or `crane plugin-manager add <plugin-name> --source=<name>` depending on which solution of the two is selected for implementation.


## Result

- `crane plugin-manager list` will list all the available plugins
- `crane plugin-manager add-plugin foo` will place the plugin `foo` inside `managed` directory inside the default plugin directory - i.e. `/plugins/managed/foo`
- `crane plugin-manager remove-plugin foo` will remove the plugin `foo` from `plugins/managed/` directory

## Changes in existing code

Transform command need to look at `/plugins/` directory recursively to include all the plugins and handle possible conflicts and priorities.