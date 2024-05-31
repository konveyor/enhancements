# Background

Condensing from the enhancement, it is very important for us to provide a
straightforward path for Konveyor users to analyze .NET Framework applications.
Some important facts in bulleted form:

* .NET Framework applications/projects are Windows only! This means that we
  must have a Windows environment to analyze. At a minimum this is a Windows
  container.
* Podman does not support running Windows containers. See
  [containers/podman#11809](https://github.com/containers/podman/issues/11809#issuecomment-1119386013).
* The END Goal is to support both in-cluster (read Windows Nodes) and CLI based
  workflows. We are not targetting in-cluster for v0.5 (the current
  enhancement) and CLI based workflows will be technical preview state.
* This says nothing of the work required to create a ruleset for analyzing .NET
  Framework applications targetting .NET 8.0.
* Kantra currently relies on `docker` or `podman` to create a network, add
  volumes, and provide those to the containers it starts.

From the original proposal:

> Introduce a new .NET Provider for Kantra. Using
  [alizer](https://github.com/devfile/alizer) to determine the language and
  frameworks used, we can differentiate between .NET projects that can be
  analyzed in a Linux container versus .NET Framework projects that must be
  analyzed in a Windows container. With this knowledge, we simply start the
  dotnet-external-provider as a Windows container when appropriate, perform the
  analysis, and retrieve the results.

We are specifcially concerned with the Windows-only .NET Project (.NET
Framework 4.5 through .NET Framework 4.8):

> This story is **fundamentally** different than how Kantra runs today, in
  that, it requires Kantra to start a Windows container that only `docker`
  supports. Additionally, communication between the Kantra Linux container and
  dotnet-external-provider Windows container must be established for the analysis
  to be run to completion.

# Problem

The original proposal didn't really provide specifics about two implementation details 
that are a cause for concern:

## Discerning Windows-only .NET Projects

Running `alizer` against two sample .NET projects it becomes clear that we
can't rely on the language `Name` alone:

**Windows-only .NET Project**
```
➜  alizer analyze NerdDinner
[
  { "Name": "JavaScript", "Aliases": [ "js", "node", "nodejs", "TypeScript" ], "Weight": 51 },
  { "Name": "C#", "Aliases": [ "csharp", "dotnet", ".NET" ], "Weight": 46, "Frameworks": [ "v4.5" ] }
]
```
Note: We need the Name and Frameworks.

**Cross-Platform .NET Project**
```
dotnet-external-provider/examples/HelloWorld on  image-build-bug [$?]
➜  ../nerd-dinner/alizer/alizer analyze HelloWorld
[ { "Name": "C#", "Aliases": [ "csharp", "dotnet", ".NET" ], "Frameworks": [ "net8.0" ] } ]
```

Looking at the analyze subcommand,
[where the input components are detected](https://github.com/konveyor/kantra/blob/aa605e96cf810979d69387b5a219143afe2f1236/cmd/analyze.go#L150-L164):

```golang
			components, err := recognizer.DetectComponents(analyzeCmd.input)
			if err != nil {
				log.Error(err, "Failed to determine languages for input")
				return err
			}
			foundProviders := []string{}
			if analyzeCmd.isFileInput {
				foundProviders = append(foundProviders, javaProvider)
			} else {
				for _, c := range components {
					log.Info("Got component", "component language", c.Languages, "path", c.Path)
					for _, l := range c.Languages {
						foundProviders = append(foundProviders, strings.ToLower(l.Name))
					}
				}
			}
```

We would need to pass around a more detailed data structure in order to
properly handle the supported use cases. Technically speaking this isn't too
difficult. Of primary concern here would be the code complexity after this
change was completed.

## Non-Homogenous Container Workloads

When we say non-homogenous here, we are referring to running Windows containers
and Linux containers from the same host. Technically speaking, the Linux
containers wouldn't be running on the same host but rather a `docker` or
`podman` virtual machine on the Windows host. Herein lies the concern, can we
reliably start a Windows container using `docker` and then a Linux container
using `docker|podman` that can reach the Windows container's network? Our
collective lack of experience dealing with Windows, Windows container
workloads, and Windows container <-> Linux container communication makes this a
huge unknown.

**tl;dr** We need to reduce the risk of technical debt added to `kantra` as
well as open questions in order to ensure timely delivery of the feature.

# Possible Solutions

## Power Through

This would be `kantra analyze` detects C# .NET Framework 4.5-4.8 project,
starts the dotnet-external-provider as a Windows container (via docker), and
tells the Linux container started by docker/podman how to reach the provider.

### Benefits

* Least burden to user. New flags/subcommands or worse having to `docker run`
  is not a great experience.
* No new Windows containers.

### Drawbacks

* Is there enough time?
* Added complexity/technical debt to kantra
* Are we certain we can reliably run Windows and Linux containers on the same
  host?

## Short Circuit Provider Logic

Initially there was conversation about a wholly new subcommand, then a new
`--provider` flag, now I'm pretty certain we could simply detect C# .NET
Framework 4.5-4.8 project and move start a whole new flow of logic for the
`analyze` subcommand in this flow. How would this work?

1. Use the `recognizer.DetectComponents()` to get all the components in a project.
1. If it contains a component that is C# **AND** it's framework is v4.5, then we
   break the normal flow of the `analyze` command to perform the remaining steps.
1. Use docker to create a network, volume, and start the
   dotnet-external-provider and kantra Windows containers. The difference here is
   that kantra will be provided as a Windows container to minimize the complexity
   of workflow (namely the complexity of this approach is in getting Kantra to run
   as a Windows container).
1. Retrieve and output the results as in `kantra` normally.

### Benefits

* Best UX for the end user, no new flags or subcommands.
* Not mixing Windows/Linux containers means less unknowns and potential for gotchas.

### Drawbacks

* Not certain how long it will take to make Kantra available as a Windows container.

## Provider Settings Override

In this last option, we leverage
[konveyor/kantra#221](https://github.com/konveyor/kantra/pull/221).
That is, we would require the end-user to start the dotnet-external-provider on
their own, create the appropriate provider-settings, and pass that to `kantra`.

This has a lot of the same problems as the original because we would be hoping
running Windows containers and Linux containers side-by-side would work smoothly.

### Benefits

* After discussion, not a whole lot.

### Drawbacks

* Issue of windows containers and linux containers on the same host.
* Terrible UX.
