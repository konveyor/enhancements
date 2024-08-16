---
title: declarative-ci
authors:
  - "@djzager"
  - "@shawn-hurley"
  - "@aufi"
reviewers:
  - "@dymurray"
  - "@jmontleon"
approvers:
  - "@dymurray"
  - "@jmontleon"
creation-date: 2024-08-15
last-updated: 2024-08-15
status: implementable
---

# Declarative CI

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [n/a] Test plan is defined
- [n/a] User-facing documentation is created

## Open Questions

1. Is this a good idea? Hopefully, this is a placeholder for good open questions.

## Summary

The Konveyor project relies very heavily on GitHub Actions and Workflows to
accomplish it's CI/CD goals. A non-exhaustive list includes:

1. [release-tools/.github/workflows](https://github.com/konveyor/release-tools/tree/4b531552fd31b63e4e1ee55c70be86d327a08da7/.github/workflows)
    is home to a handful of reusable workflows useful for handling PRs that need
    to be cherry-picked, building and pushing multi architecture images,
    creating the release notes for a specific repository, or reconciling issues
    and issue comments.
1. [release-tools/pkg/config/config.yaml](https://github.com/konveyor/release-tools/blob/4b531552fd31b63e4e1ee55c70be86d327a08da7/pkg/config/config.yaml)
    contains a list of the repositories, labels, and milestones across Konveyor.
    This file is currently used to ensure all repositories have the same labels
    and milestones. It looks like:
    ```yaml
    # Repos
    # List of repositories we are managing. We may, in the future, worry about
    # repositories in konveyor-ecosystem.
    #
    # repos:
    #   - org: the organization of the repo
    #     repo: the repo
    repos:
      - org: konveyor
        repo: operator
    ...

    # Labels
    # List of labels, their color and description, that should exist in the specified repositories.
    #
    # labels:
    # - color: the color of the label
    #   description: what does it mean?
    #   name: the name of the label
    labels:
      # Triage
      - color: ededed
        description: Indicates an issue or PR lacks a `triage/foo` label and requires one.
        name: needs-triage
    ...

    # Milestones
    # List of milestones, and their state, that should exist in the specified repositories.
    #
    # milestones:
    #   - title: the title for the milestone
    #     description: the description
    #     state: open/closed
    #     due: 
    #
    milestones:
      - title: 0.3-beta.2
        description: The second beta for v0.3.0 release cycle
        state: closed
        due: 2023-11-02
    ...
    ```
1. [ci/.github/workflows](https://github.com/konveyor/ci/tree/0f3e8c01a8604d240954e3b2a33f0e0dc3cdc684/.github/workflows)
    is where the global-ci(-bundle) reusable workflows are stored. These are
    used in testing incoming pull requests across the project as well as gating
    releases before they are published.
1. [operator/.github/workflows/create-release.yml](https://github.com/konveyor/operator/blob/2a4a88f646643790ad5761260cca49e8646db082/.github/workflows/create-release.yml)
    is responsible for creating the release in the individual component
    repositories (in a specified order so base images are available), waiting for
    those images to be published by repositories' container image build workflow,
    creating an OLM bundle image with the component images pinned to their
    respective SHA digest, testing with this bundle, and finally publishing to
    [community-operators](https://github.com/k8s-operatorhub/community-operators)
    and
    [community-operators-prod](https://github.com/redhat-openshift-ecosystem/community-operators-prod/).
    It's worth noting that the version (ie `v0.5.1`) and the branch (ie
    `release-0.5`) are inputs to the release pipeline.

To accomplish this, there is the @konveyor-release-bot and the [Konveyor CI
Bot](https://github.com/organizations/konveyor/settings/apps/konveyor-ci-bot)
that are used extensively to perform work across the organization.

## Motivation

There are several problems with how repos are currently managed in the Konveyor
project.

1. **Difficult to add new repositories to the release.** Every repo that is to
   be added to the release must have the
    [ci-release-engineering](https://github.com/orgs/konveyor/teams/ci-release-engineering)
    team added as a contributor with write permissions, have the
    `quay.io/konveyor/${repo}` created + made public + with write access granted to
    the `konveyor+publish_github_action` robot account, the release-tools
    config.yaml updated, and the release pipeline updated in konveyor/operator.
    This is in addition to the work inside the project to make sure it contains all
    of the necessary workflows to perform CI on pull requests, handle issues and
    issue comments, and build and publish images when commits are merged into
    mainline (or new tags are created).
1. **Tribal knowledge.** Related to the previous point, very few contributors
    actually know what is required to get new repos into the release.
1. **Only releases are pinned.** You can grab a published release and all images
    will have been pinned to the time when the release was created. However,
    operator bundles built from release branches (like `release-0.5`) and `main`
    still use floating tags.
1. **Components are only built in correct order in release pipeline.** For example,
    [analyzer-lsp](https://github.com/konveyor/analyzer-lsp/blob/974a5d29809e5c5ef698ce48f497d0f8f86a1939/Dockerfile#L32)
    relies on jdtls-server-base but we only force jdtls-server-base built before
    analyzer-lsp in the release pipeline.

### Goals

* Make the CI/CD more accessible to Konveyor contributors
* Standardize Konveyor component repository release artifacts
* Improve our ability to know what changes are in which images
* Encode our standards in such a way that they are enforcable across the
    Konveyor project

### Non-Goals

* Move to prow or Jenkins or any other CI/CD system

## Proposal

This proposal suggests that we make use the config.yaml (currently in
[release-tools](https://github.com/konveyor/release-tools/blob/4b531552fd31b63e4e1ee55c70be86d327a08da7/pkg/config/config.yaml))
a declarative source of truth for how the Konveyor project is built, released,
and published. With this config.yaml, we declare all of the relationships
between components as a Directed Acyclic Graph (DAG) and start using automation
to "reconcile" this desired state (in the config) with the actual state
(version controlled in individual component repositories).
We have already started this type of reconcile behavior with how we handle
[milestones](https://github.com/konveyor/release-tools/blob/4b531552fd31b63e4e1ee55c70be86d327a08da7/.github/workflows/_push-main.yml#L37-L58)
and
[labels](https://github.com/konveyor/release-tools/blob/4b531552fd31b63e4e1ee55c70be86d327a08da7/.github/workflows/_push-main.yml#L60-L81)
when they are changed in the config.yaml.

### Encode the DAG

Currently, the
[`Repo` type in release-tools](https://github.com/konveyor/release-tools/blob/4b531552fd31b63e4e1ee55c70be86d327a08da7/pkg/config/types.go#L12-L16)
only holds the the `Org`anization and `Repo`sitory coordinates of a repository.
We will extend it to include a list of repositories on which this repo depends.
So an example entry might look like:

```yaml
  - org: konveyor
    repo: analyzer-lsp
    dependencies:
      - org: konveyor
        repo: java-analyzer-bundle
```

This tells us that the java-analyzer-bundle repository must be built before
analyzer-lsp.

### Generate release pipeline programmatically

Currently, every new repository added to the Konveyor project as a component
requires an update to the config.yaml (to get labels and milestones) and then
another to the create release workflow in the operator repository. Once we have
the DAG encoded in our config.yaml, we can use that file to programmatically
generate a create release workflow that will 1) live with the release tools 2)
be versioned just as the Konveyor project is versioned and 3) be sanity checked
on PRs to the release-tools repository.

With the release-tools repository also being subject to our branching strategy,
we will take the `version` and `previous_version` as inputs but we will take the
`branch` from the `ref_name` on the
[github context](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs#github-context).

### Move all CI to the `global-ci-bundle` workflow

We developed the `global-ci-bundle` workflow (and the reusable actions inside)
to enable the operator to be effectively overriden in our test environments.
Now we can standardize on operator bundles being the artifact representing a
Konveyor for the purposes of testing and automation. We will move all PR
checks/CI workflows going forward to use the `global-ci-bundle` and remove the
actions + workflows we no longer support.

### Pin nightly operator bundle on failure

We will pull out the image mirroring, tagging, and pinning by SHA digest out of
the create release pipeline so that we can use it when Nightly CI fails. Once
we update the Slack notification to include this image, contributors will be
able to pull this image to verify regressions locally.

A useful thing to add here would be to inspect the images (and their bases) for
the source and revision information

### Use PR Checks to ensure New Repositories are configured correctly

To address the tribal knowledge concerns of new repositories being added to
Konveyor, we will add PR checks to the release-tools repository that ensure
each of the repositories in the config are properly configured; ready to be
published as a component of Konveyor.

### Trigger rebuilds based on repo dependencies

With the DAG, we can update every repositories image build workflow to emit an
event in release-tools when it is complete. We will write a new workflow in the
release-tools repository that handles these events and triggers the image build
workflows in each of the dependant projects.

## Drawbacks

While having a single source of truth in the config.yaml is an improvement in
the transparancy of how CI/CD is done in the Konveyor project. However, it will
not be immediately obvious to new contributors how a new component would be
onboarded. We believe this can be mitigated by the increased activity to the
release-tools repository these changes will necessitate.

## Alternatives

We could seriously pursue something like Prow or some other build system meant
to handle multi-repository projects like ours. However, the amount of
engineering time required to pursue (and maintain) something like Prow is more
than we could bear without a definitive improvement in the drawback(s) stated
above.
