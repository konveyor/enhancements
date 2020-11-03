### Adding Requeue in Long running phase like AnnotateResources

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