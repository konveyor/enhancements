---
title: reporting-transfer-progress
authors:
  - "@pranavgaikwad"
reviewers:
  - "@djzager"
  - "@jmontleo"
  - "@JaydipGabani"
  - "@shawn-hurley"
approvers:
  - "@djzager"
  - "@jmontleo"
  - "@JaydipGabani"
  - "@shawn-hurley"
creation-date: 2022-07-11
last-updated: 2022-07-11
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Reporting transfer progress

The `transfer-pvc` subcommand works on one PVC at a time. One of the main advantages of processing one PVC at a time is that it improves observability and debugging. Currently the subcommand tails logs from Rsync client pods directly. Since Kubernetes api does not support _streaming_ output and error logs separately, the subcommand prints both logs to the console as-is. This enhancement proposes a solution to parse Rsync logs and print custom progress of transfer to _stdout_ instead. It also proposes to separate _stdout_ and _stderr_ based on parsed output. 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

This enhancement proposes a solution to parse rsync logs and provide users with a summary of ongoing transfer. It also proposes to separate error and output streams based on parsing. This will help users grok information easily, provide a way to separate errors from successes, and make debugging easier.

## Motivation

Unlike MTC (where logs were polled at certain intervals by the controller), Crane works on a log stream directly. The working is similar to `kubectl logs` command. This reduces (perhaps, eliminates) loss of log lines and delay. As a result, we can provide transfer progress with much more accuracy, speed and detail as compared to MTC. Additionally, the subcommand works on only one PVC at a time. It makes it easier to work on logs in isolation.

### Goals

> Provide better progress information about ongoing transfer
    > Details like transfer speed, number of files transferred, total data transerred, completion percentage, errors, list of failed files 

> Separate output and error streams

> Optionally, write structured progress output to a file for automations to consume

## Proposal

Currently, rsync logs are streamed and copied to `stdout` via a io.Reader:

```golang
	reader, err := podLogsRequest.Stream(context.Background())
	if err != nil {
		return err
	}
	_, err = io.Copy(os.Stdout, rsyncLogStreamReader)
	if err != nil {
		return err
	}
```
`Stream()` api returns a reader. `Copy()` reads data from reader and writes to stdout until EOF is reached. 

We will introduce a custom Reader between the two. We will pass the stream to a custom reader `RsyncLogStreamReader`, the custom reader will parse original logs line-by-line and create its own log output and that output will be copied to stdout:
 
```golang
	reader, err := podLogsRequest.Stream(context.Background())
	if err != nil {
		return err
	}
	rsyncLogStreamReader := RsyncLogStreamReader{
		r: reader,
	}
    // Copy() copy custom log message to stdout
    _, err = io.Copy(os.Stdout, rsyncLogStreamReader)
	if err != nil {
		return err
	}
```

```golang
func (r RsyncLogStreamReader) Read(b []byte) (n int, err error) {
	// parse logs and generate our own log output
	stdout, stderr := r.generateProgressLog(string(buf[:n]))
	...
}
```

This will allow us to generate progress information as and when logs are available on stream instead of processing all logs at a time. This improves performance, and reduces delay in progress updates. The actual progress will look like:

```golang
type progress struct {
	transferPercentage *int64
	transferRate       *dataSize
	transferredData    *dataSize
	totalFiles         *int64
	totalDirs          *int64
	transferredFiles   []string
	failedFiles        []string
	errors             []string
	exitCode           *int
}

type dataSize struct {
	val  float64
	unit string
}
```

The struct fields are self-explanatory. Use of pointers is intentional. The progress information will be parsed from rsync logs, the actual regexes used are skipped for brevity.


### Separating errors

Since Kubernetes `Stream()` API does not separate std error and output, the rsync logs are printed to stdout. Errors in rsync logs are easy to parse using well known error keywords. It is possible to separate between output and errors. `failedFiles` and `errors` fields will be printed to standard error.
 
### Dumping list of failed files

It is possible that rsync fails to transfer some files. Reasons could be bad file, i/o errors, permission issues, storage timing out, among others. In case of large number of files, having a list of failed files will help diagnose problems faster. A structured output file can also be leveraged by automation for reporting, retries, manual copying etc. Users may also find files that they don't necessarily care about, e.g. lost+found, dead symlinks, device files. To achieve that, the subcommand will add `--output=<file>` option. When enabled, a list of failed filenames will be written to an output file.

### User Stories [optional]

#### Story 1

As a user, I want to see detailed transfer progress in CLI so that I don't have to grok log lines manually.

#### Story 2

As a user, I want to see a list of files that failed transfer so that I can use it to manually copy those files over.


## Implementation History

In past, we have implemented similar progress reporting capabilities in MTC. In a controller, it is impossible to tail logs continuosly without blocking the control loop. MTC polls logs at a fixed interval instead. It only processes last 10 lines of available logs to ensure it does not block the loop. Since MTC works with multiple PVCs at a time, there is also a delay in reporting logs. Both these problems don't exist in Crane.