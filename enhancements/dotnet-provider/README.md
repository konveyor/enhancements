---
title: dotnet-provider
authors:
  - "@djzager"
reviewers:
  - "@pranavgaikwad"
  - "@shawn-hurley"
approvers:
  - "@pranavgaikwad"
  - "@shawn-hurley"
creation-date: 2024-02-15
last-updated: 2024-02-15
status: implementable
see-also:
  - https://github.com/konveyor/enhancements/pull/128
  - ../multi-language support/README.md
---

# External Provider for Analyzer LSP for .NET/C# Applications

## Summary

At the time of this enhancement,
[analyzer-lsp](https://github.com/konveyor/analyzer-lsp) has a builtin provider
with the filecontent, xml, and json
[capabilities](https://github.com/konveyor/analyzer-lsp/blob/ff08ad040dff50aa29a107ec4ba3ab56c3f5b65f/provider/internal/builtin/provider.go#L15-L55)
(not exhaustive), an internal Java provider for analyzing Java applications, as
well as several in-tree [external
providers](https://github.com/konveyor/analyzer-lsp/tree/6c8c947510aeb6dfe262e518f387fdb0babc41ef/external-providers):
generic, golang, and yaml.

This enhancement proposes a new in-tree external provider to
[analyzer-lsp](https://github.com/konveyor/analyzer-lsp) for the purpose of
analyzing .NET/C# projects and applications. 

## Motivation

The [.NET framework](https://en.wikipedia.org/wiki/.NET), including [.NET
foundation](https://en.wikipedia.org/wiki/.NET_Framework), (.NET) is an
extremely popular enterprise application platform dating back to the early
2000s. Extending our analyzer's capabilities to analysis of .NET/C# projects
has the potential to provide a great benefit to a multitude of potential users
who need help modernizing and/or migrating their .NET applications.

### Goals

- Propose a solution for analyzing .NET/C# projects (as old as [.NET Framework
  4.5](https://dotnet.microsoft.com/en-us/download/dotnet-framework/net45))
  from source. Specifically, language references (ie. `dotnet.referenced`)

### Non-Goals

- Make the technical decision between
  [omnisharp-roslyn](https://github.com/OmniSharp/omnisharp-roslyn) or
  [csharp-language-server](https://github.com/razzmatazz/csharp-language-server). 
- Describe any rules or rulesets for analyzing .NET/C# projects.
- Describe how binary mode analysis would/could be done.
- Describe .NET/C# dependency analysis via this provider. 

## Proposal

Introduce a new provider, dotnet-external-provider, that is capable of
analyzing the source code of a [.NET Framework 4.5](https://dotnet.microsoft.com/en-us/download/dotnet-framework/net45)
project or newer.

### Use Cases

#### Case 1 - In container

As a migration specialist, I want to analyze my cross-platform .NET project for
deviations from best practices. This project can be built and analyzed in
containerized environments.

In this scenario, the provider can be run inside a container with the language
server provider with the code to be analyzed accessible via the filesystem.

#### Case 2 - Self hosted

As a migration specialist, I would like to be able to find code references in
the source of my project that must be updated to migrate from .NET Framework
4.5 (Windows only) to .NET (cross-platform). This project can not be built,
read analyzed, in containerized environments.

For example, if I have a project like
[NerdDinner](https://github.com/sixeyed/nerd-dinner), I want to find all
occurrences of `HttpNotFound` that was
[replaced](https://aspnetcore.readthedocs.io/en/stable/migration/rc1-to-rtm.html#controller-and-action-results-renamed)
with `NotFound` in more recent versions of the .NET platform. I would write a
rule that looks like:

```
- message: HttpNotFound was replaced with NotFound in .NET Core
  ruleID: lang-ref-example-001
  when:
    or:
    - dotnet.referenced:
        pattern: "HttpNotFound"
        namespace: "System.Web.Mvc"
```

In this scenario, the provider **MUST** be run on a Windows system where the
language server provider can run (and the project built).

## Design Details

### Provider

Acknowledging the fact that we lack contributors with experience
writing/developing/maintaining/migrating .NET applications, the best thing we
can do for our users is enable the capability. For this reason, we will start
by supporting the `referenced` [provider
capability](https://github.com/konveyor/analyzer-lsp/blob/ff08ad040dff50aa29a107ec4ba3ab56c3f5b65f/provider/provider.go#L66-L69),
like:

```
func (p *dotnetProvider) Capabilities() []provider.Capability {
	return []provider.Capability{
		{
			Name:            "referenced",
			TemplateContext: openapi3.SchemaRef{},
		},
	}
}
```

### Service Client

One novel concern that arose when interacting with the two Language Server
Providers currently available is the need to handle their requests before
considering the project "Initialized". The csharp-language-server for example
requires that we handle their `client/registerCapability`,
`workspace/configuration`, and `window/showMessage` requests (this may not be
true anymore; see [Open Questions](#open-questions)). Specifically waiting for
a message containing "finished loading solution" or with the suffix "project
files loaded" before making requests to the language server.

```go
type handler struct {
	log *logr.Logger
	ch  chan int
}

func (h *handler) replyHandler(ctx context.Context, reply jsonrpc2.Replier, req jsonrpc2.Request) error {
	method := req.Method()
	switch method {
	case protocol.MethodClientRegisterCapability:
		err := reply(ctx, nil, nil)
		return err
	case protocol.MethodWorkspaceConfiguration:
		err := reply(ctx, nil, nil)
		return err
	case protocol.MethodWindowShowMessage:
		var showMessageParams protocol.ShowMessageParams
		if err := json.Unmarshal(req.Params(), &showMessageParams); err != nil {
			return reply(ctx, nil, err)
		}
		err := reply(ctx, nil, nil)
		if strings.HasSuffix(showMessageParams.Message, "project files loaded") || strings.Contains(showMessageParams.Message, "finished loading solution") {
			h.ch <- 3
		}
		return err
	}
	return jsonrpc2.MethodNotFoundHandler(ctx, reply, req)
}
```

This most likely has to do with the primary use case of the language servers
being development environments and not strict adherence to the [protocol
specification](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/).
While contributions are possible, it is best for us now to work around these
deviations and consider making a contribution to one or both projects in the
future.

### Containerization

The distinguishing characteristic between the two use cases stated above is the
environment in which the provider is run (in or out of the container). In the
same way that [Kantra](https://github.com/konveyor/kantra) provides binaries
for other operating systems [inside their built
container](https://github.com/konveyor/kantra/blob/main/Dockerfile#L29-L31), we
will include binaries for Windows hosts of our dotnet provider inside the
provider container. This will enable end users to extract that binary onto a
Windows host where their .NET Framework project can be built, run that provider
listening on a dedicated port, and finally update the `provider_settings.json`
with the coordinates of the provider before running the analyzer.

The relevant portion of the Dockerfile would look something like:
```Dockerfile
RUN CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -a -o windows-dotnet-external-provider main.go
```

The user would extract the binary with a command like:
```shell
podman pull quay.io/konveyor/dotnet-external-provider:latest && \
podman run --name dnp quay.io/konveyor/dotnet-external-provider:latest 1> /dev/null 2> /dev/null && \
podman cp dnp:/usr/local/bin/windows-dotnet-external-provider dotnet-external-provider && \
podman rm dnp
```

Provider settings could look like:
```json
[
	{
		"name": "dotnet",
		"address": "1.2.3.4:12345",
		"initConfig": [{
			"location": "C:/path/to/project",
			"providerSpecificConfig": {
				"lspServerPath": "C:/path/to/lsp.exe"
			}
		}]
	}
]
```

**NOTE**: While we aren't fully articulating how this would be integrated into Konveyor's user experience, this should help with design considerations for future enhancements around Konveyor's multi-language support.

## Open Questions
* The Java provider had a `Location` field on it's `referenced` condition (see
  [here](https://github.com/konveyor/analyzer-lsp/blob/main/provider/internal/java/provider.go#L125-L181)),
  should we support this in the .NET provider's referenced condition?
* Looks like the concerns about respecting provider capabilities may have been
  addressed in the time since [this comment](https://github.com/razzmatazz/csharp-language-server/issues/100#issuecomment-1653534049).
