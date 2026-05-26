---
title: transform-plugin-priority
authors:
  - "@sseago"
reviewers:
  - "@shawn_hurley"
  - "@alaypatel07"
  - "@jmontleon"
approvers:
  - "@shawn_hurley"
  - "@alaypatel07"
  - "@jmontleon"
creation-date: 2021-06-24
status: implementable
see-also:
  - "/enhancements/this-other-neat-thing.md"  
replaces:
  - "/enhancements/that-less-than-great-idea.md"
superseded-by:
  - "/enhancements/our-past-effort.md"
---

# Transform plugin priority

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

## Summary

Currently, when generating a transformation, the plugins passed into
the transform runner are executed in slice order, and when there are
conflicting patches generated, the plugin which executed first in the
conflict set gets its patch applied. Subsequent plugins whose patches
conflict with the first get their patches discarded. The patch run
order isn't necessarily always predictable, and when it is
predictable, it the ordering has nothing to do with priority or
importance.

We need to introduce a notion of plugin priority which is associated with
a transform action, rather than an inherent attribute of each
plugin. The user should be able to provide a non-exhaustive list of plugin names in
priority order as a CLI option for the `transform` command. This could
be a single "must take priority" plugin, or a list of all plugins in
priority order. Any plugins not prioritized in this way will be
considered according to the default application order, based on the
slice passed into the runner.

## Motivation

Being able to choose which plugin actions take priority becomes
important when third party or custom plugins are used in conjunction with
included Crane plugins.

### Goals

- Ability to choose, for a single CLI transformation run, which
  plugins (if any) should take precedence
- Ability to run without prioritization, if desired

### Non-Goals

- Whiteout behavior will not be affected by prioritization. If any
plugin whites out a resource, it will be excluded.

## Proposal

### User Stories

#### Story 1

As a library user, I would like to be able to control the priority of
one or more plugin patches when conflicts arise.

#### Story 2

As a transform CLI user, I would like to be able to control the priority of
one or more plugin patches when conflicts arise.

### Implementation Details/Notes/Constraints

The `transform.Plugin` interface will need to add a `Name()` func to
return the name of the plugin, since plugin names will be used to
identify priorities.

## Design Details

### crane-lib changes

#### transform package

`transform.Plugin` will need a new `Name()` func:

```go
type Plugin interface {
	// Determine for a given resources what the plugin is deciding to do with this
	Run(*unstructured.Unstructured) (PluginResponse, error)
	// Returns the name of the plugin
	Name() (string)
}
```
Name uniqueness won't be enforced at this point. If two plugins return
the same name, then conflicts between them will be handled in the same
way as conflicts between plugins that are not in the prioritized list
-- the plugin patch earliest in the patch list will be applied.

For binary plugins, the initial implementation will use the filename
as the plugin name. A possible future enhancement would be for the
plugin itself to return its own name in response to a "name" or
"version" request.

`transform.Runner` will need a `PluginPriorities` map field:

```go
type Runner struct {
	// This is where we need to put extra info
	// This should include generic args to be passed to each Plugin
	// This also needs to handle the options that it will need.
	// TODO: Figure out options that the runner will need and implement here.
	PluginPriorities map[string]int
	Log *logrus.Logger
}
```

`PluginPriorities` will be a map of plugin name keys and priority
values. Lower priority numbers take precedence (0 takes precedence
over 1, which takes precedence over 10, etc.).

When the runner calls the plugins and collects the patch responses
from each, we will need to associate each patch with the name of the
plugin which added it. To do this, we will add a new struct,
`PluginOperation`:
```go
type PluginOperation struct {
	PluginName string
	Operation  jsonpatch.Operation
}
```
Instead of just building a `jsonpatch.Patch` (which is just
`[]jsonpatch.Operation`), we'll build a `[]PluginOperation`
When the runner calls SanitizePatches this `PluginOperation` list
will be passed in. The return patches and ignored patches will be
unchanged in type or format.

Within `sanitizePatches`, the existing (internal) `patchMap` will also
be modified to use `PluginOperation` instead of
`jsonpatch.Operation`. That way when iteratng over patches, we will
always have access to the plugin name for the patch we're looking at,
as well as the previous patch for the same operation (if there's a
conflict). Instead of just taking the first patch and adding the
current one to ignoredPatches, we'll use r.PluginPriorities to
determine whether one of the two patches comes from a higher-priority
plugin. If the current iteration's plugin is of a higher priority than
that of the previous operation for this `operationKey`, then we will
replace the matching entry in `patchMap` with the current operation,
and put the previous map entry on the ignored list. Otherwise, the
current operation in the map will be retained.

#### binary_plugin package

`BinaryPlugin` will get a new `name` field:
```go
type BinaryPlugin struct {
	commandRunner
	name string
	log logrus.FieldLogger
}
```
The `Name()` func will return the value of this field.

For the initial implementation, `NewBinaryPlugin()` will set `name` to
the filename component of the passed-in plugin path. In the future, we
may update the binary plugin interface to allow us to ask the plugin
what its name is.

### crane changes

#### transform command

`transform.Options` will get a `PluginPriorities` field:
```go
type Options struct {
	logger           logrus.FieldLogger
	ExportDir        string
	PluginDir        string
	TransformDir     string
	PluginPriorities string
}
```
The corresponding CLI option, "plugin-priorities", will be a comma-separated
list of plugin names, with earlier plugins named taking precedence
over later-named plugins, and any named plugin will take precedence
over unnamed ones, in the event of a conflict. When creating the
`transform.Runner`, the "plugin-priorities" list will be split, with
the index in the resulting slice being set as the value in the map --
0 for the first in the list, 1 for the second, etc.

A future enhancement to this might provide the ability to supply a
configuration file with plugin names and numbered priority values
instead of the indexed list.
