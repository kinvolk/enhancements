<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# KEP-127: Support User Namespaces

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
    - [Story 3](#story-3)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
    - [Volumes](#volumes)
    - [Container Runtime Support](#container-runtime-support)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Capabilities](#capabilities)
    - [Privileged Containers](#privileged-containers)
    - [Sharing Host Namespaces](#sharing-host-namespaces)
    - [Volumes](#volumes)
- [Design Details](#design-details)
  - [Summary of the Proposed Changes](#summary-of-the-proposed-changes)
  - [CRI API Changes](#cri-api-changes)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
- [References](#references)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. KEP editors and SIG Docs
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->

Container security consists of many different kernel features that work
together to make containers secure. User namespaces is one such features that
enables interesting possibilities for containers by allowing them to be root
inside the container while not being root on the host. This gives more
capabilities to the containers while protecting the host from the container
being root and adds one more layer to container security.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

From [user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html):
> User namespaces isolate security-related identifiers and attributes, in
particular, user IDs and group IDs, the root directory, keys, and capabilities.
A process's user and group IDs can be different inside and outside a user
namespace. In particular, a process can have a normal unprivileged user ID
outside a user namespace while at the same time having a user ID of 0 inside
the namespace; in other words, the process has full privileges for operations
inside the user namespace, but is unprivileged for operations outside the
namespace.

Linux user namespaces allows to run Pods with software that expects to run as
root or with elevated privileges while still restricting the processes
permissions on the node, as the effective user ID on the node can be
unprivileged. This protects the node and other pods running in the node.

The purpose of using user namespaces in Kubernetes is to let the processes in
Pods think they run as one uid set when in fact they run as different effective
uids on the host. If a process is able to break into the host, it'll have limited
impact as it'll be running as an unprivileged user.

### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->

- Increase node to Pod isolation in Kubernetes by mapping the root user in a
Pod to an unprivileged user in the node.
- Make workloads that require root user inside the container safer for the
node using the additional security of user namespaces.

### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->

- To have different UID/GID mappings per pod/container. All pods/containers
running in a node share a common user namespace remapping configuration.
- Remote volumes support e.g. NFS.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. The "Design Details" section below is for the real
nitty-gritty.
-->

This proposal aims to support user namespaces in Kubernetes.
As done with other namespaces, each pod will be placed in its own user namespace
(shared among all containers in the Pod). The Pod specification will be extended
with a new `userHostNamespace` field to allow sharing the host user namespace.

Linux user namespaces allow to map different ranges of IDs between the container
and the host.
According to the general Linux convention, each mapping range consists of three
parts:
1. Container ID: First ID of the range in the container namespace mapped to the
   host.
2. Host ID: First ID of the range on the host mapped to the container.
3. Length: Total number of consecutive IDs mapped between host and container
   user namespaces, starting from the first one (including) mentioned above.

The implementation of user namespaces support is divided in two phases, this
proposal covers the first one: implement support for a _single_  UID/GIDs
mapping for all the pods. In other words, two pods on the same node will use the
same range of effective UID/GIDs in the node.
A second one would consider having different IDs mapping per Pod.
Even if that would be nice to provide the full support on a single phase, that
will bring a broad discussion and the risk of losing the focus and failing to
deliver this feature.
The proposal for the first phase tries to take into consideration changes needed
for the second phase in the different interfaces like the CRI to avoid further
API breaking changes in the second phase.

The mappings used in the first phase are controlled by the `uid-mapping` and
`gid-mapping` kubelet flags. The format of such flags is
`container_id0:host_id0:length0,container_id1:host_id1:length1:...`. As an
example, `--uid-mapping="0:1000:10"` maps user IDs 0 to 9 inside the container
to user IDs 1000 to 1009 on the host.
The IDs mappings are configured in the kubelet and not in the container runtime
(as done in previous proposals) because kubelet will support different IDs
mappings per Pod in the second phase. It means that the proposed CRI extension
already considers the second phase.

The user namespace feature will be controlled by the `UserNamespaceSupport`
Kubelet feature-gate, if this is `false` the user namespace will be shared with
the host, keeping the current behaviour.

### User Stories

<!--
Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

As a cluster admin, I want to protect the node from the rogue container
process(es) running inside pod containers with root privileges, so if a process is able to break out into the host, the effective user ID on the host is unprivileged.

#### Story 2

As a cluster admin, I want to support all the images irrespective of what
user/group that image is using.

#### Story 3

As a cluster admin, I want to allow some pods to disable user namespaces if they
require elevated privileges.

### Notes/Constraints/Caveats

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

We haven't considered the validations so far, `runAsUser` not in a valid range and so on.

#### Volumes

TODO(Mauricio): Drop this completely and avoid supporting volumes at all?

TODO(mauricio): Add some more introductory material here

Kubelet must ensure that Pods are able to access the files in the volumes.
Currently it's done by changing the group owner of the files to match the
`fsGroup` when it's present in the Pod specification.

This proposal extends that logic, if `fsGroup` is specified it's assumed to be
the correct GID (the group owner of the files) in the host.
The same mechanism to change the group owners of the files is used but the
`fsGroup` ID is not remapped, i.e. an 1-to-1 mapping entry  for the `fsGroup` is
added to the GID mappings in the Pod.

If `fsGroup` is not specified, the owner of the files is changed to the mapped
UID and GID in the host.
This feature is only available in some volumes plugins, the ones calling
[`SetVolumeOwnership`](https://github.com/kubernetes/kubernetes/blob/00da04ba23d755d02a78d5021996489ace96aa4d/pkg/volume/volume_linux.go#L42).

We are aware that this solution is not perfect nor optimal because changing the
owners in big volumes takes a lot of time. The same
[issue](https://github.com/kubernetes/kubernetes/pull/88488) is already present,
however it gets worst with user namespaces because the chown operation has to be
performed in more cases.
Unfortunately the kernel is missing a feature that could solve this.
The following are some ideas that could improve this support.

- [shiftfs: uid/gid shifting filesystem](https://lwn.net/Articles/757650/)
- [A new API for mounting filesystems](https://lwn.net/Articles/753473/)
- [User namespace support for NFS by Sargun](https://github.com/sargun/linux/commits/nflx-v4.9)
- [user_namespace: introduce fsid mappings](https://lwn.net/Articles/812221/)

#### Container Runtime Support

- Docker: It supports a shared IDs mapping for all the containers running
  in the host. More details on [Isolate containers with a user
  namespace](https://docs.docker.com/engine/security/userns-remap/).
  - The ID mappings are configured using the `userns-remap` in the docker
    daemon. This configuration should match the one used in kubelet.
  - There is not support for [multiple ID
    mappings](https://github.com/moby/moby/issues/28593) in Docker. How
    this could affect and shape this proposal is still a disussion to be done.
- containerd: It's quite straigtforward to implement the CRI changes proposed
  below in containerd/cri, we did in this PoC.

TODO(mauricio): Add link
- cri-o: We haven't investigated in detail yet.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

The main risk is breaking existing workloads that don't work with user namespaces.
There are some cases that are not compatible (or have known issues) with user
namespaces:

- Capabilities like `SYS_TIME`, `SYS_MODULE`, `MKNOD`, etc.
- Sharing host namespaces like NET, IPC, PID, etc.
- Privileged containers.
- Volumes without `chown` support.

[#31169](https://github.com/kubernetes/kubernetes/pull/31169) introduced the
`ExperimentalHostUserNamespaceDefaultingGate` feature gate that forces to share
the host user namespace when any of the above features is present in the Pod
specification (please notice that it is not working after the switch to CRI
[76982](https://github.com/kubernetes/kubernetes/issues/76982)).
This KEP intends to use that behaviour to avoid breaking existing workloads.
It proposes to rename the above flag (or create a new one)
`DisableHostUserNamespaceDefaulting`, the mechanism is used by default in alpha
to avoid breaking existing workloads and users can disable it to enforce Pods to
be placed in new user namespaces if `userHostNamespace: true` is not present. If
the feature flag is `true` (the mechanism is disabled), the pod will be placed
in a new user namespace without performing any validation check in that
regarding.
The motivation for this mechanism is that it will  provide time for the users to
update their templates (explicitly setting `userHostNamespace: true`) and it
will also allow the developers to investigate more in detail those limitations.

The following of this section brings more details about why those features
cannot be used with user namespaces.

#### Capabilities

The Linux kernel takes into consideration the user namespace a process is
running in while performing the capabilitis check:

> Having a capability inside a user namespace permits a process to
perform operations (that require privilege) only on resources
governed by that namespace.  In other words, having a capability in a
user namespace permits a process to perform privileged operations on
resources that are governed by (nonuser) namespaces owned by
(associated with) the user namespace (see the next subsection).

> On the other hand, there are many privileged operations that affect
resources that are not associated with any namespace type, for
example, changing the system (i.e., calendar) time (governed by
CAP_SYS_TIME), loading a kernel module (governed by CAP_SYS_MODULE),
and creating a device (governed by CAP_MKNOD).  Only a process with
privileges in the initial user namespace can perform such operations.

#### Privileged Containers

Privileged containers have all the capabilities added, for the same above reason
they cannot work together with user namespaces.

#### Sharing Host Namespaces

> When a nonuser namespace is created, it is owned by the user
namespace in which the creating process was a member at the time of
the creation of the namespace.  Privileged operations on resources
governed by the nonuser namespace require that the process has the
necessary capabilities in the user namespace that owns the nonuser
namespace.

In other words, if an application wants to perform a privileged operation in a host shared non-usernamesace it needs to have the necessary capabilities in the **host** usernamespace.
It means that pods sharing host namespaces like IPC, NET or PID but not sharing the host user namespace could not work. Infact, it's not possible to share the host network namespace without sharing the user namespace in [Docker](https://github.com/moby/moby/blob/88241b99893cce78a7734a19b38d468d0dcb6156/daemon/daemon_unix.go#L696) or [runc](https://github.com/opencontainers/runc/issues/799) at the time of writing this proposal.

#### Volumes

There are volumes that don't support changing the user of a file, i.e. the ones
not using
[`SetVolumeOwnership`](https://github.com/kubernetes/kubernetes/blob/00da04ba23d755d02a78d5021996489ace96aa4d/pkg/volume/volume_linux.go#L42),
in these cases the user namespace should be shared with the host to avoid access
problems.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->
### Summary of the Proposed Changes

- Add a `userHostNamespace` bool field to the Pod spec to allow sharing the host
  user namespace.
- Add `uid-mapping` and `gid-mapping` flags to kubelet to configure a per
  cluster ID mappings.
- Extend the CRI with a user namespace mode and the ID mappings.
- Add a `UserNamespaceSupport` feature flag to enable / disable the user
  namespaces support.
- Update owner of volumes mounted in `/var/lib/kubelet/pods/xxx/volumes`.
- Implement a defaulting to host user namespace mechanism based to the already
  existing (and broken) `ExperimentalHostUserNamespaceDefaultingGate` feature
  gate.

### CRI API Changes

The CRI has to be extended to allow kubelet to specify the user namespace mode
and the ID mappings for a Pod. The
[`NamespaceOption`](https://github.com/kubernetes/cri-api/blob/1eae59a7c4dee45a900f54ea2502edff7e57fd68/pkg/apis/runtime/v1alpha2/api.proto#L228)
is extended with two new fields:
- A `user` `NamespaceMode` that defines if the Pod should run in an independent
  user namespace (`POD` mode) or if it should share the host user namespace
  (`NODE` mode).
- The ID mappings to be used if the user namespace mode is `POD`.

```
// LinuxIDMapping represents a single user namespace mapping in Linux.
message LinuxIDMapping {
   // container_id is the starting ID for the mapping inside the container.
   uint32 container_id = 1;
   // host_id is the starting ID for the mapping on the host.
   uint32 host_id = 2;
   // number of IDs in this mapping.
   uint32 length = 3;
}

// LinuxUserNamespaceConfig represents the user and group id mappings in Linux.
message LinuxUserNamespaceConfig {
   // uid_mappings is an array of user ID mappings.
   repeated LinuxIDMapping uid_mappings = 1;
   // gid_mappings is an array of group ID mappings.
   repeated LinuxIDMapping gid_mappings = 2;
}

// NamespaceOption provides options for Linux namespaces.
message NamespaceOption {
    ...
    // User namespace for this container/sandbox.
    // Note: There is currently no way to set CONTAINER scoped user namespace in the Kubernetes API.
    // POD is the default value. Kubelet will set it to NODE when trying to use host user namespace.
    // Namespaces currently set by the kubelet: POD, NODE
    Namespace user = 5;
    // ID mappings to use when the user namespace mode is POD.
    LinuxUserNamespaceConfig mappings = 6;
}
```

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation, and anything particularly
challenging to test, should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. The KEP
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- N examples of real-world usage
- N installs
- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least two releases between beta and
GA/stable, because there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

#### Removing a Deprecated Flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md
-->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/20190731-production-readiness-review-process.md.

The production readiness review questionnaire must be completed for features in
v1.19 or later, but is non-blocking at this time. That is, approval is not
required in order to be in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.

-->

### Feature Enablement and Rollback

_This section must be completed when targeting alpha to a release._

* **How can this feature be enabled / disabled in a live cluster?**
  - [X] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: UserNamespaceSupport
    - Components depending on the feature gate: kubelet
  - [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

* **Does enabling the feature change any default behavior?**
  Yes, pods will be put into a new user namespace.

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?**
  Also set `disable-supported` to `true` or `false` in `kep.yaml`.
  Describe the consequences on existing workloads (e.g., if this is a runtime
  feature, can it break the existing applications?).

* **What happens if we reenable the feature if it was previously rolled back?**

* **Are there any tests for feature enablement/disablement?**
  The e2e framework does not currently support enabling or disabling feature
  gates. However, unit tests in each component dealing with managing data, created
  with and without the feature, are necessary. At the very least, think about
  conversion tests if API types are being modified.

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**
  Try to be as paranoid as possible - e.g., what if some components will restart
   mid-rollout?

* **What specific metrics should inform a rollback?**

* **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**
  Describe manual testing that was done and the outcomes.
  Longer term, we may want to require automated upgrade/rollback tests, but we
  are missing a bunch of machinery and tooling and can't do that now.

* **Is the rollout accompanied by any deprecations and/or removals of features, APIs,
fields of API types, flags, etc.?**
  Even if applying deprecation policies, they may still surprise some users.

### Monitoring Requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**
  Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
  checking if there are objects with field X set) may be a last resort. Avoid
  logs or events for this purpose.

* **What are the SLIs (Service Level Indicators) an operator can use to determine
the health of the service?**
  - [ ] Metrics
    - Metric name:
    - [Optional] Aggregation method:
    - Components exposing the metric:
  - [ ] Other (treat as last resort)
    - Details:

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**
  At a high level, this usually will be in the form of "high percentile of SLI
  per day <= X". It's impossible to provide comprehensive guidance, but at the very
  high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99,9% of /health requests per day finish with 200 code

* **Are there any missing metrics that would be useful to have to improve observability
of this feature?**
  Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
  implementation difficulties, etc.).

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**
  Think about both cluster-level services (e.g. metrics-server) as well
  as node-level agents (e.g. specific version of CRI). Focus on external or
  optional services that are needed. For example, if this feature depends on
  a cloud provider API, or upon an external software-defined storage or network
  control plane.

  For each of these, fill in the following—thinking about running existing user workloads
  and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:


### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field._

* **Will enabling / using this feature result in any new API calls?**
  Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
  focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)

* **Will enabling / using this feature result in introducing new API types?**
  Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)

* **Will enabling / using this feature result in any new calls to the cloud
provider?**

* **Will enabling / using this feature result in increasing size or count of
the existing API objects?**
  Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)

* **Will enabling / using this feature result in increasing time taken by any
operations covered by [existing SLIs/SLOs]?**
  Think about adding additional work or introducing new steps in between
  (e.g. need to do X to start a container), etc. Please describe the details.

* **Will enabling / using this feature result in non-negligible increase of
resource usage (CPU, RAM, disk, IO, ...) in any components?**
  Things to keep in mind include: additional in-memory state, additional
  non-trivial computations, excessive access to disks (including increased log
  volume), significant amount of data sent and/or received over network, etc.
  This through this both in small and large cases, again with respect to the
  [supported limits].

### Troubleshooting

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**

* **What are other known failure modes?**
  For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.

* **What steps should be taken if SLOs are not being met to determine the problem?**

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

The previous [design
proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/node-usernamespace-remapping.md)
proposed to configure the ID mappings in the container runtime instead of the
kubelet.
This proposal considers that it's better to configure it in the kubelet because
it'll be the one choosing these ID mappings for each Pod in the future.

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->

## References

- Support Node-Level User Namespaces Remapping design proposal document.
  - https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/node-usernamespace-remapping.md
- Node-Level UserNamespace implementation
  - https://github.com/kubernetes/kubernetes/pull/64005
- Support node-level user namespace remapping
  - https://github.com/kubernetes/enhancements/issues/127
- Default host user namespace via experimental flag
  - https://github.com/kubernetes/kubernetes/pull/31169
- Add support for experimental-userns-remap-root-uid and
  experimental-userns-remap-root-gid options to match the remapping used by the
  container runtime
  - https://github.com/kubernetes/kubernetes/pull/55707
- Track Linux User Namespaces in the Pod Security Policy
  - https://github.com/kubernetes/kubernetes/issues/59152
