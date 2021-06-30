---
title: Adding Requeue in Long running phase - AnnotateResources
authors:
  - "@JaydipGabani"
reviewers:
  - "@jortel"
  - "@djwhatle"
  - "@alaypatel07"
  - "@dymurray"
  - "@pranavgaikwad"
approvers:
  - "@jortel"
  - "@djwhatle"
  - "@alaypatel07"
  - "@dymurray"
  - "@pranavgaikwad"
creation-date: 2020-11-04
last-updated: 2020-11-04
status: implementable
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Adding Requeue in Long running phase - AnnotateResources

The goal of this document is to highlight user experience issues in the current implementation of progress reporting in MTC, and propose improvements in some of the areas for a better user experience. The driving goal of the effort is to ensure that the user has enough information in their hands to understand what exactly is happening in the background of an ongoing migration phase AnnotateResources.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined

## Summary 

With the proposed changes, migration controller will periodically halt the process of phase `AnnotateResource` and update `mig_migration` CR and Resume the process on the next reconcile.

## Motivation

Currently `AnnotateResource` phase runs for loops to annotate or label resources. Currently this happens in a single reconcile. The whole phase takes few seconds to several minutes depending on the number of resources included in the migration. This can leave user without any sense of progress. 

## Proposal

This design doc proposes an approach to add `Requeue` logic to such long running phase to update user about the progress while the whole phase is getting processed.

We propose the following modifications in the function responsible for annotating.

```
total := len(allResource)
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
    itemsUpdated ++
    if itemsUpdated > 50 {
        t.Progress = []string{fmt.Sprintf("%v/%v Namespace labeled", i, total)}        
        return nil
    }
}

```
Changes in `annotateStageResources()` function
```
itemsUpdate := 0
// Namespaces
itemsUpdate, err = t.labelNamespaces(sourceClient, itemsUpdate)
<...>
if itemsUpdate > 50 {
  t.Requeue = FastReQ
  return nil
}
// similar code for all the other resource processing

```
Changes in task.go file in `AnnotateResource` switch case:

```
if t.Requeue != FastReQ {
    if err = t.next(); err != nil {
        return liberr.Wrap(err)
    }
}
```

The above code will requeue periodically to report progress and then continue the loop to annotate resource. We are checking if label exists or not before hand to make sure every resource gets processed only once.
