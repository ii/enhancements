---
title: Behavior-driven Conformance Testing
authors:
  - "@johnbelamaric"
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

### Representation of Behaviors

Behaviors must be captured in the repository and agreed upon as required for
conformance. Behaviors are broken into feature areas, and there are multiple
suites for each feature area. Some of these suites may be machine-generated
based upon the API schema, whereas others are handwritten. Keeping the
generated and handwritten suites in separate files allows regeneration of the
auto-discovered behavior suites.

Validation and conformance designations are made on a per-suite basis,
not a per-behavior basis. There may be multiple suites in a feature area
that are required for validation and/or conformance.

The grouping at the suite level [JB: or should this be called "feature"?]
should be defined based upon subjective judgement of how behaviors relate
to one another, along with an understanding that all behaviors in a given
suite may be required to function for a given cluster to pass validation
for that suite. [JB: we may also want to group based on API group and version?]

Typical suites defined for any given feature will include:
 * API spec. This suite is generated from the API schema and represents
   the basic field-by-field functionality of the feature. For features
   that include provider-specific fields (for example, various VolumeSource
   fields for pods), those must be segregated into separate suites.
 * Internal interactions. This suite tests interactions between settings
   of fields within the API schema for this feature.
 * External interactions. This suite tests interactions between this feature
   and other features.

Each suite may be stored in a separate file in a directory for the specific
area. For example, a "Pods" area would be structured as a `pods` directory with
these files:
 * `spec.yaml` describing the set of behaviors auto-generated from the API
   specification.
 * `lifecycle.yaml` describing the set of behaviors expected from the Pod
   lifecycle.

The structure of the YAML files is described by these Go types:

```
// Area defines a general grouping of behaviors
type Area struct {
        // Area is the name of the area.
        Area   string  `json:"area,omitempty"`

        // Suites is a list containing each suite of behaviors for this area.
        Suites []Suite `json:"suites,omitempty"`
}

type Suite struct {
        // Suite is the name of this suite.
        Suite       string     `json:"suite,omitempty"`

        // Level is `Conformance` or `Validation`.
        Level       string     `json:"level,omitempty"`

        // Description is a human-friendly description of this suite, possibly
        // for inclusion in the conformance reports.
        Description string     `json:"description,omitempty"`

        // Behaviors is the list of specific behaviors that are part of this
        // suite.
        Behaviors   []Behavior `json:"behaviors,omitempty"`
}

type Behavior struct {
        // Id is a unique identifier for this behavior, and will be used to tie
        // tests and their results back to this behavior. For example, a
        // behavior describing the defaulting of the PodSpec nodeSelector might
        // have an id like `pods/spec/nodeSelector/default`.
        Id          string `json:"id,omitempty"`

        // ApiObject is the object whose behavior is being described. In
        // particular, in generated behaviors, this is the object to which
        // ApiField belongs. For example, `core.v1.PodSpec` or
        // `core.v1.EnvFromSource`.
        ApiObject   string `json:"apiObject,omitempty"`

        // ApiField is filled out for generated tests that are testing the
        // behavior associated with a particular field. For example, if
        // ApiObject is `core.v1.PodSpec`, this could be `nodeSelector`.
        ApiField    string `json:"apiField,omitempty"`

        // ApiType is the data type of the field; for example, `string`.
        ApiType     string `json:"apiType,omitempty"`

        // Generated is set to `true` if this entry was generated by tooling
        // rather than hand-written.
        Generated   bool   `json:"generated,omitempty"`

        // Description specifies the behavior. For those generated from fields,
        // this will identify if the behavior in question is for defaulting,
        // setting at creation time, or updating, along with the API schema field
        // description.
        Description string `json:"description,omitempty"`
}
```

### Behavior and Test Generation Tooling

Some sets of behaviors may be tested in a similar, mechanical way. Basic CRUD
operations, including updates to specific fields and constraints on immutable
fields, operate in a similar manner across all API resources. Given this, it is
feasible to automate the creation of simple tests for these behaviors, along
with the behavior descriptions in the `spec.yaml`. In some cases a complete test
may not be easy to generate, but a skeleton may be created that can be be
converted into a valid test with minimal effort.

For these tests, the input is a set of manifests that are applied to the
cluster, along with a set of conditions that are expected to be realized within
a specified timeframe. The test framework will apply the manifests, and monitor
the cluster for the conditions to occur; if they do not occur within the
timeframe, the test will fail.

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

Additional, handwritten tests will be needed that modify the resource in
multiple ways and evaulate the behavior. The scaffolding must be built such that
the same process is used for these tests. The test author must only need to
define:
* A patch to apply to create or update the resource.
* A function to evaluate the effect of the API call.

With those two, the same creation and update scaffolding defined for individual
field updates can be reused.

### Coverage Tooling

In order to tie behaviors back to the tests that are generated, including
existing e2e tests that already cover behaviors, new tags with behavior IDs will
be added to conformance tests. Using the existing conformance framework
mechanism allows incremental adoption of this proposal. Thus, rather than a new
conformance framework function, test authors will indicate the behaviors covered
by their tests with a tag in the `framework.ConformanceIt` call.

This also enables a single test to validate multiple behaviors, although that
should be discouraged.

### Risks and Mitigations

The behavior definitions may not be properly updated if a change is made to a
feature, since these changes are made in very different areas in the code.
However, given that the behaviors defining conformance are generally stable,
this is not a high risk.

## Design Details

## Implementation History

- 2019-04-12: Created
- 2019-06-11: Updated to include behavior and test generating from APIs.

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
