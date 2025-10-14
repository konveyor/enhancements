---
title: local-maven-search
authors:
  - "@jmle"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-09-18
last-updated: 2025-09-18
status: implementable
---

# Local Maven Search

This enhancement presents a potential solution for relying on a local copy of the
Maven Central Index (from now on, MCI) to query for SHAs of dependencies when analyzing
binary applications.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

TBD

## Summary

This enhancements presents a possible solution for locally bundling and querying
the Maven Index list of publicly available artifacts instead of relying on external services.

## Motivation

Up until now, Konveyor has relied mainly on HTTP queries to the [MCI search service](https://central.sonatype.com/search)
to know whether a JAR found within a Java binary is a publicly available library or not.

This is not convenient and causes issues in several scenarios:
- When the user does not have access to the internet.
- When the MCI search service is down.

Even though we have several fallback routines, like relying on a local list of open-source Group IDs or
directly parsing the POM, they are not perfect and their results leave room for improvement.

Here we will present an alternative, consisting on locally bundling the information and querying it.

### Goals

- To be able to generate the index data on demand, whenever it is required.
- To be able to deploy the data and any related artifacts with each release (if required).
- To be able to query the information quickly from the Java provider, be it in the UI or the CLI.
- To be able to consume as few resources as possible.

### Non-Goals

N/A

## Proposal

The main part of the proposal consists of a new binary with the ability to query a file with the information (Ordered list of artifacts). This file will have been previously indexed, creating an additional file (offsets file) which contains the offsets in disk of each artifact's hash in the ordered list of artifacts.

```
Analyzer                         Offsets file         Ordered list of artifacts
    | ------- getOffset(hash) ------> |                           |
    |                                 |                           |
    | <------- offset in list ------- |                           |
    |                                                             |
    | ----------------------- getArtifact(offset) --------------> |      
    |                                                             |
    | <-------------------------- artifact ---------------------- |      
```

This method is very quick and only requires disk space, eliminating the need of loading the entire index into memory,
and saving hundreds of MB of RAM. Search is done via binary search, which runs in O(log n) in the worst case.

### User Stories

#### Story 1
As a user,
I want to be able to get a full list of dependencies
When I analyze a Java binary with embedded libraries

### Implementation Details/Notes/Constraints

There are several sides to this proposal:

#### 1. Data generation
The number of artifacts available in Maven Central is constantly changing (increasing), therefore we need to be able to
regenerate the list of artifacts at will. There exists an [old Windup project](https://github.com/windup/nexus-repository-indexer)
which does exactly this, and then creates a new, simplified index for consumption in Windup. The part we are interested in is the
*.txt file(s) created as a byproduct of the Windup index generation (we are not interested in the new Lucene index created after this).

Ideally, we would want to modernize this project, since it's very old and gobbles up a lot of RAM during execution.

The process of regenerating the index could either be included in the process of generating the Java provider image,
or could be completely independent. The Java provider image build would then simply download the pre-generated data
from a Github releases link.

#### 2. Data bundling
Data could be pre-generated and potentially **compressed** for a smaller disk footprint:
```
-rw-r--r--. 1 jmle jmle 877M Sep 16 10:57 central.archive-metadata.sorted.txt
-rw-r--r--. 1 jmle jmle 504M Sep 16 11:49 index.bin
```

would become
```
-rw-r--r--. 1 jmle jmle 388M Sep 18 16:31 data.tar.gz
-rw-r--r--. 1 jmle jmle 241M Sep 18 16:30 index.tar.gz
```

This data could be directly included in the Java provider and the CLI. If compressed, the data would have to be decompressed
before the first analysis

#### 3. Data access
Accessing the data would be done through a few simple functions to query the files.


### Security, Risks, and Mitigations

No security risks are identified.

## Design Details

### Test Plan

Testing would be done by comparing a successful analysis of an older version of Konveyor with enabled Maven Search
with an analysis done by the latest version with local index data.

### Upgrade / Downgrade Strategy

Upgrading of the data would be done at first by using the Windup project mentioned earlier. Ideally, we would upgrade
and modify it so that it is more modern and suits our needs.

## Implementation History

TBD

## Drawbacks

The main drawback of this approach is the increase in the size of the image (in the case of the Konveyor UI) and the
CLI release (in the case of Kantra). My argument is that ideally we want to avoid using even more RAM memory, as
the LS already uses quite a bit.

## Alternatives

A possible alternative would be to load all the information into memory, or using a specialized DB, although these
approaches also come with their own drawbacks.

## Infrastructure Needed

No special infrastructure will be needed.