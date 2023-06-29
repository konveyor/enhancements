---
title: application-facts
authors:
  - "@pranavgaikwad"
reviewers:
  - "@fabianvf"
  - "@shawn-hurley"
approvers:
  - "@fabianvf"
  - "@shawn-hurley"
creation-date: 2023-06-27
last-updated: 2023-06-29
status: implementable
---

# Reorganizing application facts in analyzer output

Analyzer LSP rules have two actions - tag and message. The "message" action is responsible for creating violations in the output, while the "tag" action is responsible for creating tags. Tags don't have associated incidents. This enhancement describes a use-case around needing incidents to be associated with certain tagging rules, and proposes to enhance the existing output to accomodate this new information via a separate field in the output called "facts". 

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Motivation

In some cases, a user may want to see more information about an identified tag. For instance, I have applications that use a certain telemetery service for monitoring. I want to tag the application as "telemetery" based on a config file found in the code base, and I would also like to know where that config file is. Today, to do this, I would write following rule:

```yaml
- ruleID: find-telemetery-config
  when:
    - builtin.json:
        <logic-to-find-config>
  tag:
  - "Telemetery"
```

The above rule will generate a tag "Telemetery", but not tell me where that file is found. I will have to use a "message" action to force the analyzer to give me file URIs in incidents:

```yaml
- ruleID: find-telemetery-config
  when:
    - builtin.json:
        <logic-to-find-config>
  tag:
  - "Telemetery"
  message: "Telemetery found"
```

Above will generate tag as well as incidents in violations. While it is perfectly fine to do this, the notion of "violations" doesn't fit very well into this scenario. This is because I don't know if this is really a problem in modernization / migration context. This is additional information I want to know about my codebase and is merely informational. It will help me with docuementation, code reviews and help make dev teams aware that these config files exist. As you can see in the rule, the message is redundant, it's saying the same thing the tag says. In the dynamic reports, there is a special place for information like this called "application facts". Today, we have rules that are already giving us this kind of information, and they are all using "message" action to force analyzer to generate incidents. It'd be nice to have a separate field that clearly indicates my desire to generate incidents for this tag.

This enhancement proposes that there should be a separate field in the rule that conveys user's intention of generating incidents for tags. These incidents should not be part of violations. They should have a separate place in the output so that it's clear that they are for informational purpose only.

### Goals

- Describe a use case for needing additional information associated with tags

- Propose a new field in rule for explicitely indicating engine to generate additional information in addition to a tag

- Propose a new field in the violations output for this information

### Non-Goals

While the final outcome of this will help generate "application facts" in dynamic reports more easily, I don't want to bring that into discussion here. 

## Proposal

We should introduce a new boolean field `generateFacts` in the rule which indicates that the engine should generate more information along with a tag:

```yaml
- ruleID: find-telemetery-config
  when:
    - builtin.json:
        <logic-to-find-config>
  tag:
  - "Telemetery"
  generateFacts: true  <--- new field
```

When a match is found, the output will generate facts in addition to the original tag under a new field "facts" in the output:

```yaml
- name: konveyor-analysis
  tags:
  - Telemetery
  facts:
    Telemetery:
      - uri: file:///code/examples/telemetery.json
        codeSnip: |
          blah blah blah
  violations:
    rule-000:
      [...]
```

### Changes in structs

We will introduce a new field in rule Perform block:

`engine/conditions.go`

```golang
type Perform struct {
	Message Message  `yaml:",inline"`
	Tag     Tag `yaml:",inline"`
}

type Tag struct {
	Tag           []string `yaml:"tag,omitempty"`
	GenerateFacts bool     `yaml:"generateFacts,omitempty"`
}
```

There will be no change in providers. They continue to generate incidents as-is. 

When creating tags, the rule engine will additionally check for this new boolean flag and if it is set, it will take incidents and put them in a new field in the output:

`output/v1/konveyor/violations.go`

```golang
type RuleSet struct {
	Name        string               `yaml:"name,omitempty"`
	Description string               `yaml:"description,omitempty"`
	Tags        []string             `yaml:"tags,omitempty"`
	Violations  map[string]Violation `yaml:"violations,omitempty"`
	Errors      map[string]string    `yaml:"errors,omitempty"`
	Unmatched   []string             `yaml:"unmatched,omitempty"`
	Skipped     []string             `yaml:"skipped,omitempty"`

  // New field for facts
  // each key in the map is the associated tag
	Facts map[string][]Incident `yaml:"facts,omitempty"` 
}
```


### Upgrade / Downgrade Strategy

The new field `generateFacts` in the rule is defaulted to False. So there will be no change in existing rules that don't have it defined.

The `Facts` field in the output is omitempty and will be only populated when there is a rule that uses `generateFacts` so no upgrade / downgrade concerns there.

## Drawbacks

One minor concern I have with this is that `Incident` has some extra fields besides just the File URI that are unnecessary in the context of Facts. But I think that is fine right now, as all of them are omitted when not set.
