
---
title: MTC Progress Reporting Improvements
authors:
  - "@pranavgaikwad"
  - "@JaydipGabani"
reviewers:
  - "@jortel"
  - "@djwhatle"
  - "@alaypatel07"
  - "@dymurray"
approvers:
  - "@jortel"
  - "@djwhatle"
  - "@alaypatel07"
  - "@dymurray"
creation-date: 2020-11-03
last-updated: 2020-11-03
status: implementable
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Adding Requeue in Long running phase like AnnotateResources

The goal of this document is to highlight user experience issues in the current implementation of progress reporting in MTC, and propose improvements in some of the areas for a better user experience. The driving goal of the effort is to ensure that the user has enough information in their hands to understand what exactly is happening in the background of an ongoing migration.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined

Currently `AnnotateResource` phase runs for loops to annotate or label resources. Currently this happens in a single reconcile. The whole phase takes few seconds to several minutes depending on the number of resources included in the migration. This can leave user without any sense of progress. This design doc proposes an approach to add `Requeue` logic to such long running phase to update user about the progress while the whole phase is getting processed.

We propose the following modifications in the function responsible for annotating.

```
total := len(allResource)
annotation := 0
for i, ns := range allResource() {
    retrieve single resource
    .
    .
    .

    if _, exist := <resource>.GetLabels()[IncludedInStageBackupLabel]; exist {
			continue
	}

    set annotations
    .
    .
    .
    annotation ++
    if total >= 5 && annotation == total / 5 {
        t.Requeue = PollReQ
        return nil
    }
    t.Progress = []string{fmt.Sprintf("%v/%v Namespace labeled", i, total)}    
}

```
Changes in task.go file

```
if t.Requeue != PollReQ {
    if err = t.next(); err != nil {
        return liberr.Wrap(err)
    }
}
```

The above code will requeue periodically to report progress and then continue the loop to annotate resource. We are checking if label exists or not before hand to make sure every resource gets processed only once.
