---
title: Providers IPC over Local Sockets and Named Pipes
authors:
  - "@shawn-hurley"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-07-30
last-updated: 2025-07-30
status: implementable
see-also:
  - "https://github.com/konveyor/analyzer-lsp/pull/860"  
replaces:
  - "/enhancements/multi-language support/README.md"
---

# Using Sockets and Named Pipes for IPC of analyzer to providers

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

## Summary

Currently, users face a slow and cumbersome experience when running analysis with
providers, primarily due to the reliance on container-based communication Along
with this, there is not user experience for provider management, specifically
adding or removing providers.
This enhancement proposes a shift to a more efficient communication method
using local sockets and named pipes.
This change will dramatically improve analysis speed and simplify provider management
by introducing a new command-line interface for installing and managing providers.
The result for the end user will be a significantly faster, more streamlined, and
user-friendly analysis process by enabling users to take control of the providers.

Adding configuration as a first class citizen to kantra, will enable easier use with
the CLI, as you will need to add less options per command. Along with this will
be a more ergonomic CLI interface, that will both feel familar and not break backwards
compatability but will move to a style where commands manage one thing, and figuring
out what to do is easier from the command help.

## Motivation

As we were working on Kai and the `kai_analyzer_rpc` project, we needed to communicate
over sockets and named pipes in Windows. This is how VS Code will communicate with
its extensions. This led me to an experiment, which was that if we expected users
to use VS Code and extensions, then we can do the same process for IPC.

Providers and the Analyzer (either run through `kantra` or `kai_analyzer_rpc`)
currently can only communicate over in tree/binary process or HTTP. This has led
to some issues, specifically around using containers to run providers as the process
is slow. We have found that running locally is much faster, and that is why we built
[containerless java analysis](https://github.com/konveyor/kantra/pull/338). I believe
that moving analysis to be able to communicate over a socket/named pipe that we can
achieve the same performance benefits for all providers.

Furthermore, the absence of a streamlined user experience for managing providers
is a major usability gap. Users lack a simple way to add, remove or configure their
analysis tools, creating a significant barrier to adoption and customization.
This proposal is equally motivated by the need to empower users with direct control
over their environment through a clear and simple command-line interface for `kantra`.

### Goals

- We should enable the analysis engine to communicate over a
[Unix Domain Socket](https://pubs.opengroup.org/onlinepubs/9799919799/functions/socket.html)
or a [Named Pipe](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipes)
on Windows.

- Providers should be installed on the local machine and should ensure all of
the necessary dependencies are installed when being installed.

- Providers should be first-class entities from `kantra`, with their own subcommand.

- Providers should be usable in `kai_analyzer_rpc` and be first-class entities.

### Non-Goals

- We should not be responsible for the installation of the programming language
- We should not be responsible for the installation of the programming language's
package manager

- Changes to the way the Hub runs analysis.

## Proposal

### Enable Communication over Sockets and Named Pipes

The first step should be to enable the provider server and client to connect over
sockets and named pipes (will only use sockets from here on, but know that named
pipes are what will be used on Windows). The changes can be seen in the [pr](https://github.com/konveyor/analyzer-lsp/pull/860)
but will be summarized to the main points here.

#### New Package to Handle Sockets

We will create a new package that will have exposed methods for handling the
use of sockets. Note that we will have to make use of the operating system [build
constraints](https://pkg.go.dev/cmd/go#hdr-Build_constraints) to get a different
implementation on Windows than on unix like systems.

```go
// Used by the provider client, in the engine to connect to the provider.
func ConnectGRPC(connectionString string) (*grpc.ClientConn, error)

// Used by the provider start-up in the engine, to pass the socket name
// to the provider via a command line flag.
func GetSocketAddress(name string) (string, error)

// Used by the provider server, that will handle creating a server for the
// given socket.
func Listen(socket string) (net.Listener, error)
```

#### Update Provider Config and Provider Start Up

We will add a new config value, that will default to false for backwards compatability.

```go
type Config struct {
...
    UseSocket    bool         `yaml:"useSocket,omitempty" json:"useSocket,omitempty"`
...
```

We will update the `NewServer` to have a new parameter named socketPath.

*** TODO: this needs to be changed for backwards compatibility

```go
func NewServer(... socketPath string, logger logr.Logger) Server 
```

The server now decides if it is using a port, or if it is using a socket, and will
create the correct `net.Listener` for the `grpc` server to use.

In the grpc provider package, which contains the internal provider implementation
for the engine, we will use the config value to determine if network or socket
and choose the correct grpc client to use.

#### Update External Providers

External providers will need to add support for an optional parameter `socket`
to know which socket to serve on.

```go
// New flag that is optional
    socket        = flag.String("socket", "", "Socket to be used")
...

// Handle if there is no place to listen and error.
    if (socket == nil || *socket == "") && (port == nil || *port == 0) {
        log.Error(fmt.Errorf("no serving location"), "port or socket must be set.")
...
// Use the new method to create the server.
    s := provider.NewServer(client, *port, c, k, secret, *socket, log)

```

### Enable Kantra to Use Socket Communication

To enable Kantra to use sockets, we will add a new flag that will help preserve
old behavior of running containers. We will call this flag `--run-container`.
`runLocal` will continue to mean run the in-tree java provider.

If both of these are false we will use shared code in analyzer-lsp repo to load
the configurations and insert the correct values based on the flags.

To facilitate having three different modes of running analysis, we will need to
encapsulate the logic of each mode in a struct. Each analysis mode
will get it's own type and an interface will be what the command uses to run
analysis.

The command will still be responsible for exiting early based on the presence of
`listSources`, `listTargets`.

We will move the `listProviders` and `listLanguages` to a new command that will
be responsible for dealing with providers.

#### rewrite analysis command

The rewrite of analysis is going to remove many methods from the analyze.go
file and move them to their respective types in `pkg/analysis'. There will be
a factory method that will take the necessary flags from analysis command to create
the objects.

```go

func CreateAnalysisRunner(...flags, providers)

type AnalysisRunner interface {

    Run() ([]outputv1.RuleSet, error)
}

type inTreeJavaRunner struct {
    ...Fields
}

type containerRunner struct {
    ...Fields
}

type socketRunner struct {
    ...Fields
}
```

Once all of the functionality has moved around, I think that debugging a
particular run mode will be much easier.

#### new provider command

The new provider command will be responsible for installing into '$HOME/.kantra`
the provider settings template file, all the necessary utilities beside the programming
language and the dependency management tool. We will have to add docs to detail
this and have good error messages.

The providers list to be installed will be driven by the `$HOME/.kantra/config.yaml`.
Here we will set the for each provider that is installable, the location for downloading
the zip or the container. Kantra will be responsible for downloading and extracting
the information to the correct place.

A user who has their own provider will be able to update the config and point to
the download location, this could incldue an on-disk path.

Each provider, may additionally come with their own default ruleset. This ruleset
should be added to the kantra ruleset's directory, and should get added to it's own
directory in that.

#### provider install

`kantra provider list [-i,--installed]'

This command will allow you to list the providers that you have installed if
using the -i or --installed option, otherwise it will list all the well-known
providers telling you if they are installed or not. The output should be well
formated as a table.

```bash
$ kantra provider list
Provider Name    Language    Installed    Version
java-provider     java         yes         1.8.0
 go-provider     golang         no         1.8.0
....
```

`kantra provider install <name-of-provider>`

This command will let you install the provider. To facilitate installation we will
first try to use a container and copy out the necessary bits. If that
is not available, then we will curl a .zip file. You will be able to, in a config
file for kantra, set the locations from which to pull from.

During install each provider will create a `<language>_provider_settings.json.tmpl`
file. This file will be used to generate for a given run of analysis the provider_config
for that provider. The `<language>` will be the name of the provider and have a
directory that will be for it's tools and what it needs.

The .kantra directory

```bash
$ tree .kantra
.
|
|-- config.yaml // The kantra config
|-- java_provider_settings.json.tmpl
|-- java
    |-- jdtls
    |-- bundle.jar
etc...
```

`kantra provider remove <name-of-provider>`

This command should undo all that the install command does.

### Security, Risks, and Mitigations

I don't see any security risks with this approach. There will be no network
traffic, as we are moving from containers serving over a port to a socket.

## Design Details

### Test Plan

For testing, we will need an extensive amount of help getting all the infrastructure
in place to deploy the providers in this way. We will need to verify for all the
well-known providers that they can be installed on Windows, Mac, and Linux.

We will have to think carefully about when installing Kantra or Kai if we want to
by default to installing the Java provider to keep backwards compatibility.

We will have to make sure that we have CI jobs that cover all the ways to run
kantra and Kai, as well as the different operating systems that they run on. We
will need to have tests that install and remove providers and verify that the list
commands still function correctly.

### Upgrade / Downgrade Strategy

We will need to do a thorough job of testing to make sure that the commands that
 worked before continue to work, and if they don't, we need to expose why and
what to do to the user. This includes error messages, validations, and documentation.
We should have a thorough document that covers the breaking changes that this will
introduce.

## Implementation History

There have been many iterations of different ways and defaults for the way that
`kantra` runs. Because of this, we need to take special care in how we communicate
to users about how things have changed and try our best to have the commands that
worked before continue to work. This may mean that if you try to use a well-known
provider that is not installed, it will be automatically installed.

## Drawbacks

There will be a considerable amount of change to the `kantra` repo as well as
the way it functions. It will also cause the testing scenarios to increase and
will require a shift in the release and CI processes.

## Alternatives

### Alternative 1

There is a possibility of not doing some sections of this and still doing others.
This should be thought about, and the work described here should be broken up.

### Alternative 2

We could do nothing in Kantra and continue with the approach of using containers
for all non-Java providers. I think that this will have negative consequences as
we move the project forward.

## Infrastructure Needed [optional]

We need more CI jobs for Kantra and we will need more release artifacts for
the providers.
