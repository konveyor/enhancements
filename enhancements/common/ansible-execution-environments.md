---
title: ansible-execution-environments
authors:
  - "@pemcg"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: yyyy-mm-dd
last-updated: yyyy-mm-dd
status: provisional
see-also:
  - 
replaces:
  - 
superseded-by:
  - 
---

# Use Ansible Execution Environments for Hooks

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Two of the Konveyor projects - Crane and Forklift - have a requirement to run pre and post migration _hooks_ (usually Ansible playbooks) for per-instance configuration either side of the container or VM migration.

These playbooks need to run podified, from container images built to contain the playbooks and all dependencies such as Python, Ansible Runner, collections, and user-defined roles. They will be run using `ansible-runner`. This is conceptually very similar to an Ansible _Execution Environment_ (see [Execution Environments](https://ansible-runner.readthedocs.io/en/stable/execution_environments.html)).

This enhancement proposes that Konveyor projects use the established Ansible Execution Environment (EE) technology for Ansible-based hooks, along with [Ansible Builder](https://ansible-builder.readthedocs.io/en/latest/) to build the hook images.

## Motivation

The Forklift project has a requirement to create an Ansible Hooks Image Builder. The outline of this requirement is as follows:
##### Goal
Provide users a way to use the Git repositories where they store Ansible Playbooks as a source to build Images to be used for automation before and after a VM migration.
##### Background
Automation will be run as a container image from the registry. We will provide a way to run Ansible playbooks by building the images containing them.
We want to be sure that the playbooks being run in a plan are not modified during the plan. Therefore the image has to be created, stored and run, not being modified during the run or afterwards.
##### User Stories
As a migration owner, when creating a migration plan, I would like the ability to add some Ansible playbooks for automation to be executed before or after a VM is migrated.
##### Implementation Notes
Source for playbooks will be a git repository (including branch)
In the repository we may find more than one playbook. We will provide a way to select the playbook to be run.
##### Acceptance Criteria
I can provide a git repository with a playbook to be run. 
An image will be generated from that repository taking into account branch and available playbooks in it. 
I will be able to choose for my Migration Plan, the generated image which will be run. I will be able to choose if it will run before or after migration.

### Goals (of this proposal)

1. The main goal in the implementation of this Forklift story is to re-use as much of the existing Execution Environment and ansible-builder technology as possible. This will allow for:

 - Use existing technologies - no wheel re-invention.
 - Less custom development and maintenance effort for Konveyor.
 - Commonality of technologies between Ansible & Konveyor projects.
 - Greater end-user familiarity with the technology being used.

2. Creating an EE-based hook image using the UI components added to Forklift should be the easiest way for users to add hook images - the path of least resistance for hook implementation. Creating Ansible-based hook images manually or via alternate means should still be possible if a user wishes and has knowldge to do so (but this should be discouraged/deprecated).

3. The implementation of this automation hook architecture for the Forklift project should be shareable and useable by other Konveyor projects.

### Non-Goals

1. Considerable development of the Execution Environment, `ansible-runner`, or `ansible-builder` technology.
2. Preventing other executables from being run in a hook image (golang binary, bash script, etc.).

## Proposal

### User Stories

The following user stories can be considered for this proposal.

#### Story 1

As a migration user I would like to use an Ansible Execution Environment image that I have already created as a pre/post migration hook.

#### Story 2

As a migration user hook developer I would like to use the `ansible-builder` tool that I'm already familiar with to build my hook image.

### Implementation Details/Notes/Constraints

The following components will probably be required:

- UI support
- Hook definition CR
- Hook execution CR
- Hook controller

Some contributions may need to be made to the [ansible-builder](https://github.com/ansible/ansible-builder) repository.

#### Notes

`ansible_builder` uses 5 files to create and build the image: 

- `execution-environment.yml` <--- ansible-bulder configuration file
- `ansible.cfg` <--- normal Ansible .cfg file
- `bindep.txt` <--- optional platform rpm content needed in the image
- `requirements.txt` <--- optional pip package dependencies needed in the image
- `requirements.yml` <--- optional ansible collections needed in the image

The Konveyor projects will need a UI extension to point to a Git repository that contains the required configuration files, and well as roles and playbooks for the hook. The resultant hook image will be saved to the local cluster registry.

#### Constraints

The Ansible Automation Platform tool for running execution environments is `ansible-navigator `([GitHub Repository](https://ansible-navigator.readthedocs.io/en/latest/) & [Docs](https://ansible-navigator.readthedocs.io/en/latest/)). The Konveyor projects would probably not use this tool as it assumes that the playbook(s) to be run are outside of the Execution Environment image. 

In the Konveyor implementation it is proposed that the hook playbook(s) are included in the EE image, and that `ansible-runner` within the EE container is called directly, passing inventory and playbook path parameters as necessary on the run command line.

### Risks and Mitigations

TBD

## Design Details

TBD

## Implementation History

N/A

## Drawbacks

This proposal potentially ties the Konveyor project to the release functionality of an external project.

## Alternatives

Implement a Forklift-specific hook image builder.

