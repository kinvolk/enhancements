# KEP-127: Support User Namespaces

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
    - [Story 4](#story-4)
    - [Story 5](#story-5)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
    - [UID space](#uid-space)
    - [Volume support](#volume-support)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Pod.spec changes](#podspec-changes)
  - [Phases](#phases)
    - [Phase 1: pods &quot;without&quot; volumes](#phase-1-pods-without-volumes)
    - [Phase 2: pods with volumes](#phase-2-pods-with-volumes)
    - [Phase 3: pod to pod isolation](#phase-3-pod-to-pod-isolation)
  - [Summary of the Proposed Changes](#summary-of-the-proposed-changes)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [pod.spec.useHostUsers graduation](#podspecusehostusers-graduation)
      - [Alpha](#alpha)
      - [Beta](#beta)
      - [GA](#ga)
    - [pod.spec.securityContext.userns.pod2podIsolation graduation](#podspecsecuritycontextusernspod2podisolation-graduation)
      - [Alpha](#alpha-1)
      - [Beta](#beta-1)
      - [GA](#ga-1)
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
- [References](#references)
<!-- /toc -->

## Release Signoff Checklist

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

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP adds a new `hostUsers` field  to `pod.Spec` to allow to enable/disable
using user namespaces for pods. Furthermore, it allows increased pod to pod
isolation by means of `pod.spec.securityContext.userns.pod2podIsolation` field.

It allows users to place pods in different user namespaces increasing the
pod-to-pod and pod-to-host isolation. This extra isolation increases the cluster
security as it protects the host and other pods from malicious or compromised
processes inside containers that are able to break into the host.

## Motivation

From
[user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html):
> User namespaces isolate security-related identifiers and attributes, in
particular, user IDs and group IDs, the root directory, keys, and capabilities.
A process's user and group IDs can be different inside and outside a user
namespace. In particular, a process can have a normal unprivileged user ID
outside a user namespace while at the same time having a user ID of 0 inside the
namespace; in other words, the process has full privileges for operations inside
the user namespace, but is unprivileged for operations outside the namespace.

The goal of supporting user namespaces in Kubernetes is to be able to run
processes in pods with a different user and group IDs than in the host.
Specifically, a privileged process in the pod runs as an unprivileged process in
the host. If such a process is able to break out of the container to the host,
it'll have limited impact as it'll be running as an unprivileged user there.

The following security vulnerabilities are (completely or partially) mitigated
with user namespaces as we propose here and it is expected that it will mitigate
similar future vulnerabilities too.
- [CVE-2019-5736](https://nvd.nist.gov/vuln/detail/CVE-2019-5736): Host runc binary can be overwritten from container. Completely mitigated with userns.
  - Score: [8.6 (HIGH)](https://nvd.nist.gov/vuln/detail/CVE-2019-5736)
  - https://github.com/opencontainers/runc/commit/0a8e4117e7f715d5fbeef398405813ce8e88558b
- [Azurescape](https://unit42.paloaltonetworks.com/azure-container-instances/):
  Completely mitigated with userns, at least as it was found (needs CVE-2019-5736). This is the **first cross-account container takeover in the public cloud**.
- [CVE-2021-25741](https://github.com/kubernetes/kubernetes/issues/104980): Mitigated as root in the container is not root in the host
  - Score: [8.1 (HIGH) / 8.8 (HIGH)](https://nvd.nist.gov/vuln/detail/CVE-2021-25741)
- [CVE-2017-1002101](https://github.com/kubernetes/kubernetes/issues/60813): mitigated, idem
  - Score: [9.6 (CRITICAL) / 8.8 (HIGH)](https://nvd.nist.gov/vuln/detail/CVE-2017-1002101)
- [CVE-2021-30465](https://github.com/opencontainers/runc/security/advisories/GHSA-c3xm-pvg7-gh7r): mitigated, idem
  - Score: [8.5 (HIGH)](https://nvd.nist.gov/vuln/detail/CVE-2021-30465)
- [CVE-2016-8867](https://nvd.nist.gov/vuln/detail/CVE-2016-8867): Privilege escalation inside containers
  - Score: [7.5 (HIGH)](https://nvd.nist.gov/vuln/detail/CVE-2016-8867)
  - https://github.com/moby/moby/issues/27590
- [CVE-2018-15664](https://nvd.nist.gov/vuln/detail/CVE-2018-15664): TOCTOU race attack that allows to read/write files in the host
  - Score: [7.5 (HIGH)](https://nvd.nist.gov/vuln/detail/CVE-2018-15664)
  - https://github.com/moby/moby/pull/39252

### Goals

Here we use UIDs, but the same applies for GIDs.

- Increase node to pod isolation by mapping user and group IDs
  inside the container to different IDs in the host. In particular, mapping root
  inside the container to unprivileged user and group IDs in the node.
- Increase pod to pod isolation by allowing to use non-overlapping mappings
  (UIDs/GIDs) whenever possible. IOW, if two containers runs as user X, they run
  as different UIDs in the node and therefore are more isolated than today.
  See phase 3 for limitations.
- Allow pods to have capabilities (e.g. `CAP_SYS_ADMIN`) that are only valid in
  the pod (not valid in the host).
- Benefit from the security hardening that user namespaces provide against some
  of the future unknown runtime and kernel vulnerabilities.

### Non-Goals

- Provide a way to run the kubelet process or container runtimes as an
  unprivileged process. Although initiatives like [kubelet in user
  namespaces][kubelet-userns] and this KEP both make use of user namespaces, it is
  a different implementation for a different purpose.
- Implement all the very nice use cases that user namespaces allows. The goal
  here is to allow them as incremental improvements, not implement all the
  possible ideas related with user namespaces.

[kubelet-userns]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2033-kubelet-in-userns-aka-rootless

## Proposal

This proposal aims to support running pods inside user namespaces. This will
improve the pod to node isolation (phase 1 and 2) and pod to pod isolation
(phase 3) we currently have.

This mitigates all the vulnerabilities listed in the motivation section.

### User Stories

#### Story 1

As a cluster admin, I want run some pods with privileged capabilities because
the applications in the pods require it (e.g. `CAP_SYS_ADMIN` to mount a FUSE
filesystem or `CAP_NET_ADMIN` to setup a VPN) but I don't want this extra
capability to grant me any extra privileges on the host or other pods.

#### Story 2

As a cluster admin, I want to allow some pods to run in the host user namespace
if they need a feature only available in such user namespace, such as loading a
kernel module with `CAP_SYS_MODULE`.

#### Story 3

As a cluster admin, I want to allow users to run their container as root
without that process having root privileged on the host, so I can mitigate the
impact of a compromised container.

#### Story 4

As a cluster admin, I want to allow users to choose the user/group they use
inside the container without it being the user/group they use on the node, so I
can mitigate the impact of a container breakout vulnerability (e.g to read
host files).

#### Story 5

As a cluster admin, I want to use different host UIDs/GIDs for pods running in
the same node (whenever kernel/kube features allow it), so I can mitigate the
impact a compromised pod can have on other pods and the node itself.

### Notes/Constraints/Caveats

#### UID space

#### Volume support

### Risks and Mitigations

## Design Details

Note: Names are preliminary yet, I'm using field names to simplify explanations.

### Pod.spec changes

The following changes will be done to the pod.spec:

- `pod.spec.useHostUsers`: bool.
If true or not present, uses the host user namespace (as today)
If false, a new userns is created for the pod.
This field will be used for phase 1, 2 and 3.

- `pod.spec.securityContext.userns.pod2podIsolation`: Enum
If enabled, we will make the userns mappings be non-overlapping as much as possible.
This field will be used in phase 3.

### Phases

We propose to divide the work in 3 phases. Each phase makes this work with
either more isolation or more workloads. When no support is yet added to handle
some workload, a clear error will be shown.

Please note the last sub-section here is a table with the summary of the changes
proposed on each phase.

#### Phase 1: pods "without" volumes

This phase makes pods "without" volume work with user namespaces. This is
activated via the bool `pod.spec.HostUsers` and works only when no volume or any
combination of the following volume types are used by the pod:

 - configmap
 - secret
 - downwardAPI
 - emptyDir
 - projected

This list of volumes was chosen as they can't be used to share files with other
pods.

The mapping length will be 65535, mapping the range 0-65534 to the pod. This wide
range makes sure most workloads will work fine.

The mapping will be chosen by the kubelet, using a simple algorithm to give
different pods in this category ("without" volumes) a non-overlapping mapping.
Giving non-overlapping mappings generates the best isolation for pods.

Furthermore, the node UID space of 2^32 can hold up to 2^16 pods each with a
mapping length of 65535 (2^16-1) top. This imposes a limit of 65k pods per node,
but that is not an issue we will hit in practice for a long time, if ever (today
we run 110 pods per node by default).

#### Phase 2: pods with volumes

This phase makes user namespaces work in pods with volumes too. This is
activated via the bool `pod.spec.HostUsers` too and pods fall into this mode if
some other volume type than the ones listed for phase 1 is used. IOW, when phase
2 is implemented, pods that use volume types not supported in phase 1, fall into
the phase 2.

All pods in this mode will use _the same_ mapping, chosen by the kubelet, with a
length 65535, and mapping the range 0-65534 too. IOW, each pod will have its own user
namespace, but they will map to _the same_ UIDs/GIDs in the host.

Using the same mapping allows for pods to share files and mitigates all the
listed vulnerabilities (as the host is protected from the container). It is also
a trivial next-step to take, given that we have phase 1 implemented: just return
the same mapping if the pod has other volumes.

#### Phase 3: pod to pod isolation

This phase will provide more isolation between pods that use volumes (as in
phase 2) and requires another opt-in field:
`pod.spec.securityContext.pod2podIsolation`.

This phase will try to not share the same mapping for all pods with volumes, as
phase 2 does, but to achieve it some trade off needs to be made. This phase
builds on the work of the previous phases and more details will be defined while
the other phases evolve.

Here are some ideas so far:

One idea is to give different mappings to pods in different k8s namespaces or
that use a different service account. This needs to be explored in further
detail, but will probably impose limits to which workloads can run this (we need
to expose a shorter mapping, less than 65535).

Another idea is to use id mapped mounts. This probably needs changes to the
OCI runtime-spec, only works with certain filesystems and kernels that may take
too long for some users to get (like managed services). Giuseppe started to
experiment in crun with this
[here](https://github.com/containers/crun/pull/780).

The value for `pod.spec.securityContext.pod2podIsolation` will be an enum, to
select different strategies and allow room for future improvements.

It is being considered having a value that is "auto" for this fields, that
will select the best strategy that your node supports. However, as different
strategies will change the effective UID a container uses, if we add such an
option the documentation will be VERY clear about the implications and
automatizations will be provided whenever possible (we have some ideas on this
front).

Another improvement suggested by @ddebroy to do here is:
 * Pods using also only [local ephemeral CSI volumes][csi-ephemeral-vol], as
   they share the same lifecycle of the pod, can be moved to use non-overlapping
   mappings.

This change can probably be done under the hood without the user noticing, to
achieve more pod 2 pod isolation, and might not need the user to use
`pod.spec.securityContext.pod2podIsolation`. However, some changes for the CSI
vol to use the effective UID/GID might be needed and not trivial. @ddebroy has
[kindly offered to help][csi-help] with this improvement

[csi-ephemeral-vol]: https://kubernetes-csi.github.io/docs/ephemeral-local-volumes.html#overview
[csi-help]: https://github.com/kubernetes/enhancements/pull/3065/files#r762046107

If this phase turns out to be a lot of work, it will be left out as future work
for other KEPs.

### Summary of the Proposed Changes

[This table](https://docs.google.com/presentation/d/1z4oiZ7v4DjWpZQI2kbFbI8Q6botFaA07KJYaKA-vZpg/edit#slide=id.gfd10976c8b_1_41) gives you a quick overview of each phase.


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

TBD

### Graduation Criteria

Graduation for each pod.spec field we introduce will be separate.

#### pod.spec.useHostUsers graduation

##### Alpha

##### Beta

##### GA

#### pod.spec.securityContext.userns.pod2podIsolation graduation

##### Alpha

##### Beta

##### GA

### Upgrade / Downgrade Strategy

### Version Skew Strategy

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [ ] Feature gate
    - Feature gate name:
    - Components depending on the feature gate:

* **Does enabling the feature change any default behavior?**

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?**

* **What happens if we reenable the feature if it was previously rolled back?**

* **Are there any tests for feature enablement/disablement?**

### Rollout, Upgrade and Rollback Planning

Will be added before transition to beta.

* **How can a rollout fail? Can it impact already running workloads?**

* **What specific metrics should inform a rollback?**

* **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**

* **Is the rollout accompanied by any deprecations and/or removals of features, APIs,
fields of API types, flags, etc.?**


### Monitoring Requirements

Will be added before transition to beta.

* **How can an operator determine if the feature is in use by workloads?**

* **What are the SLIs (Service Level Indicators) an operator can use to determine
the health of the service?**

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**

* **Are there any missing metrics that would be useful to have to improve observability
of this feature?**

### Dependencies

* **Does this feature depend on any specific services running in the cluster?**: No.

### Scalability

* **Will enabling / using this feature result in any new API calls?** No.

* **Will enabling / using this feature result in introducing new API types?** No.

* **Will enabling / using this feature result in any new calls to the cloud
provider?** No.

* **Will enabling / using this feature result in increasing size or count of
the existing API objects?** Yes. The PodSpec will be increased.

* **Will enabling / using this feature result in increasing time taken by any
operations covered by [existing SLIs/SLOs]?**

* **Will enabling / using this feature result in non-negligible increase of
resource usage (CPU, RAM, disk, IO, ...) in any components?**: No.

### Troubleshooting

Will be added before transition to beta.

* **How does this feature react if the API server and/or etcd is unavailable?**

* **What are other known failure modes?**

* **What steps should be taken if SLOs are not being met to determine the problem?**

## Implementation History

## Drawbacks

## Alternatives

## References
