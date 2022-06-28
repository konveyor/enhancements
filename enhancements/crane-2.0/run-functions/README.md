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
- We will add a subcommand in the crane cli that can run KRM functions with the arguments. The underlying functionality can be built on top of [kyaml](https://pkg.go.dev/sigs.k8s.io/kustomize/kyaml) package that provides libraries to run containerized function images.

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
		
	  --env:
		List of local environment variables to be exported to the container function.
		The value can be in key=value format or only the key of an already exported environment variable.
		
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


## Alternatives
Alternative to creating an integrated solution to crane cli, we can leverage the already built solutions that provide similar functionality like [KPT](https://github.com/GoogleContainerTools/kpt) and [Kustomize](https://github.com/kubernetes-sigs/kustomize).

At present, user can perform the following steps to utilize KRM functions in crane:
 1. User export the resources using `crane export`, then 
 2. User transform the resources with `kpt | kustomize`  and store them in the transform directory, finally
 3. User can apply the transformed resources to target cluster using ```kubectl apply```
  
This approach is not entirely rejected and could come into play in the future; for now, we would like to know how the proposed solution of function execution will work with crane, and it also gives us the flexibility to customize function execution as per the crane's requirement.
