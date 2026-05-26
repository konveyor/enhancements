---
title: dvm-crane-feature-parity
authors:
  - "@pranavgaikwad"
reviewers:
  - "@jmontleon"
  - "@shawn_hurley"
  - "@alaypatel07"
approvers:
  - "@jmontleon"
creation-date: 2021-06-28
last-updated: 2021-06-28
status: implementable
see-also:
  - "../../state-only-migrations/README.md"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane state transfer feature parity with DVM

This is a follow up enhancement on [State Only Migrations](../../crane-1.x/state-only-migrations/README.md). It mainly discusses Crane 2 implementation details required to address the MTC API changes proposed for state only transfer in the previous enhancement. 

Additionally, it lists DVM features that are missing in Crane 2 and proposes solutions to fill that gap of features. This will ensure that the Crane 2 state transfer library can be seamlessly integrated into DVM. Implementation details of some of the features will be covered in separate documents in the same directory referenced here. This will allow us to merge enhancements incrementally as we make progress toward the common goal of achieving feature parity between DVM and crane-lib.
## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

This document proposes solutions to the API changes in MTC proposed for State Only Migrations in a previous [enhancement](../../crane-1.x/state-only-migrations/README.md).  This document lists the features in DVM that are missing in the Crane 2 state transfer library. It proposes solutions to implement those features in Crane library. 

## Motivation

In an effort to eventually integrate Scribe with MTC, we want to start taking little steps toward facilitating that integration. The flexibility of Crane 2 State Transfer library provides a natural path to first integrate the library with MTC and then integrate with Scribe.  

### Goals

- Propose implementation details of proposed MTC API changes proposed in [State Only Migrations](../../crane-1.x/state-only-migrations/README.md)

- List features in DVM that are currently missing in Crane 2 library and propose solutions to implement them in Crane library

### Non-Goals

- Propose integration path for Crane State Transfer into DVM

## Proposal

#### Mapping PVs

In state only migrations, one of the important features is to let users migrate data into a pre-provisioned PV. Currently, crane-lib assumes that there will be only one PVC reference for both the source and the target. The target PVC is created from the source PVC definition preserving the namespace and the name of the object. We will update the `Transfer` interface to take two different PVCs - a source and a target:

_[transfer.go](https://github.com/konveyor/crane-lib/blob/a5ead85cb740c0325f2797c23a5cbb74f8972220/state_transfer/transfer.go#L14-L33)_

```golang
type Transfer interface {
	[...]

	SetSourcePVC(*v1.PersistentVolumeClaim)
	SourcePVC() *v1.PersistentVolumeClaim
	SetDestinationPVC(*v1.PersistentVolumeClaim)
	DestinationPVC() *v1.PersistentVolumeClaim
}
```

The `RsyncTransfer` struct will be updated to adhere to the new interface by breaking down existing `pvc` reference into source and destination: 

```golang
type RsyncTransfer struct {
    [...]

    sourcePVC      *v1.PersistentVolumeClaim
    destinationPVC *v1.PersistentVolumeClaim
}
```

_Please note that the existing absolute objects are replaced with a memory reference instead to allow 'nil' checks by the consumer_

With the proposed change above, the Rsync Transfer Server will be updated to use Source PVC reference while the Client will be updated to use Destination PVC reference. Same applies to the Transport server & client as well. I may have used server and client interchangeably, but the point is to update in right places.


#### Allowing custom labels and custom Rsync options for Rsync Transfer

The custom labels and the custom command options can be envisioned as additional configuration options passed by the consumer to the library. They can be consolidated in a struct called `RsyncTransferOptions`:

_options.go_

```golang
package rsync

// TransferOptions defines customizeable options for Rsync Transfer
type TransferOptions struct {
	CommandOptions
	sourceResourceMetadata      transfer.ResourceMetadata
	destinationResourceMetadata transfer.ResourceMetadata
}

// TransferOption
type TransferOption interface {
	ApplyTo(*TransferOptions) error
}

// CommandOptions defines options that can be customized in the Rsync command
type CommandOptions struct {
	Recursive    bool
	SymLinks     bool
	Permissions  bool
	ModTimes     bool
	DeviceFiles  bool
	SpecialFiles bool
	Groups       bool
	Owners       bool
	HardLinks    bool
	Delete       bool
	Partial      bool
	BwLimit      int
	Info         []string
	Extras       []string
}
```
Struct _ResourceMetadata_ defines extra customizeable information that can be used while creating any temporary resource used by State Transfer.

```golang
// ResourceMetadata defines options that let consumers customize different metadata placed on Rsync resources
type ResourceMetadata struct {
	Labels            map[string]string
	Annotations       map[string]string
	OwnerReferences   []metav1.OwnerReference
	SecurityContext   *corev1.SecurityContext
}
```

The `RsyncTransferOptions` struct will be a part of `RsyncTransfer`:

_common.go_

```golang
package rsync

type RsyncTransfer struct {
	[...]
	options     RsyncTransferOptions
}
```

An additional interface `RsyncTransferOption` will be defined:

_options.go_

```golang
package rsync

type RsyncTransferOption interface {
	ApplyTo(*RsyncTransferOptions) error
}
```

This interface will prove useful in cases where we want to provide users with a set of pre-defined options like below: 

```golang
type ArchiveFiles bool

func (rca ArchiveFiles) ApplyTo(opts *TransferOptions) error {
	opts.Recursive = bool(rca)
	opts.SymLinks = bool(rca)
	opts.Permissions = bool(rca)
	opts.ModTimes = bool(rca)
	opts.Groups = bool(rca)
	opts.Owners = bool(rca)
	opts.DeviceFiles = bool(rca)
	opts.SpecialFiles = bool(rca)
	return nil
}
```

The method that creates a new Rsync Transfer will take a list of options as parameter:

_common.go_

```golang
package rsync

func NewTransfer(t transport.Transport, e endpoint.Endpoint, src *rest.Config, dest *rest.Config,
	pvc corev1.PersistentVolumeClaim, opts ...TransferOption) (transfer.Transfer, error) {
	options := TransferOptions{}
	err := options.Apply(opts...)
	if err != nil {
		return nil, err
	}
	return &RsyncTransfer{
		transport:   t,
		endpoint:    e,
		source:      src,
		destination: dest,
		pvc:         pvc,
		options:     options,
	}, nil
}
```
#### Rsync Retry

DVM provides a retry mechanism for Rsync Pods that allows retrying a failing Rsync Transfer to overcome intermittent network failures. The number of retries can be configured. DVM will retry Rsync either until it is succesful or the number of specified retries is met. In an effort to speed up integration with crane-lib, this will be solved in phases. At the end of each phase, we will ensure that the Rsync Transfer flow is end-to-end complete. Without that, we will not implement the next phase. 

In the first phase, we will implement the custom label options discussed in the previous section in crane-lib. Rsync Retry mechanism depends on using custom labels on the Rsync Pods created. Once this phase finishes, we will simply replace DVM code that creates Rsync Pods with crane-lib function calls. We will keep the logic of maintaining count of retries in DVM. This will allow us to integrate with DVM in the early phase and begin testing. 

In the second phase, we will gradually bring the Rsync retry logic in crane-lib. When this phase finishes, DVM will simply pass on all the Retry related options to the library. At this point, DVMP will still work as before. It will use the label selector information passed to crane-lib to observe progress of Rsync Pods.

In the last phase, we will also bring the DVMP's logic in the library. At the end of this phase, we will simply call crane-lib functions to get progress information of Rsync Pods in DVM.

This enhancement will be updated at every phase to include implementation details.

### Risks and Mitigations

#### Rsync Retry Timeline

Due to involvement of DVMP and its effects on MTC UI, Rsync Retry is not as straight-forward as other features explained in this doc. It is possible that we won't be able to bring all of the retry logic within MTC 1.6 timeframe. Therefore, the phased approach is necessary. In each phase, we will test end-to-end Rsync flow in MTC and the phase will only be called complete when the Rsync functionality meets all the criteria as defined by MTC (which means running MTC e2e test cases).
