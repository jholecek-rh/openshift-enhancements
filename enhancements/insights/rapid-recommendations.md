---
title: rapid-recommendations
authors:
  - "@jholecek"
  - "@tremes"
reviewers: 
  - "@deads2k"
approvers: # A single approver is preferred, the role of the approver is to raise important questions, help ensure the enhancement receives reviews from all applicable areas/SMEs, and determine when consensus is achieved such that the EP can move forward to implementation.  Having multiple approvers makes it difficult to determine who is responsible for the actual approval.
  - TBD
api-approvers: 
  - None
creation-date: 2024-01-03
last-updated: 2024-01-08
tracking-link: # link to the tracking ticket (for example: Jira Feature or Epic ticket) that corresponds to this enhancement
  - TBD
see-also:
  - "/enhancements/insights/conditional-data-gathering.md"
replaces:
  - None
superseded-by:
  - "/enhancements/our-past-effort.md"
---

# Insights Rapid Recommendations

This enhancement proposal introduces a new approach on how the data 
collected by the Insights Operator can be externally defined.

## Summary

The Insights Operator collects various data and resources from the OpenShift and Kubernetes APIs. The definition of the collected data is mostly hardcoded in the operator's source code and largely locked in the corresponding OCP version.
This proposal introduces a different approach of external data definition that will help us to avoid the tedious backport procedure when we need to collect new data in older OCP versions. As a side effect, this will allow us to collect some "temporary" data from different versions of OCP.   

## Motivation

The main motivation is to reduce the effort and time to develop a new Insights Advisor recommendation as well as reduce the time to impact existing clusters. Currently, when a new data request is made to the Insights Operator, a new gatherer must be developed and then (optionally) backported to all supported z-stream versions of OCP. This limits the impact of the new Insights Advisor recommendation based on this data. 
The new approach, introduced in the proposal, should also help us to address some new recommendations - e.g bug related. 

### User Stories

* As an Insights recommendation developer/engineer I request a new data to be collected/gathered by the Insights Operator.
* As an Insights recommendation developer/engineer I request a new data from OCP version X.Y.Z so that I can prepare a new Insights recommendation related to some bug present in the X.Y.Z OCP version. 
* As an analytics person or product manager I request a new one-off data about my product deployed in the OpenShift. 

### Goals

The main goal is to reduce the current effort and time needed to develop a new Insights Advisor recommendation. The key factor (which currently blocks any optimization of this effort) is that the gathered Insights data is defined in the source code of the Insights Operator. The goal of this enhancement is to change that and allow external (to the cluster) definition of the data that is gathered. Another perspective is to avoid tedious and risky backport procedure that cannot address the existing OCP versions anyway. 

### Non-Goals

* Fundamental change in the nature of the collected data - we still have some data that we require in every Insights archive (e.g clusteroperator conditions)
* No remote code execution in OpenShift cluster
* Change frequency of data gathering
  
## Proposal

The plan is to create and deploy a new service running in console.redhat.com environment that will serve the definition of the data gathered by the Insights Operator. The existing data gathered by the Insights Operator can be divided into three groups based on their format:

  * JSON/YAML - we should be able to define most of this data by providing the required group, version, kind (GVK)
  * container logs - this data can be identified by the namespace name, Pod name and also the messages to be filtered from the log 
  * Prometheus metrics - this can be identified by the metric name (and perhaps some labels?), but this currently one of the unknowns

All these requirements and identifiers will be exposed by the service and the Insights Operator will periodically (2 hours by default) query the service (TBD caching??), read the configuration and gather the data defined in the external configuration.

Similar or substantially the same behaviour or connection already exists - you can see [Conditional data gathering enhancement](../insights/conditional-data-gathering.md)

But, of course, this approach raises a number of new questions, which are discussed in the [Open questions](#open-questions-optional) and [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints-optional) sections.

### Workflow Description

The main interaction will be on the side of the person requesting the new data for Insights purposes. Let's call this person as "data requester" for the purpose of this enhancement. The main workflow will be following:

1. Data requester must know the data and corresponding formatting required by the external configuration service.
2. Data requester creates a pull request to the repository containing the configuration or source files for the external configuration service. 
3. The pull request is reviewed (and either approved or denied) by a member of the Observability inteligence (CCX) team.
4. When the pull request is merged, a new release of the external configuration version is required. 
5. New version of the external configuration is released and the Insights Operator needs to be able to retrieve it and use it (see more in the [Versioning of the external configuration](#versioning-of-the-external-configuration) section)

#### Variation and form factor considerations [optional]

Do we want to keep this section???? Maybe only for the HyperShift ??

How does this proposal intersect with Standalone OCP, Microshift and Hypershift?

If the cluster creator uses a standing desk, in step 1 above they can
stand instead of sitting down.

See
https://github.com/openshift/enhancements/blob/master/enhancements/workload-partitioning/management-workload-partitioning.md#high-level-end-to-end-workflow
and https://github.com/openshift/enhancements/blob/master/enhancements/agent-installer/automated-workflow-for-agent-based-installer.md for more detailed examples.

### API Extensions

This proposal does not add any API extension.

### Implementation Details/Notes/Constraints 

The following subsections discuss some important implementation topics.

#### Validation of the external configuration

The external configuration must be validated. The validation will happen on the Insights Operator side and the idea is to have JSON schema defining the validation rules. The JSON schema is likely to be part of the Insights Operator codebase, which means that the schema can ideally only change between y-stream or x-stream OCP versions. The scenario of failing validation is described in the [Risk mitigation](#risks-and-mitigations) section.

#### Reading and using the external configuration (including data obfuscation)

The external configuration will be unmarshalled and each part will require different approach based on the resulting format of the requested data:

* Data defined with their group, version, kind can be processed using the Kubernetes dynamic client. This client only provides untyped data in the form of `map[string]interface{}`, which of course brings some limitations. One of the limitations is the data obfuscation. We usually know the JSON path of resource attributes we want to obfuscate, but it's probably easier to work with typed data than untyped data. The Kubernetes `unstructured` package offers good ways to work with the JSON paths and unstructured objects, so if we want to mimic the original Insights Operator obfuscation, we can either remove the attributes from the resource or extract them and replace them with `X` characters. This probably means keeping a list of these paths and checking and iterating over them for each resource. The opt-in obfuscation of IP addresses and cluster domain name remains unchanged.
* Log data will require function to retrieve and read the logs of the particular containers. We already have several such functions in the Insights Operator codebase, but it would be good to unify this approach and use one common function to retrieve and parse or filter container logs.  
* Prometheus metrics - ??? how to limit the metrics cardinality ??? This can mean data explosion in the archive

#### Versioning of the external configuration

By default, the Insights operator knows only the cluster version so the external configuration can provide the content based on the provided X.Y.Z OCP version. If the corresponding configuration version cannot be parsed or is invalid then the operator tries to get either X.Y.Z-1 or perhaps X.Y.0 - THIS NEEDS TO BE DISCUSSED - [tremes note]: this is probably not very resilient approach. Suppose the following situation - there is important Insights recommendation targeting version 4.15.10, but we want to additionally add some new data for the same version 4.15.10, but the configuration is not correct after recent changes and thus cannot be parsed or validated -> with this approach we lose the data for the important recommendation. 

#### Hypershift [optional]

Does the design and implementation require specific details to account for the Hypershift use case?
See https://github.com/openshift/enhancements/blob/e044f84e9b2bafa600e6c24e35d226463c2308a5/enhancements/multi-arch/heterogeneous-architecture-clusters.md?plain=1#L282


### Risks and Mitigations

Risk: The external configuration is not available or cannot be parsed or is invalid.

Mitigation: The Insights operator should provide information about the version of the external configuration it is currently working with and it should also provide information in case of parsing or validation failure. This information must be available in the Insights archive so that we can analyse potential issues.  !!! this requires discussion about caching and versioning !!!

Risk: No data found/gathered for the specific configuration/request.

Mitigation: This should not be a major issue and should only affect the relevant Insights recommendation that requires the data. The Insights archive metadata and also the `insightsoperator.operators.openshift.io` CR should indicate that no matching data was found in the cluster for a particular GVK request.

Risk: Increased archive size or potentially big archives.

Mitigation: There is an archive size limitation on the Insights Operator side.

### Drawbacks

The idea is to find the best form of an argument why this enhancement should
_not_ be implemented.  

What trade-offs (technical/efficiency cost, user experience, flexibility, 
supportability, etc) must be made in order to implement this? What are the reasons
we might not want to undertake this proposal, and how do we overcome them?  

Does this proposal implement a behavior that's new/unique/novel? Is it poorly
aligned with existing user expectations?  Will it be a significant maintenance
burden?  Is it likely to be superceded by something else in the near future?


## Design Details

### Open Questions [optional]

* How to limit the number of resources gathered? Suppose there is a GVK and there are X instances of this GVK. How can we limit it? Do we need to distinguish between namespaced and cluster-wide resources? 
* How does this proposal fit with the existing techpreview API defined in the [On demand data gathering](../insights/on-demand-data-gathering.md) enhancement proposal? The point is that the existing techpreview API allows the cluster admin to exclude some specific gatherers/data and the exclusion depends on the gatherer name. Should we introduce some naming in the external configuration file?  
* Should we consider an option of providing or serving a different external configuration for HyperShift clusters? 
* Do we want to "transform" some existing gatherers into the external configuration?
* Does each new change in the configuration file mean a new version? The question is whether an update of configuration X.Y.Z is still the same X.Y.Z version or if the versioning of the configuration will be more granular - i.e X.Y.Z-alpha or X.Y.Z-1 or something similar. The problem of the second approach is how the operator can get the newer or latest version. Perhaps the endpoint with X.Y.Z version can only serve as a kind of nagivation in the sense that it would provide a list of corresponding versions. This would require at least two requests from the operator to the external service - but the operator can then cache (in memory) the latest version for the given z-stream and from time to time check if there's some newer version.  

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
  - [`alpha`, `beta`, `stable` in upstream Kubernetes][maturity-levels]
  - `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

**If this is a user facing change requiring new or updated documentation in [openshift-docs](https://github.com/openshift/openshift-docs/),
please be sure to include in the graduation criteria.**

**Examples**: These are generalized examples to consider, in addition
to the aforementioned [maturity levels][maturity-levels].

#### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

#### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

#### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary
  disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to
  this should be identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version.

### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

### Operational Aspects of API Extensions

The proposal does not add any API extensions and so there is not operational aspects to consider. 

#### Failure Modes

- Describe the possible failure modes of the API extensions.
- Describe how a failure or behaviour of the extension will impact the overall cluster health
  (e.g. which kube-controller-manager functionality will stop working), especially regarding
  stability, availability, performance and security.
- Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
  and add them as reviewers to this enhancement.

#### Support Procedures

Describe how to
- detect the failure modes in a support situation, describe possible symptoms (events, metrics,
  alerts, which log output in which component)

  Examples:
  - If the webhook is not running, kube-apiserver logs will show errors like "failed to call admission webhook xyz".
  - Operator X will degrade with message "Failed to launch webhook server" and reason "WehhookServerFailed".
  - The metric `webhook_admission_duration_seconds("openpolicyagent-admission", "mutating", "put", "false")`
    will show >1s latency and alert `WebhookAdmissionLatencyHigh` will fire.

- disable the API extension (e.g. remove MutatingWebhookConfiguration `xyz`, remove APIService `foo`)

  - What consequences does it have on the cluster health?

    Examples:
    - Garbage collection in kube-controller-manager will stop working.
    - Quota will be wrongly computed.
    - Disabling/removing the CRD is not possible without removing the CR instances. Customer will lose data.
      Disabling the conversion webhook will break garbage collection.

  - What consequences does it have on existing, running workloads?

    Examples:
    - New namespaces won't get the finalizer "xyz" and hence might leak resource X
      when deleted.
    - SDN pod-to-pod routing will stop updating, potentially breaking pod-to-pod
      communication after some minutes.

  - What consequences does it have for newly created workloads?

    Examples:
    - New pods in namespace with Istio support will not get sidecars injected, breaking
      their networking.

- Does functionality fail gracefully and will work resume when re-enabled without risking
  consistency?

  Examples:
  - The mutating admission webhook "xyz" has FailPolicy=Ignore and hence
    will not block the creation or updates on objects when it fails. When the
    webhook comes back online, there is a controller reconciling all objects, applying
    labels that were not applied during admission webhook downtime.
  - Namespaces deletion will not delete all objects in etcd, leading to zombie
    objects when another namespace with the same name is created.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Alternatives

We are not aware of any similar alternatives. Clearly, we can stick with the current implementation, which does not allow us to respond quickly to the current new Insights recommendations and corresponding data requirements. We do not have a better proposal that would help us avoid the Insights Operator backport process when new potential recommendation for an existing (released) version of X.Y.Z. come up. 

## Infrastructure Needed [optional]

This proposal requires an external service (running in console.redhat.com) exposing the definition of the data gathered by the Insights Operator. At the same time there will very likely be a tooling preparing/building the content for this service. 


