---
title: Different protocols in the same Service definition with type=LoadBalancer
authors:
  - "@janosi"
owning-sig: sig-network
participating-sigs:
  - sig-cloud-provider
reviewers:
  - "@thockin"
  - "@dcbw"
  - "@andrewsykim"
approvers:
  - "@thockin"
editor: TBD
creation-date: 2020-01-03
last-updated: 2020-01-03
status: provisional
see-also:
replaces:
superseded-by:
---

# different protocols in the same service definition with type=loadbalancer

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories [optional]](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Implementation Details/Notes/Constraints [optional]](#implementation-detailsnotesconstraints-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Examples](#examples)
      - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
      - [Beta -&gt; GA Graduation](#beta---ga-graduation)
      - [Removing a deprecated flag](#removing-a-deprecated-flag)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Drawbacks [optional]](#drawbacks-optional)
- [Alternatives [optional]](#alternatives-optional)
- [Infrastructure Needed [optional]](#infrastructure-needed-optional)
<!-- /toc -->

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

The `Different protocols in the same Service definition with type=LoadBalancer` feature enables the creation of such LoadBalancer Services that have different port definitions with different protocols. 

## Motivation

The ultimate goal of the feature is to support the use cases of the users when they want to expose their applications via a single IP address but different L4 protocols on a cloud provided load balancer. 
The following issue and PR shows the interest of the users in this feature:
https://github.com/kubernetes/kubernetes/issues/23880
https://github.com/kubernetes/kubernetes/pull/75831

### Goals

The goals of this KEP are:

- analyze the impact of this feature in relation considering the load balancer implementation of cloud providers
- define how the activation of this feature could be made configurable if the load balancer implementations of the cloud providers requires such optional behavior

### Non-Goals


## Proposal

The first thing proposed here is to lift the hardcoded limitation in Kubernetes that currently rejects Service definitions with different protocols if their type is LoadBalancer. Kubernetes would not reject Service definitions like this from that point:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mixed-protocol
spec:
  type: LoadBalancer
  ports:
    - name: dns-udp
      port: 53
      protocol: UDP
    - name: dns-tcp
      port: 53
      protocol: TCP
  selector:
    app: my-dns-server
  ```

Once that limit is removed those Service definitions will reach the Cloud Provider LB controller implementations. The logic of the particular Cloud Provider LB controller and of course the actual capabilities and architecture of the backing Cloud Load Balancer services determines how the actual exposure of the application really mnaifests. For this reason it is important to understand the capabilities of those backing services and to design this feature accordingly.

### User Stories [optional]

#### Story 1

As a Kubernetes cluster user I want to expose my application that provides its services on different protocols on a single cloud provider load balancer IP. In order to achieve this I want to define different `ports` with different protocols in my Service definition with `type: LoadBalancer`

### Implementation Details/Notes/Constraints [optional]

#### Alibaba

The Alibaba CPI supports both UDP and TCP in Service definitions and can configure the SLB listeners with the protocols defined in the Service.
The Alibaba SLB supports TCP and UPD on listeners, an own listener must be configured for each protocol. 
The number of listeners does not affect SLB pricing.
https://www.alibabacloud.com/help/doc-detail/74809.htm

A user can ask for an internal TCP/UPD Load Balancer via a K8s Service definition that also has the annotation `service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"`. Internal SLBs are free.

#### Azure

Azure Cloud Provider's LB documentation: https://github.com/kubernetes-sigs/cloud-provider-azure/blob/master/docs/services/README.md

The Azure Cloud Provider already supports the usage of both UDP and TCP protocols in the same Service definition. It is achieved with the CPI specific annotation `service.beta.kubernetes.io/azure-load-balancer-mixed-protocols`. If this key has value `true` in the Service definition the Azure CPI adds the other protocol value (UDP or TCP) to its internal Service representation, and as a consequence it also manages 2 load balancer rules for the specific frontend. 
Only TCP and UDP are supported in the mixed protocol configuration.
Basic Azure Load Balancers are free.
Pricing of the Standard Azure Load Balancer is based on load-balancing rules and outbound rules.  There is a flat price up to 5 rules, on top of which every new forwarding rule has additional cost. 
https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview#pricing

A user can ask for an internal Azure Load Balancer via a K8s Service definition that also has the annotation `service.beta.kubernetes.io/azure-load-balancer-internal: true`. There is not any limitation on the usage of mixed service protocols with internal LBs in the Azure CPI.

#### AWS

The AWS CPI does not support other protocol than TCP for load balancers.
If this restriction were removed from the AWS CPI then AWS NLB could support TCP and UDP protocols behind the same IP address. An NLB Listener can be configured to listen both on TCP and UDP protocols.
From pricing perspective the AWS NLB has the following model:
https://aws.amazon.com/elasticloadbalancing/pricing/
It is rather usage based than "number of protocols" based, though it is true that TCP and UDP has separated quotas in the pricing units.

A user can ask for an internal TCP/UPD Load Balancer via a K8s Service definition that also has the annotation `service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0`. So far the author could not find any difference in the usage and pricing of those when compared to the external LBs - except the pre-requisite of a private subnet on which the LB can be deployed.

#### GCE

The GCE CPI supports both TCP and UDP protocols in Services. Other protocols are not supported.
GCE/GKE creates Network Load Balancers based on the K8s Services with type=LoadBalancer. In GCE there are "forwarding rules" that define how the incoming traffic shall be forwarded to the compute instances. A single forwarding rule can support either TCP or UDP but not both. In order to have both TCP and UPD forwarding rules we have to create separate forwarding rule instances for those. Two or more forwarding rules can share the same external IP if
- the network load balancer type is External
- the external IP address is not ephemeral but static

Forwarding rule pricing is per rule: there is a flat price up to 5 forwarding rule instances, on top of which every new forwarding rule has additional cost. 

https://cloud.google.com/compute/network-pricing#forwarding_rules_charges

As we can see two different protocols in the same K8s Service definition would result in the creation of 2 forwarding rules in GCP, which has the same fixed price only up to 5 forwarding rule instances, but on top of that every new rule means extra cost.

A user can ask for an internal TCP/UPD Load Balancer via a K8s Service definition that also has the annotation `cloud.google.com/load-balancer-type: "Internal"`. Forwarding rules are also part of the GCE Internal TCP/UPD Load Balancer architecture, but in case of Internal TCP/UPD Load Balancer it is not supported to define different forwarding rules with different protocols for the same IP address. That is, for Services with type=LoadBalancer and with annotation `cloud.google.com/load-balancer-type: "Internal"` this feature would not be supported.

#### IBM Cloud

The VPC Load Balancer supports only TCP.
NLB supports both TCP and UDP. The usage of NLB does not have pricing effects, it is part of the IKS basic package.

#### OpenStack

The OpenStack CPI both UDP and TCP in Service definitions and can configure the Octavia listeners with the protocols defined in the Service.
Octavia supports TCP and UPD on listeners, an own listener must be configured for each protocol. 

#### Tencent Cloud

The Tencent Cloud CPI supports both UDP and TCP in Service definitions and can configure the CLB listeners with the protocols defined in the Service.
The Tencent Cloud CLB supports TCP and UPD on listeners, an own listener must be configured for each protocol. 
The number of listeners does not affect CLB pricing. CLB pricing is time (day) based and not tied to the number of listeners.
https://intl.cloud.tencent.com/document/product/214/8848

A user can ask for an internal Load Balancer via a K8s Service definition that  has the annotation `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxxxxxxx`. Internal CLBs are free.


### Risks and Mitigations

The goal of the current restriction on the K8s API was to prevent an unexpected extra charging for Load Balancers that were created based on Services with mixed protocol definitions.
If the current limitation is removed without any option control we introduce the same risk. Let's see which clouds are exposed:
Alibaba: the pricing here is not protocol or forwarding rule or listener based. No risk.
Azure: Azure pricing is indeed based on load-balancing rules, but this cloud already supports mixed protocols via annotations. There is another risk for Azure, though: if the current restriction is removed from the K8s API, the Azure CPI must be prepared to handle Services with mixed protocols.
AWS: there is no immediate impact on the pricing side as the AWS CPI limits the scope of protocols to TCP only.
GCE: here the risk is valid once the user exceeds the threshold of 5 forwarding rules.
IBM Cloud: no risk.
OpenStack: here the risk is valid as there is almost no chance to analyze all the public OpenStack cloud providers with regard to their pricing policies
Tencent Cloud: no risk

A possible mitigation is to put the feature behind option control. 

## Design Details

The implementation of the basic logic is ready in this PR:
https://github.com/kubernetes/kubernetes/pull/75831

Currently a feature gate is used to control its activation status. Though if we want to keep this feature behind option control even when it reaches its GA state we should come up with a different solution, as feature gates are used to control the status of a feature as that graduates from alpha to GA, and they are not meant for option control for features with GA status.

### Option Control Alternatives

#### Annotiation in the Service definition

In this alternative we would have a new annotation in the `kubernetes.io` annotation namespace. Example: `kubernetes.io/mixedprotocol`. If this annotation is applied on a Service definition the Service would be accepted by the K8s API.

Pro: 
- a kind of "implied conduct" from the user's side. The user explicitly defines with the usage of the annotation that the usage of multiple protocols on the same LoadBalancer is accepted
- Immediate feedback from the K8s API if the user configures a Service with mixed protocol set but without this annotation
Con:
- Additional configuration task for the user
- Must be executed for each and every Service definitions that are to define different protocols for the same LB

#### Merging Services in CPI

This one is not really a classic option control mechanism. The idea comes from the current practice implemented in MetalLB: https://metallb.universe.tf/usage/#ip-address-sharing

I.e. if a cloud provider wants to support this feature the CPI must have a logic to apply the Service definitions with a common key value (for exampele LoadBalancerIP) on the same LoadBalancer instance. If the CPI does not implement this support it will work as it does currently.
Pro:
- the cloud provider can decide when to support this feature, and until that it works as currently for this kind of config
Con:
- the users must maintain two Service instances
- Atomic update can be a problem - the key must be such a Service attribute that cannot be patched

### Test Plan

**Note:** *Section not required until targeted at a release.*

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

#### Examples

These are generalized examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

##### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

##### Beta -> GA Graduation

- N examples of real world usage
- N installs
- More rigorous forms of testing e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least 2 releases between beta and GA/stable, since there's no opportunity for user feedback, or even bug reports, in back-to-back releases.

##### Removing a deprecated flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality which deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include [conformance tests].**

[conformance tests]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to make on upgrade in order to make use of the enhancement?

### Version Skew Strategy

If applicable, how will the component handle version skew with other components? What are the guarantees? Make sure
this is in the test plan.

Consider the following in developing a version skew strategy for this enhancement:
- Does this enhancement involve coordinating behavior in the control plane and in the kubelet? How does an n-2 kubelet without this feature available behave when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI or CNI may require updating that component before the kubelet.

## Implementation History

Major milestones in the life cycle of a KEP should be tracked in `Implementation History`.
Major milestones might include

- the `Summary` and `Motivation` sections being merged signaling SIG acceptance
- the `Proposal` section being merged signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded

## Drawbacks [optional]


## Alternatives [optional]

## Infrastructure Needed [optional]

