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
- Able to upgrade any/all installed plugins (long term goal)
- Able to add/remove custom repositories to list the plugins from (long term goal)

## Motivation

Apart from the offerings of crane-lib, we would like to add a way to allow users to extend the functionality of the tool by installing the managed or custom plugins. And we want users to use developed plugins as seamlessly as possible. We want them to consume out of binary plugins and make it possible to easily do so.

### Goals (for the initial release)

1. Add a new command `plugin-manager` that is able to handle plugin management functionalities.
2. This command must provide sub-commands to fullfil the requirement such as listing, installing, and uninstalling plugins

### Non-goals (this includes long term goals)

The initial solution will ship with listing, installing, and uninstalling plugins from the managed repository only and will not support adding/removing custom repository and list plugins from there. The document might include some details on how custom repository might be added, but that is a high level approach and is subjected to change.

## Proposal

We will provide cli tool within `crane` cli to make it possible for users to list, install and remove plugins.

### User stories

#### Story 1

As a user, I would like to list all the available plugins to me, `crane plugin-manager list`

The output should be:

```
<name>  <version>  <short description> <source>
```

#### Story 2

As a user, I would like to list all the installed plugins, `crane plugin-manager list --installed`

The output should be:

```
<name>  <version>  <short description> <source>
```
with the plugins installed via cli

#### Story 3

As a user, I would like to install any plugin that I might want on top of crane-lib, `crane plugin-manager add foo`

Should place the desired plugin in appropriate directory from which `crane transform` will include the plugin in the process.

#### Story 4

As a user, I would like to uninstall any plugin that I might want to remove, `crane plugin-manager remove foo`

Should remove the plugin from appropriate directory to not have this plugin execute during `crane transform`

## Design details

There are two parts of the solutions, the first part includes solving how to list plugins from the repository. The second part includes installing/uninstalling the plugin inside `<plugin-dir>/managed-plugins/<repo-name>`.

<i>Note: managed-plugins means the plugins that are downloaded by the cli commands and not put inside the `plugins` directory manually</i>

### First Part

The first part includes what should a repository from where we will list the plugins should look like.

One of the proposed solution is to have a `plugins` github directory at the root of such repositories and respective `<plugin_name-version.yml>` inside the said directory from where we are listing plugins. 

Example plugin repository layout:

```
.
└── plugins/
    ├── crane_x-1.0.yaml
    ├── crane_y-2.4.yaml
    └── crane_z-1.2.yaml

```

The format of the yml file should be something like this


```
name: <plugin-name>
shortDescription: <short description>
description: <description>
version: <version>
metadata:
- optionalFields:
    flagName: <flag-name>
    help: <help-message>
    example: <example>
binaries:
- os: <os name>
  arch: <arch>
  uri: <uri of binary>
  sha: <sha of binary>
- os: <os name>
  arch: <arch>
  uri: <uri of binary>
  sha: <sha of binary>
```

Any other suggestions are welcome to improve plugin list functionality. 

- Example manifest file - `foo-1.0.yaml`

```
name: foo
shortDescription: Example foo
description: This is a plugin example foo, there is no such plugin and this file doesn't do much.
version: 1.0
metadata:
- optionalFields:
    flagName: test-foo
    help: test
    example: test
binaries:
- os: linux
  arch: amd64
  uri: <valid-url>
  sha: <valid-sha>
```

### Second Part

An additional command `plugin-manager` to manage all the plugins.

```
crane plugin-manager --help

Usage:
  crane plugin-manager [command]

Available Commands:
  list       lists all the available plugins
  add        installs the desired plugin
  remove     removes the desired plugin

Flags:
  -h, --help                 Help for plugin-manager 

Global Flags:
  --debug   Debug the command by printing more information
```

```
crane plugin-manager list --help

Usage:
  crane plugin-manager list [flags]

Flags:
  -h, --help                 Help for plugin-manager list
  --repo string              List plugins from specific repo (optional)
  --installed bool           If this flag is passed, list all the installed plugins
  -p, --plugin-dir string    The path where binary plugins are located (default "plugins")
	--params bool								 If passed, returns with metadata information for all the version of specific plugin. This flag is to be used with "--name" flag
	--name string              To be used with "--info" and "--versions" flag to specify the plugin for which additional information is needed. In case of conflict, command fails and asks for a specific repository information.
	--versions bool						 If passed, returns with all the versions available for a plugin. This flag is to be used with "--name" flag.

Global Flags:
  --debug   Debug the command by printing more information

Example:
	crane plugin-manager list --info --name=foo --repo=bar, will retrieve all the information for all the versions of plugin "foo" in repository "bar"

```


```
crane plugin-manager add --help

Usage:
  crane plugin-manager add <name> [flags]

Flags:
  -h, --help                 help for plugin-manager add
  -p, --plugin-dir string    The path where binary plugins are located (default "plugins")
  --repo string              Install plugin from specific repo (optional), if not passed iterate through all the repo and install the desired plugin. In case of conflicting name the command fails and asks user to specify the repo from which to install the plugin
  --version string           Install specific plugin version (if not passed, installs latest plugin version)

Global Flags:
  --debug   Debug the command by printing more information

```
<i><b> Note:</b> to maintain uniqueness among the installed plugin name, binary name would be replaced with `<plugin-name>` from the manifest file and added to the `--plugin-dir`.</i>

```
crane plugin-manager remove --help

Usage:
  crane plugin-manager remove <name>

Flags:
  -h, --help                 Help for plugin-manager remove
  -p, --plugin-dir string    The path where binary plugins are located (default "plugins")
  --repo string              Remove plugin from specific repo (optional), if not passed iterate through all the repo and remove the desired plugin. In case of conflicting name the command fails and asks user to specify the repo from which to remove the plugin
  

Global Flags:
  --debug   Debug the command by printing more information

```



## Future scope

### To support additional repository to install plugins

<b>Solution 1</b>:

Other than default repository (managed by our team), also read from env variable `CUSTOM_PLUGINS` to read from custom plugin github repository and list plugins.

`CUSTOM_PLUGINS` variable holds the links to the github repositories separated by `;`. For example `CUSTOM_PLUGINS=github.com/konveyor/plugins;github.com/jgabani/plugins`. List command will iterate through all the custom repositories url and list all the plugins.

<b>Solution 2</b>:

Add subcommands within `plugin-manager` to support adding/removing any plugin repos. That could look something like this


```
crane plugin-manager repo --help

Usage:
  crane plugin-manager repo [command]

Available Commands:
  add           Add the desired plugin repo to install the plugins from 
  remove        Remove the added plugin repo
  
Flags:
  -h, --help    help for plugin-manager repo

Global Flags:
  --debug   Debug the command by printing more information

```
```
crane plugin-manager repo add --help

Usage:
  crane plugin-manager repo add <name> <url>

Flags:
  -h, --help    help for plugin-manager repo add

Global Flags:
  --debug   Debug the command by printing more information
```
```
crane plugin-manager repo remove --help

Usage:
  crane plugin-manager repo remove <name> 

Flags:
  -h, --help    help for plugin-manager repo remove

Global Flags:
  --debug   Debug the command by printing more information
```


We will maintain a `conf.yml` file in a specific location, either at `$XDG_CONFIG_HOME/crane` if exists or at ` $HOME/.config/crane/`. Based on the command executed we will add or remove entries for repo from this `conf.yml` file. This `conf.yml` should look something like following:

```
foo: <repo-url-1>
bar: <repo-url-2>
```

<i>

Solution 2+ (extension to solution 2): We can make use of both the above approach and have a way to over ride conf.yml file when the respective env var is set. This is not needed, but a nice to have in case where user doesn't want to remove any plugin repos but just want to check plugins of some other repo, for that one can use the said env var in solution 1.

</i>

## Result

- `crane plugin-manager list` will list all the available plugins for the current os arch.
- `crane plugin-manager add foo` will place the plugin `foo` inside `managed/<repo>` directory inside the default plugin directory. For example if plugin `foo` belongs to repository `bar` then the local plugin path would be - i.e. `/plugins/managed/bar/foo`, if the same plugin name is present across multiple repositories then the command will fail with the message including list of the repos where the plugin is available, so that user can run the command again with `--repo` flag.
- `crane plugin-manager remove foo` will remove the plugin `foo` from `plugins/managed/bar` directory. If there is another plugin with same name belonging to different repositories installed then the command will fail with message including the list of repositories from where the plugin was installed, so that user can run the command again with `--repo` flag.

<i> Note: any plugins that are manually added should be placed outside of managed directory inside plugins </i>

## Changes in existing code

Transform command need to look at `plugins/` directory recursively to include all the plugins and handle possible conflicts and priorities.

Collective for all command for consistency we may want to move to `$HOME/.crane/plugins` from `plugins/`.


<i><h2> Note: </h2>  Functionality related to multiple plugin repository will not be available in the first release. So commands with `--repo` flag might not work as mentioned in the initial release </i>