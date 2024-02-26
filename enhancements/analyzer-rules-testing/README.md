---
title: Analyzer - Testing Rules
authors:
  - "@pranavgaikwad"
reviewers:
  - "@djzager"
  - "@fabianvf"
  - "@jmle"
  - "@shawn-hurley"
approvers:
  - "@djzager"
  - "@fabianvf"
  - "@jmle" 
  - "@shawn-hurley"
creation-date: 2024-01-22
last-updated: 2024-01-22
status: implementable
---

# Analyzer - Testing Rules

As of today, testing an analyzer rule involves creating a provider config file, running analysis using the rule against a sample application, finally verifying the results by inspecting the output file manually. 

This enhancement proposes a new test runner for YAML rules. It will enable rule authors to write tests for rules declaratively, run analysis using the rules against sample data, verify that the expected output in the test matches the actual output and finally, produce a result detailing passed / failed tests.

## Motivation

The biggest drawback with testing rules right now is that the output must be verified by the rule author manually. They have to go through all incidents / violations and verify if they see a desired result. Once they do verify the output, the only way to share this expected output is to share the entire _output.yaml_ file generated post analysis. Comparing the expected output with the actual results is done by computing _diff_ between the expected and actual output files. While this works great in end-to-end CI tests that run on a single rules file, there are limitations. First, it's very hard to understand what exactly is failing by only looking at diff output. And second, you have to generate that very first "expected output" file by running the analyzer. You cannot expect someone to write that file by hand. 

Assume that you're a rule author creating rules for a certain technology. You write your first rule, you run it against sample application, manually verify that your rule matched in the `output.yaml` file. You count the number of incidents, check if produced messages are correct. You check for any tags created if applicable. All good! You move onto writing your second rule, you run the rules file again and get an output file. This time, you ensure that the first rule still produces the same output as before in addition to verifying your new rule. In every iteration, you manually verify every rule in the ruleset. Ideally, we want the rule author to write a test first and then iteratively work towards perfecting their rule until it produces the desired output. It's necessary that they can write the test by hand and on top of that, they should be able to run a command to verify the results.

In addition to the UX concern above, there is another important concern. As analyzer-lsp adds more and more providers, contribution to rulesets will increase. For the long-term health of the project, it is essential that any new rules added to rulesets have associated tests and sample data. Manual testing process will only discourage rule authors from submitting tests.

Lastly, the project has been mainly relying on rules from the Windup project which all have associated test cases. At some point, we will discontinue the Windup shim and move onto the new YAML format for good. What happens to the existing test cases in that case? We need to make sure we can convert the existing tests into this new format and continue to have coverage for those old rules.

### Goals

- Propose a declarative way of writing a test for a rule

- Propose a CLI tool to run the tests

- Talk about general concerns around requiring tests in konveyor/rulesets repo and CI

### Non-Goals

- Discuss specific implementation details of the test runner

### Proposal

#### Writing a test for a rule

Rule authors should should be able to "declare" a test for a rule in a YAML file:

```yaml
providers:
- name: java
  location: ./java-sample-data/
- name: go
  location: ./go-sample-data/
tests:
- ruleID: same-rule-id-as-the-rule-to-be-tested
  testCases:
  - description: In source-only mode, should match 10 times
    analysisParams:
      mode: source-only
      depLabelSelector: "!konveyor.io/source=open-source"
    hasIncidents:
      and:
      - fileURI: "<file/uri/where/expected/incidents/should/be/present>"
        exactly: 10
        atLeast: 5
      - fileURI: "<file/uri/of/expected/incident/with/custom/variables>"
        locations:
        - lineNumber: 8
          messageContains: "incident on line number 8 should contain this"
          codeSnipContains: "code snippet should contain this text"
    hasTags:
    - Category=tag1
```
* providers: List of configs needed to setup providers. The runner will take a settings file at runtime and mutate it with values defined here for every run. This can also be defined at ruleset level. Values here will take precedance:
  * name: Name of the provider to match in _provider_settings.json_
  * location: Path to sample data for this provider
* tests: List of tests, each testing a distinct rule
  * ruleID: ID of the rule this test applies to. 
  * testCases: A list of different test cases for this test with each case containing following fields:
    * description: This is what will be printed as text next to PASS / FAIL result
    * analysisParams: Additional _optional_ parameters to configure analysis when running this test. By default, these won't be set.
      * mode: Analysis mode, defaulted by analyzer-lsp to provider default.
      * depLabelSelector: Dependency label selector, defaulted to nil.
    * hasIncidents: This defines how incidents should be verified. There could be multiple ways verification can be done for a test case. So this is a nested structure just like how we define _when_ block of a rule. It has exactly one _IncidentCondition_.
      * _IncidentCondition_ is defined for a _fileURI_. It will apply to a specific file. In a given file, the incidents can be verified in one of several ways:
        * _Simple Verification_: This simply checks if the given file contains exactly given number of incidents. Exactly one of the following parameters can be defined for this type of verification:
          * count: there should be exactly this many incidents.
          * atLeast: there should at least this many incidents, more are OK.
          * atMost: there should at most this many incidents, less are OK.
        * _Per incident verification_: This is defined on a per incident basis and takes a list of code locations as input. Each code location will take following parameters:
          * lineNumber: incident should be found on this line
          * messageContains: message should contain this string
          * codeSnipContains: code snippet should contain this string
      * _and_ and _or_ are two special type of _IncidentCondition_ that can be used to form a complex test.
    * hasTags: A list of tags to verify. The test will FAIL when any one or more tags in the list are not found in the output.

Notice that there could be more than one _testCases_ in a test with different _analysisParams_. This is useful when you want to test the same rule but under different analysis parameters. 

A testcase passes when all of the following conditions are met:

- either one of _hasTags_ or _hasIncidents_ is defined
- _hasTags_ is not defined OR all given tags are present in the output.
- _hasIncidents_ is not defined OR all of the following is true:
  - _count_ is equal to the actual count of incidents found in the output
  - _messageContains_ is not defined OR (it is defined AND message is found in all incidents)
  - _codeSnipContains_ is not defined OR (it is defined AND code snip is found in all incidents)

_If and / or are used for incident verification, the result of the top level condition will be evaluated based on results of all children accordingly_

> Having a schema for the tests would be nice. It will allow users to auto-generate the boilerplate.

#### Organizing test files in a ruleset

The YAML file we described in the previous section will be saved in the same directory as its rules file. The name of this test file will be same as of the rules file appended with `_test` suffix. For instance, tests for a rules file `rules.yaml` will be stored in `rules_test.yaml` in the same directory. The analyzer-lsp engine will ignore any files that have `_test.yaml` during rules parsing. 

The providers can also be defined at a ruleset level in case users just want to re-use for all tests in a rulest. A new field `testConfig` can be introduced in `ruleset.yaml` file:

```yaml
name: my-ruleset
testConfig:
  providers:
  - name: java
    location: /path/to/java/
```

> When a specific provider config is defined in a test file, that config will take precedance over the ruleset level config

#### Running the tests

We are proposing a new CLI that takes the tests as input. 

To run a single test file `rules_test.yaml`:

```sh
konveyor-analyzer-test --provider-settings provider_settings.json ./rules_test.yaml
```

The test runner will run the rules in `rule.yaml`, it will use `provider_settings.json` to set up providers.

To run more than one test files:

```sh
konveyor-analyzer-test --provider-settings provider_settings.json ./removals_rules_test.yaml ./dep_rules_test.yaml
```

To run all rules in a directory of rulesets:

```sh
konveyor-analyzer-test --provider-settings provider_settings.json ./rulesets/
```

The test runner will recursively find and run all tests in `./rulesets/`. In this case, it is assumed that all directories have a `ruleset.yaml` file defined, error otherwise.

> The `provider_settings.json` file here is only required to set up providers correctly. The location field will be mutated based on the actual sample data used and the _analysisParams_ in the test case.

#### Test Output

The output for a test should look something like:

```sh
--- rules_test.yaml    1/2 PASSED
  - rule-1000          2/2 PASSED
  - rule-1001          1/2 PASSED 
    - testCase[1]      FAIL
      - Unexpected message string in 8/10 incidents
        - Log created in ./tmp/analyzer-test-099912/analysis.log
        - Output created in ./tmp/analyzer-test-099912/output.yaml

Total: 2, 1/2 PASSED, Pass Rate: 50%
```

The output is produced per test file aggregating total number of failed / passed rules at the top.

Within a test file, it's further broken down into results of individual rules, with results of each test case for a rule.

For any failed test case, the log and the output location is printed so that users can debug. _We will also grab the provider logs and place them in the output directory so that users can debug the providers themselves._

Finally, a summary line will be printed showing overall statistics.

Additionally a YAML output file will be generated which will be later used by the CI:

```yaml
tests:
  total: 100
  passed: 99
  failed: 1
rules:
  total: 102
  tested: 100
  untested: 2
```

This output will be later used by the CI. When a new PR is opened in _konveyor/rulesets_ repo, a github action will run all _test.yaml files in the repo, and comment on the PR the above output.  The reviewer's can compare the numbers with existing numbers on main branch, and determine whether to merge the PR or ask for more coverage. 


### Technical details

#### Test Case Structure

The Go structs are defined below:

```go
type Tests struct {
    Providers []Provider `yaml:"providers,omitempty" json:"providers,omitempty"`
    Tests     []Test     `yaml:"tests,omitempty" json:"tests,omitempty"`
}

type Provider struct {
    Name     string `yaml:"name" json:"name"`
    DataPath string `yaml:"dataPath" json:"dataPath"`
}

type Test struct {
    RuleID    string     `yaml:"ruleID" json:"ruleID"`
    TestCases []TestCase `yaml:"testCases" json:"testCases"`
}

type TestCase struct {
    Description    string `yaml:"description,omitempty" json:"description,omitempty"`
    AnalysisParams `yaml:"analysisParams,omitempty" json:"analysisParams,omitempty"`
    HasIncidents   IncidentCondition `yaml:"hasIncidents,omitempty" json:"hasIncidents,omitempty"`
    HasTags        []string          `yaml:"hasTags,omitempty" json:"hasTags,omitempty"`
}

type IncidentCondition interface {
    Schema() openapi3.SchemaRef
    Verify([]konveyor.Incident) bool
}

type SimpleVerification struct {
    FileURI string `yaml:"fileURI" json:"fileURI"`
    Exactly *int   `yaml:"exactly,omitempty" json:"exactly,omitempty"`
    AtLeast *int   `yaml:"atLeast,omitempty" json:"atLeast,omitempty"`
    AtMost  *int   `yaml:"atMost,omitempty" json:"atMost,omitempty"`
}

type PerIncidentVerification struct {
    FileURI   string                 `yaml:"fileURI" json:"fileURI"`
    Locations []LocationVerification `yaml:"locations" json:"locations"`
}

type LocationVerification struct {
    LineNumber      string `yaml:"lineNumber" json:"lineNumber"`
    MessageContains string `yaml:"messageContains" json:"messageContains"`
    CodeSnipContains string `yaml:"codeSnipContains" json:"codeSnipContains"`
}

type AndCondition struct {
    Conditions []IncidentCondition `yaml:"and"`
}

type OrCondition struct {
    Conditions []IncidentCondition `yaml:"or"`
}

```

_Mode_ and _DepLabelSelector_ are nil by default. So when they are not defined in the test, provider / engine defaults will be used.

Also notice that _MessageContains_ and _CodeSnipContains_ checks in the testcase are nil by default. The only required field is _Count_. So at the minimum, the test will check for number of incidents for a rule. If any of these optional fields are defined, the test case will add those additional checks.

#### CLI Options

Here are options for the  test runner CLI:

 * --provider-settings: Required. Points to provider configuration.
 * --output-stats: Optional. Print stats to the given YAML file in addition to stdout. Defaults to stdout only.
 * --log-dir: Optional. Temporary directory for storing logs & output. Defaults to system's temp dir.
 * --verbose: Optional. Increase verbosity of output.

#### Running Tests

The runner will run each test file at once setting up the provider by merging two things - provider settings passed at runtime via `--provider-settings` and locations specified in the test file itself for each provider. Each test will run at once. This can be run concurrently in future.

Based on different analysis params used, there will be different combinations of analyses. The overlapping analyses runs can be grouped together to make things faster. It might potentially affect the results if there are rules that somehow affect each other. But maybe there's profit in catching those things early on.

The groups of tests will look something like:

```sh
.
├── data-1
│   ├── analyzer-params-1
│   │   ├── kubernetes_rules_test.yaml
│   │   └── storage_rules_test.yaml
│   └── analyzer-params-2
│       ├── kubernetes_rules_test.yaml
│       └── networking_rules_test.yaml
└── data-2
    ├── analyzer-params-1
    │   ├── kubernetes_rules_test.yaml
    │   └── storage_rules_test.yaml
    └── analyzer-params-2
        ├── kubernetes_rules_test.yaml
        └── networking_rules_test.yaml
```

The test output and logs will be generated in a temp directory per analysis run:

```sh
.
└── tmp
    ├── analyzer-test-999021
    │   ├── analysis.log
    │   ├── output.yaml
    │   ├── provider_settings.json
    │   └── rules
    │       ├── kubernetes_rules.yaml
    │       └── storage_rules.yaml
    └── analyzer-test-3321234
        ├── analysis.log
        ├── output.yaml
        ├── provider_settings.json
        └── rules
            ├── kubernetes_rules.yaml
            └── storage_rules.yaml
```

The provider_settings.json file will be created by mutating required fields per analysis.



