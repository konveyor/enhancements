---
title: dynamic-pipeline
authors:
  - "@pranavgaikwad"
reviewers:
  - "@djwhatle"
  - "@alpatel"
approvers:
  - "@ernelson"
  - "@djwhatle"
  - "@alpatel"
creation-date: 2021-02-02
last-updated: 2021-02-04
status: implementable
see-also:
  - "../progress-reporting/README.md"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Dynamic Pipeline

Progress reporting [enhancement](../progress-reporting/README.md) introduced the idea of a Migration Pipeline in MTC. While the pipeline greatly improves observability and the overall experience, there is still room for improvement in the pipeline implementation. This enhancement document highlights some of the areas of improvement in the pipeline. 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

> Does flag evaluation depend on outcome of previous migration phases? 

## Summary

This enhancement addresses following problems in the pipeline:

1. Some steps have confusing names. For instance, StageBackup actually involves PV and Image backup, it runs even during a final migration. Since `Stage` has a special meaning in the context of migrations, the terminology of stage steps creates confusion. 

2. Some steps in the pipeline are marked `Skipped` when they are not executed which may add to confusion.

## Motivation

### Goals

1. Propose new names for some of the pipeline steps.

2. Do not include steps in the pipeline which will be eventually skipped.

### Non-Goals [omitted]

## Proposal

### Goal 1: Propose new names for some of the pipeline steps

`StageBackup` can be renamed to `DataBackup` or `StateBackup` as Stage backup is responsible for migrating both images and persistent volume data. `StageRestore` can be renamed to `DataRestore` or `StateRestore`. This will make sure that we dont have conflicting meaning to the term `stage`. 

### Goal 2: Do not include skipped steps in the pipeline

In migration itineraries, when a phase finishes, the next phase in the itinerary is returned by the `t.next()` function. This function internally applies `all` or `any` flags on the phase to determine whether the next phase will be executed or skipped. The current pipeline implementation depends on output from `t.next()` and skips a particular step when none of the phases included in that step are executed. This is done at run-time. This can be changed and the migration pipeline can pre-determine the inclusion / exclusion of steps using the same flags that `t.next()` uses. 

`initPipeline()` function runs in every reconcile before execution of a phase. This function is responsible for reconciling state of `Status.Pipeline` with its in-memory counterpart:

`initPipeline()`
``` 
1. Read current itinerary, step and phase from MigMigration

2. Iterate over all phases in the current itinerary 

  - Add the associated step of phase to the pipeline if it doesn't exist already
```

The above logic can be updated to:

```
1. Read current itinerary, step and phase from MigMigration

2. Iterate over all phases in the current itinerary 

  - Apply allFlags() and anyFlags() on the phase to determine whether this phase is actually part of migration
  - If the phase is included in the migration, add the associated step of phase to the pipeline if it doesn't exist already
```

```go
func (t *Task) initPipeline() {
	for _, phase := range t.Itinerary.Phases {
		currentStep := t.Owner.Status.FindStep(phase.Step)
		if currentStep != nil {
			continue
		}
		allFlags, _ := t.allFlags(phase)
		if !allFlags {
			continue
		}
		anyFlags, _ := t.anyFlags(phase)
		if !anyFlags {
			continue
		}
		t.Owner.Status.AddStep(&migapi.Step{
			Name:    phase.Step,
			Message: "Not started",
		})
  }
  ...
}
```

This will add significant time since evaluation of some flags takes several seconds and `initPipeline()` runs every time the controller reconciles. Therefore, the loop can be updated to only trigger when there's a change in the itinerary:

```go
func (t *Task) initPipeline(prevItinerary string) {
  if prevItinerary != currentItinerary {
    for _, phase := range t.Itinerary.Phases {
	  	[...]
    }
  }
  ...
}
```

Above will make sure that the pipeline is built _only once_ per itinerary even though the function is called in every reconcile. 


### User Stories [omitted]


### Implementation Details/Notes/Constraints [omitted]

### Risks and Mitigations [omitted]

## Design Details

### Test Plan

#### E2E tests

##### Test 1: Perform an indirect image migration

In this case, `DirectImage` step should not be seen in the pipeline.

##### Test 2: Perform an indirect volume migration

In this case, `DirectVolume` step should not be seen in the pipeline.

##### Test 3: Perform a direct image migration

In this case, `StageBackup` and `StageRestore` steps should not be seen in the pipeline.


### Upgrade / Downgrade Strategy

#### What happens to existing migrations? 

If existing migrations have steps that are marked `skipped`, those migrations *may* create some problems. I will update the doc once I have better idea about it. 

## Implementation History [omitted]

## Drawbacks [omitted]

## Alternatives [omitted]

## Infrastructure Needed [omitted]

