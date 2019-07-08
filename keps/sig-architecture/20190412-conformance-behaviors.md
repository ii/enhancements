---
title: Behavior-driven Conformance Testing
authors:
  - "@johnbelamaric"
  - "@hh"
owning-sig: sig-architecture
participating-sigs:
  - sig-testing
reviewers:
  - TBD
approvers:
  - "@bgrant0607"
  - "@smarterclayton"
editor: TBD
creation-date: 2019-04-12
last-updated: 2010-06-11
status: provisional
---

# Behavior-driven Conformance Testing

## Table of Contents

- [Title](#title)
  - [Table of Contents](#table-of-contents)
  - [Release Signoff Checklist](#release-signoff-checklist)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Implementation History](#implementation-history)
  - [Drawbacks](#drawbacks)
  - [Alternatives](#alternatives)

## Release Signoff Checklist

**ACTION REQUIRED:** In order to merge code into a release, there must be an issue in [kubernetes/enhancements] referencing this KEP and targeting a release milestone **before [Enhancement Freeze](https://github.com/kubernetes/sig-release/tree/master/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core Kubernetes i.e., [kubernetes/kubernetes], we require the following Release Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These checklist items _must_ be updated for the enhancement to be released.

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

**Note:** Any PRs to move a KEP to `implementable` or significant changes once it is marked `implementable` should be approved by each of the KEP approvers. If any of those approvers is no longer appropriate than changes to that list should be approved by the remaining approvers and/or the owning SIG (or SIG-arch for cross cutting KEPs).

**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://github.com/kubernetes/enhancements/issues
[kubernetes/kubernetes]: https://github.com/kubernetes/kubernetes
[kubernetes/website]: https://github.com/kubernetes/website

## Summary

This proposal modifies the conformance testing framework to be driven by a list
of agreed upon behaviors. These behaviors are identified by processing of the
API schemas, documentation, expert knowledge, and code examination. They are
explicitly documented and tracked in the repository rather than in GitHub
issues, allowing them to be reviewed and approved independently of the tests
that evaluate them. Additionally it proposes new tooling to generate tests, test
scaffolding, and test coverage reports.

## Motivation

It has proven difficult to measure how much of the expected Kubernetes behavior
the current conformance tests cover. The current measurements are based upon
identifying which API endpoints are exercised by the tests. The Kubernetes API
is CRUD-oriented, and most of the client’s desired behavior is encapsulated in
the payload of the create or update calls, not in the simple fact that those
endpoints were hit. This means that even if a given endpoint is shown as
covered, it’s impossible to know how much that tests the actual behavior.

Coverage is measured this way because there is no single, explicit list of
behaviors that comprise the expected behavior of a conformant cluster. These
expectations are spread out across the existing design documents, KEPs, the user
documentation, a subset of the e2e test, and the code itself. This makes it
impossible to identify if the conformance suite provides a meaningful test of a
cluster’s operation.

Additionally, progress in writing and promoting tests has been slow and too much
manual effort is involved. As a starting point, this proposal includes new
tooling that uses the API schemas to identify expected behaviors and produce
tests and test scaffolding to quickly cover those behaviors.

### Goals

* Enable separate review of behaviors and tests that evaluate those behaviors.
* Provide a single location for defining conforming behavior.
* Provide tooling to generate as many of the behaviors as possible from API
  schemas.
* Provide tooling to generate tests and test scaffolding for validating
  behaviors.
* Provide tooling to measure the conformance test coverage of the behaviors.
* Provide an incremental migration path from current conformance testing to the
  updated framework.

### Non-Goals

* Develop a complete set of behaviors that define a conforming cluster. This is
  an ongoing effort, and this proposal is intended to make that more efficient.
* Add new conformance tests. It is expected that during this effort new
  tests may be created using the proposed tooling, but it is not explicitly part
  of this proposal.

## Proposal

The proposal consists of four deliverables:
* A machine readable format to define conforming behaviors.
* Tooling to generate lists of behaviors from the API schemas.
* Tooling to generate tests and test scaffoloding to evaulate those behaviors.
* Tooling to compare the implemented tests to the list of behaviors and
  calculate coverage.

### Representation of Behaviors as Feature files with Scenarios

Behaviors must be captured in the repository and agreed upon as required for
conformance. Behaviors are broken into areas as folders, and there are
multiple feature files for each area. Some of these features may be
machine-generated based upon the API schema, whereas others are handwritten.
Keeping the generated and handwritten suites in separate files allows
regeneration of the auto-discovered behavior suites.

Validation and conformance designations are made on a per-scenario basis via
tagging, not a per-feature file basis. There may be multiple scenarios in a
feature that are required for validation and/or conformance.

The grouping at the feature level should be defined based upon subjective
judgement of how behavior scenarios relate to one another, along with an
understanding that all behavior scenarios in a given feature may be required to
function for a given cluster to pass validation for that feature.

Typical features defined for any given area will include:
 * API spec. This suite is generated from the API schema and represents
   the basic field-by-field functionality of the feature. For features
   that include provider-specific fields (for example, various VolumeSource
   fields for pods), those must be segregated into separate features.
 * Internal interactions. This suite tests interactions between settings
   of fields within the API schema for this feature.
 * External interactions. This suite tests interactions between this feature
   and other features.

#### Complex Storytelling combined with json/yaml

Inline json or yaml as CRUD input/output can be autogenerated for verification. The
json or yaml can also be contained in external files. The functions matching the
step definitions would be re-used for all matching scenarios as needed.

```feature
Feature: Intrapod Communication
  Pods need to be able to talk to each other, as well as the node talking to the Pod.
  @sig-node @sig-pod
  Scenario: Nodes can communicate to each other
    Given a pods A and B
    When pod A says hello to pod B
    Then pod B says hello to pod A
  @wip @tags-are-no-longer-part-of-test-names
  Scenario: Pods can can communicate to Nodes
    Given a pod A on a node 
    When the node says hello to pod A
    Then pod A says hello to the node
    And this is fine
```

Each feature may be stored in a separate file in a directory for the specific
area. For example, a "Pods" area would be structured as a `pods` directory with
these files:
 * `api-generated.feature` describing the set of behaviors auto-generated from the API
   specification.
 * `lifecycle.feature` describing the set of behaviors expected from the Pod
   lifecycle.

### Behavior and Test Generation Tooling

Some sets of behaviors may be tested in a similar, mechanical way. Basic CRUD
operations, including updates to specific fields and constraints on immutable
fields, operate in a similar manner across all API resources. Given this, it is
feasible to automate the creation of simple tests for these behaviors, along
with the behavior descriptions in the `spec.feature`. In some cases a complete test
may not be easy to generate, but a skeleton may be created that can be be
converted into a valid test with minimal effort.

For these tests, the input is a set of manifests that are applied to the
cluster, along with a set of conditions that are expected to be realized within
a specified timeframe. The test framework will apply the manifests, and monitor
the cluster for the conditions to occur; if they do not occur within the
timeframe, the test will fail.

#### Autogenerated Behaviour Scenarios

These could be supplied as files, or inline as demonstrated in the handwritten
behaviour section.

```feature
Feature: Autogenerated for v1.Ingress
  As the autogenerated feature testing for v1.Ingress
  I want to maintain a list of scenarios iterating over my parameters
  In order to ensure CRUD style updates work well
  @auto-generated
  Scenario: v1.Ingress Create
    Given I kubectl apply ingress.create.yaml
    When I kubectl apply ingress.update.yaml
    Then I observe the changes in ingress.observation.yaml
    And this is fine
  Scenario: v1.Ingress Delete
    Given I kubectl apply ingress.create.yaml
    When I kubectl apply ingress.delete.yaml
    Then I observe the deletion within 2 minutes
    And this is fine
  Scenario: v1.Ingress Field X Mutation 
    Given I kubectl apply ingress.create.yaml
    When I kubectl apply ingress.mutate-x.yaml
    Then I observe ingress-verify-x.yaml within 5 minutes
    And this is fine
```

For each Spec object, scaffolding can be defined to include the following tests:

* Creation and read of the resource with only required fields specified.
  * API functions as expected: Resource is created and may be read, and defaults
    are set. This is mechanical and can be completely generated.
  * Cluster behaves as expected. This cannot be generated, but a skeleton can be
    generated that allows test authors to evaluate the condition of the cluster
    to make sure it meets the expectations.
* Deletion of the resource. This may be mostly mechanical but if there are side-
  effects, such as garbage collection of related resources, we may want to have
  manually written evaluation here as well.
* Creation of resource with each field set, and update of each mutable field.
  * For each mutable field, apply a patch to the based resource definition
    before creation (for create tests), or after creation (for update tests).
  * Evaluate that the API functions as expected; this is mechanical and
    generated.
  * Evaluate that the cluster behaves as expected. In some cases this may be
    able to re-use the same evaluation function used during the creation tests,
    but often it will require hand-crafted code to test the conditions.
    Nonetheless, the scaffolding can be generated, minimizing the effort needed
    to implement the test.


#### Handwritten Behaviour Scenarios

Additional, handwritten tests will be needed that modify the resource in
multiple ways and evaulate the behavior. The scaffolding must be built such that
the same process is used for these tests. The test author must only need to
define:
* A patch to apply to create or update the resource.
* A function to evaluate the effect of the API call.

```feature
Feature: Manually using Manifests to CRUD and evaluate effects
  Pods need to be able to talk to each other, as well as the node talking to the Pod.
  Scenario: Pods can can communicate to Nodes
    Given I create pod A with this yaml spec
      """
      yaml: [
         values
      ]
      """
    And I create pod B with this json spec
      """
      {
        json: values
      }
      """
    When I request pod A and pod B talk to each other
    Then I can observe a v1.PodCommunication matching this json spec
      """
      {
        "node a": "talked to node b" 
      }
      """
    And this is fine
```

With those two, the same creation and update scaffolding defined for individual
field updates can be reused.

#### Generating scaffolding from Gherkin .feature files 

A Gherkin **feature** is synonymous with our definition of **behaviour**, and
tagging can be used for **@conformance** or **@release-X.Y** metadata.

```feature
Feature: Structured Metadata allowing Behaviour Driven tooling automation
  In order to auto-generate testing scaffolding
  As a sig-X member
  I want to decribe the behaviour of X

  @sig-X
  Scenario: Behaviour X
    Given a well formed file describing the behaviour X
    When I run the automation
    Then I am provided with the basic structure for a corresponding test
    And this is fine
  @sig-Y
  Scenario: Behaviour Y
    Given a well formed file describing the behaviour Y
    When I run the automation
    Then I am provided with the basic structure for a corresponding test
    And this is fine
  @sig-Y @sig-X
  Scenario: Behaviour X+Y
    Given a well formed file describing the behaviour X
    And a well formed file describing the behaviour Y
    When I run the automation
    Then I can reuse existing step definitons on multiple tests
    And this is fine
```

#### Autogeneration of Test Skaffolding

```shell
~/go/bin/godog --no-colors
```

```feature
Feature: Structured Metadata allowing Behaviour Driven tooling automation
  In order to auto-generate testing scaffolding
  As a sig-X member
  I want to decribe the behaviour of X

  Scenario: Behaviour X                                                  # features/behaviour.feature:7
    Given a well formed file describing the behaviour X
    When I run the automation
    Then I am provided with the basic structure for a corresponding test
    And this is fine

  Scenario: Behaviour Y                                                  # features/behaviour.feature:13
    Given a well formed file describing the behaviour Y
    When I run the automation
    Then I am provided with the basic structure for a corresponding test
    And this is fine

  Scenario: Behaviour X+Y                                       # features/behaviour.feature:19
    Given a well formed file describing the behaviour X
    And a well formed file describing the behaviour Y
    When I run the automation
    Then I can reuse existing step definitons on multiple tests
    And this is fine

3 scenarios (3 undefined)
13 steps (13 undefined)
1.253405ms

You can implement step definitions for undefined steps with these snippets:

func aWellFormedFileDescribingTheBehaviourX() error {
  return godog.ErrPending
}

func iRunTheAutomation() error {
  return godog.ErrPending
}

func iAmProvidedWithTheBasicStructureForACorrespondingTest() error {
  return godog.ErrPending
}

func thisIsFine() error {
  return godog.ErrPending
}

func aWellFormedFileDescribingTheBehaviourY() error {
  return godog.ErrPending
}

func iCanReuseExistingStepDefinitonsOnMultipleTests() error {
  return godog.ErrPending
}

func FeatureContext(s *godog.Suite) {
  s.Step(`^a well formed file describing the behaviour X$`, aWellFormedFileDescribingTheBehaviourX)
  s.Step(`^I run the automation$`, iRunTheAutomation)
  s.Step(`^I am provided with the basic structure for a corresponding test$`, iAmProvidedWithTheBasicStructureForACorrespondingTest)
  s.Step(`^this is fine$`, thisIsFine)
  s.Step(`^a well formed file describing the behaviour Y$`, aWellFormedFileDescribingTheBehaviourY)
  s.Step(`^I can reuse existing step definitons on multiple tests$`, iCanReuseExistingStepDefinitonsOnMultipleTests)
}

```

These functions and the Suite.Step matchers that tie them to Gherkin steps can
be pasted into a test_steps.go file as a initial skaffolding.

### Coverage Tooling

Our current tests are not super easy to write, read, or review. BDD in go was in
it's early days when k8s started integration testing with a closely coupled
component testing approach. Our Ginko based e2e framework evolved based upon
those tightly coupled assumptions. This approach unfortuneatly lacks the
metadata, tags, and descriptions of the desired behaviours required for clear
separation of acceptance behaviors and tests.

Documenting and discovering of all our behaviours will require a combination of
automated introspection and well as some old fashioned human storytelling.

To do so need to standardize the business language that our bottlenecked people
can use to write these stories in a way can be assisted with some automation.
This would reduce complexity for articulating concrete requirements for
execution in editors, humans, and automation workflows.

Defining our Behaviours in Gherkin would allow us to leverage our existing
conformance framework and test mechanisms to allow incremental adoption of this
proposal.

Scenarios could be defined for existing tests using the form:

```feature
Scenario: Use existing ginkgo framework
  As a test contributor
  I want to not throw away all our old tests
  In order to retain the value generated in them
  @sig-node @sig-pod @conformance @release-1.15
  Feature: Map behaviours to existing ginkgo tests
    Given existing test It('should do the right thing')
    And I optionally tag it with @conformance
    When I run the test
    Then we utilize our existing test via our new .feature framework
    And this is fine
```

Thus, test authors will indicate the behaviors covered by adding a
**@conformance** tag to Feature/Behaviours using `Given an existing test
It('test string')`

### Risks and Mitigations

The behavior definitions may not be properly updated if a change is made to a
feature, since these changes are made in very different areas in the code.
However, given that the behaviors defining conformance are generally stable,
this is not a high risk.

## Design Details


## Implementation History

- 2019-04-12: Created
- 2019-06-11: Updated to include behavior and test generating from APIs.
- 2019-07-08: Updated to include Gherkin / godog as possible behaviour workflow

## Drawbacks

* Separating behaviors into a file that is not directly part of the test suite
  creates an additional step for developers and could lead to divergence.

## Alternatives

### Annotate test files with behaviors

This option is essentially an extension of the existing tagging of e2e tests.
Rather than just tagging existing tests, we can embed the list of behaviors in
the files as well. The same set of metadata that is described in Option 1 can be
embedded as specialized directives in comments.


*Pros*
* Keeps behaviors and tests together in the same file.

*Cons*
* All of the same features may be met, but the tooling needs to parse the Go
  code and comments, which is more difficult than parsing a YAML.
* Behaviors are scattered throughout test files and intermingled with test code,
  making it hard to review whether the list of behaviors is complete (this
  could be mitigated with tooling similar to the existing tooling that extracts
  test names).
* Adding or modifying desired behaviors requires modifying the test files, and
  leaving the behaviors with a TODO or similar flag for tracking what tests are
  needed.

### Annotate existing API documentation with behaviors
The current API reference contains information about the meaning and expected
behavior of each API field. Rather than producing a separate list, the metadata
for conformance tests can be attached to that documentation.

*Pros*
* Avoids adding a new set of files that describe the behavior, leveraging what
  we already have.
* API reference docs are a well-known and natural place to look for how the
  product should behave.
* It is clear if a given API or field is covered, since it is annotated directly
  with the API.

*Cons*
* Behaviors are spread throughout the documentation rather than centrally
  located.
* It may be difficult to add tests that do not correspond to specific API
  fields.
