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
last-updated: 2010-04-12
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
of agreed upon behaviors. Behaviors are tracked in the repository, rather than
in GitHub issues, allowing them to be reviewed and approved independently of the
tests that evaluate them.

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

This proposal is to explicitly define a machine-readable list of expected
behaviors from a conformant cluster. This top-down approach is a lot of work,
but it has some important benefits:

* The reviewers of “conforming behavior” do not need to be the same people as
  the reviewers of “this test code validates the behavior”.
* Distributors or implementors of pluggable components have a single location to
  reference for expectations.
* Coverage can be measured by tying specific tests back to the list of
  behaviors.
* Tooling can produce an explicit conformance report displaying the behaviors
  and the status of each test that checks that behavior.
* Behaviors can be grouped into sets of functionality that are conformant only
  for specific cluster profiles.
* It is a forcing function to clearly document each expected conforming
  behavior.

### Goals

* Enable separate review of behaviors and tests that evaluate those behaviors.
* Provide a single location for defining conforming behavior.
* Provide tooling to measure the conformance test coverage of the behaviors.
* Provide an incremental migration path from current conformance testing to the
  updated framework.
* Update the tooling for producing conformance reports.

### Non-Goals

* Develop a complete set of behaviors that define a conforming cluster.
* Add new conformance tests.

## Proposal

Files that define each behavior in a machine-readable format will
be created and stored in the k/k repository. These files include identifiers
for the behaviors that can be used within the e2e tests, so that the test logs
contain enough information to connect the test back to the behavior.

The structure of the YAML file is described by these Go types:

```
package conformance

type Area struct {
	Area      string     `json:"area,omitempty"`
	Api       string     `json:"api,omitempty"`
	Behaviors []Behavior `json:"behaviors,omitempty"`
}

type Behavior struct {
	Id          string `json:"id,omitempty"`
	Description string `json:"description,omitempty"`
}
```

An `Area` is an arbitrary grouping used to organize tests. The `Api` field is
optional and may specify the API group to which the tests apply, if any. Each
behavior has a unique identifier and a description. An example is shown below.

```
area: pods
api: core/v1.Pod
- category: Lifecycle
  behaviors:
  - id: pods/lifecycle/hostip
    description: After Pod creation, the Pod status MUST return successfully and contain a valid IP address.
  - id: lifecycle/submit-remove
    description: When a Pod is be created with a unique label, it can be queried using the label selector. It can be watched and will more to Running state. The Pod can be deleted, which sets the pod deletion timestamp, and the watch will receive a deletion event. Afterward, the query with the label selector must be empty.
  - id: pods/lifecycle/label-update
    description: Updating a Pod label is successful.
  - id: pods/lifecycle/active-deadline
    description: The activeDeadlineSeconds is honored when updated on the podSpec.
- category: Graceful Termination
  behaviors:
  - id: pods/graceful/podspec-period-honored
    description: The terminationGracePeriodSeconds as specified in the podSpec is honored.
  - id: pods/graceful/request-period-honored
    description: The terminationGracePeriod as specified in the DELETE request is honored.
  - id: pods/graceful/with-prestop
    description: The terminationGracePeriod is honored when a preStop hook blocks.
- category: Images
  behaviors:
  - id: pods/images/update
    description: When a podSpec container image is updated, the image is pulled and the container restarted.
  - id: pods/images/pull-policy
    description: The podSpec imagePullPolicy is honored during creation and update.
- category: OS Namespaces
  behaviors:
  - id: pods/os-namespaces/network-default
    description: By default, containers in a pod share the same network namespace.
  - id: pods/os-namespaces/network-host
    description: Containers share the host network namespace when hostNetwork is true.
  - id: pods/os-namespaces/ipc-default
    description: By default, containers in a pod share the same IPC namespace.
  - id: pods/os-namespaces/ipc-host
    description: Containers share the host IPC namespace when hostIPC is true.
  - id: pods/os-namespaces/pid-default
    description: By default, containers are in their own PID namespace.
  - id: pods/os-namespaces/pid-host
    description: Containers share the host PID namespace when hostPID is true.
- category: Volumes
  behaviors:
  - id: pods/volumes/sa-token
    description: Pods by default have a service account token mounted as a volume.
  - id: pods/volumes/sa-token-disabled
    description: No service account token volume is mounted if automountServiceAccountToken is false.
  - id: pods/volumes/shared
    description: Two containers can mount the same volume and see each others' files.
```

In order to allow incremental adoption of this framework, rather than a new
conformance framework function, test authors will indicate the behaviors covered
by their tests with a label. So, a change may look something like:

```
-       framework.ConformanceIt("should get a host IP [NodeConformance]", func() {
+       framework.ConformanceIt("should get a host IP [NodeConformance] [Behavior:pods/lifecycle/hostip]", func() {
```

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
