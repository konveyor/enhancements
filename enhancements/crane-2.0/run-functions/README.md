---
title: run-krm-functions-from-crane
authors:
  - "@MundraAnkur"
reviewers:
  - "@jwmatthews" 
  - "@dzager"
  - "@shawn-hurley"
  - "@shubham-pampattiwar"
approvers:
  - "@jwmatthews" 
  - "@dzager"
  - "@shawn-hurley"
  - "@shubham-pampattiwar"
creation-date: 2022-06-21
last-updated: 2022-06-21
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Run KRM functions from crane

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions
- Do we really need the capability to execute KRM functions integral to the Crane?

## Summary
This enhancement discusses a possible solution to run KRM functions from the crane. It proposes to add the crane CLI subcommand that can execute a KRM function.

On Crane, adding the subcommand to execute functions will ensure that -
- It can execute a containerized function against the given input resources and store the transformed output to a destination directory

## Motivation
At present, we can not utilize the KRM functions in the crane. To leverage the benefits of KRM functions and aligning with the community standards, we are proposing to add the capability in the crane CLI to execute a function. 

### Goals

- Propose solution to handle execution of KRM functions in crane.
- Implement the proposed solution.
 

## Proposal
- We will add a subcommand in the crane cli that can run KRM functions with the arguments. The underlying functionality can be built on top of [kyaml](https://pkg.go.dev/sigs.k8s.io/kustomize/kyaml) package that provides libraries to run containerized function images. We'll also create a library which will parse the function configuration from a functionconfig file and invoke the functions with their respective arguments defined in the config file. It will help us to enhance the existing crane commands and to automate the execution of function pipeline.

### Command
Crane subcommand will follow this synopsis:
```
crane fn run [IMAGE] [flags] [-- args]
```
> The above subcommand will be used for transforming resources in a directory using functions. If a function fails, the process is aborted and the resource files will be left unchanged.
 
#### Args:
  *IMAGE:*
>	Container image of the function to execute e.g. quay.io/krm-fn/set-annotation:v3.1.
	
  *args:*
>	Arguments to pass to the function. The value can be in `key=value` format and come after the separator **'--'**

  *Flags:*
>

	  --export-dir, e:
	  	Path to the local directory containing resources. 
		Defaults to the export directory.
  	
	  --transform-dir, t:
		If specified, the output resources are written to provided location,
		if not specified, resources are written to transform directory.
		
	  --as-current-user:
		Use the uid and gid of the command executor to run the function in the container.
		If the flag is not specified the container will run as `nobody`
  
	  --env:
		List of local environment variables to be exported to the container function.
		The value can be in key=value format or only the key of an already exported environment variable..
  
	  --mount:
		List of storage options to enable reading from the local filesystem.
	  
	  --network:
		If enabled, container functions are allowed to access network.
		By default it is disabled.

#### Examples:

```
# apply function example-fn on the resources in export directory and write output back to DIR
$ crane fn run quay.io/example.com/example-fn --export-dir export
```

```
# apply function example-fn with an input ConfigMap containing `data: {foo: bar}`
$ crane fn run gcr.io/example.com/example-fn -- foo=bar
```

```
# apply function example-fn on the resources in exportGuest directory with network access
$ crane fn run -e exportGuest gcr.io/example.com/example-fn:v2.6 --network
```

```
# apply function example-fn on the resource and export foo environment variable
$ crane fn run gcr.io/example.com/example-fn --env foo=bar
```

```
# apply function 'set-namespace' on the resources in current directory and write
  the output resources to dest directory
$ crane fn run docker.io/example.com/set-namespace:v0.1 --transform-dir path/to/dest -- namespace=crane
```

### User Stories

#### Story 1
As a crane user, I would like to run a single KRM function with crane subcommand using function image as an argument and other command-line flags, such that the function is applied against the input resources and produces the transformed output. 

#### Story 2
As a crane user, I would like to run KRM functions with enhanched ```crane transform``` subcommand by passing a function config file, **without** explicitly using the ```crane fn run``` cli subcommand, such that the given function configuration is applied against the input resources and produces the transformed output.

We will pass a configuration file containing all the functions we want to execute and any additional arguments needed for each function invocation.

Example Config:
```
apiVersion: v1
kind: FunctionConfig
metadata:
  name: fn-config1
spec:
  input-resource-dir: testdata/export/
  functions:
    - image: quay.io/fnproject/fn-test-utils:latest
      env:
        FN_TEST_ENV: fn-test-env
        FN_TEST_ENV_2: fn-test-env-2
      mount:
        - type: bind
          src: path/to/fn-test-utils
          dst: /fn-test-utils
          readOnly: true
    - image: set-namespace:v0.2
      network: true
      data:
          namespace: guestbook
  output-resource-dir: testdata/transform/
```

#### Story 3
As a crane user, I would like to run multiple functions in a predefined manner such that they are applied to input resources and produces a transformed output.


## Alternatives
Alternative to creating an integrated solution to crane cli, we can leverage the already built solutions that provide similar functionality like [KPT](https://github.com/GoogleContainerTools/kpt) and [Kustomize](https://github.com/kubernetes-sigs/kustomize).

At present, user can perform the following steps to utilize KRM functions in crane:
 1. User export the resources using `crane export`, then 
 2. User transform the resources with `kpt | kustomize`  and store them in the transform directory, finally
 3. User applies the transformed resources to the target using `crane apply`
  
This approach is rejected for following reasons:
- In the future, we would like to expand the functionality of the runner provided by ```kpt | kustomize``` and therefore, we need our wrapper of conatiner runner, which will be integral to the crane.
- We are also proposing a [KRM Function Manager](https://github.com/konveyor/enhancements/pull/73)(Kaffine) enhancement, which manages the KRM functions. An in-house wrapper of container runner gives us the flexibility to integrate features of ```kaffine``` to enhance the KRM function execution.
- As we are also proposing to enhance the ```crane transform``` command to execute functions under the hood, we need to have function execution capability with the crane.
