# Enhancements Tracking and Backlog

Enhancement tracking repository for Konveyor.

Inspired by the [Kubernetes enhancement](https://github.com/kubernetes/enhancements) process.

This repository provides a rally point to discuss, debate, and reach consensus
for how Konveyor [enhancements](./enhancements) are introduced.  Konveyor encompasses
several projects (Container Migration, Virt Migration) each have sub-repos. Coordinated
change across repos is often needed to implement new features, and this repo serves as 
a centralized place to describe Konveyor enhancements via an actionable design
proposal.

Enhancements may take multiple releases to ultimately complete and thus provide
the basis of a community roadmap.  Enhancements may be filed from anyone in the
community, but require consensus from domain specific project maintainers in
order to implement and accept into the release.

For a quick-start, FAQ, and template references, see [the guidelines](guidelines/README.md).

## Is My Thing an Enhancement?

A rough heuristic for an enhancement is anything that:

- impacts how Konveyor projects are operated including addition or removal of significant
  capabilities
- needs significant effort to complete
- requires consensus/code across multiple domains/repositories
- demands formal documentation to utilize

It is unlikely to require an enhancement if it:

- fixes a bug
- adds more testing
- internally refactors a code or component only visible to that components
  domain
- minimal impact to Konveyor project as a whole

If you are not sure if the proposed work requires an enhancement, file an issue
and ask!

## When to Create a New Enhancement

Create an enhancement here once you:

- have circulated your idea to see if there is interest
- (optionally) have done a prototype in your own fork  

## Why are Enhancements Tracked

As the project evolves, its important that the Konveyor community understands how we
build, test, and document our work.  Individually it is hard to understand how
all parts of the system interact, but as a community we can lean on each other
to build the right design and approach before getting too deep into an
implementation.

## When to Comment on an Enhancement Issue

Please comment on the enhancement issue to:
- request a review or clarification on the process
- update status of the enhancement effort
- link to relevant issues in other repos

## Code of Conduct
Refer to Konveyor's Code of Conduct [here](https://github.com/konveyor/community/blob/main/CODE_OF_CONDUCT.md).
