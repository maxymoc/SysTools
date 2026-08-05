---
title: "The hidden capacity of the data center: designing VMware Cloud Foundation 9.1 before buying new hardware"
date: 2026-08-04T17:40:00+02:00
draft: true
description: "A VMware cluster at 65% utilization is not necessarily close to saturation. How to distinguish installed capacity from usable capacity before buying new hardware with VMware Cloud Foundation 9.1."
translationKey: "capacita-nascosta-vcf-9-1"
tags:
  - VMware Cloud Foundation
  - Private Cloud
  - Hybrid Cloud
  - Capacity Planning
  - vSAN
  - FinOps
  - Infrastructure
  - Enterprise Architecture
slug: "hidden-capacity-vcf-9-1"
cover:
  image: "img/capacita-nascosta-vcf-9-1.png"
  alt: "Architect analysing and optimizing the capacity of a VMware Cloud Foundation 9.1 data center"
  relative: true
---

## Introduction

![From a fragmented data center to a governed platform](/img/vcf-capacity-transition.png)

This article draws on many years spent designing, transforming and operating VMware infrastructures and, above all, on Broadcom Architect training. Those courses go beyond product features: they teach architects to make decisions traceable, separate requirements from constraints, and state risks and trade-offs explicitly. The method used here follows the same sequence: start from the operational problem, measure it, and only then choose the technology.

In 2026, hardware availability and cost are only part of the problem. Uncertain lead times, the growth of AI workloads, higher compute density and licensing changes make a strategy based simply on adding servers increasingly fragile.

When a VMware cluster reaches 60–70% utilization, planning new hosts is understandable, but often premature. That percentage does not identify the constrained resource, recoverable capacity, the impact of oversized VMs, or the headroom required for resilience. It does not distinguish between compute, memory, storage, network or architectural constraints.

The useful question is not how many servers to buy, but how much usable capacity already exists and which architecture can exploit it without reducing resilience, supportability or performance.

VMware Cloud Foundation 9.1 brings compute, storage, networking, security and operations into a common private-cloud platform. It does not, by itself, correct an inefficient design. Technology expands the available options; method determines which options to use and why.

Broadcom references: [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html) · [Announcing VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/announcing-vcf-9-1-modern-private-cloud-built-for-efficiency-and-resilience/) · [Fleet Management](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/overview-of-vmware-cloud-foundation-9/what-is-vmware-cloud-foundation-and-vmware-vsphere-foundation/vcf-operations-overview/fleet-management.html)

## The misleading utilization percentage

![Average utilization does not identify the real constraint](/img/vcf-utilization-trap.png)

A cluster at 65% utilization is not necessarily close to saturation. A cluster at 40% is not necessarily healthy. The same average may hide allocated but unused memory, excessive vCPU, nearly full datastores, rising storage latency, uneven VM placement, NUMA constraints, forgotten snapshots or obsolete workloads.

The first mistake is to confuse installed capacity with usable capacity. In simplified form:

**Usable capacity = physical capacity − resilience reserve − overhead − inefficiency − operating headroom**

Physical capacity is the nameplate figure. Usable capacity is what can be assigned to workloads while preserving the expected behaviour during maintenance, failures and demand peaks. This figure—not the nominal sum of resources—should guide capacity planning and investment.

Broadcom references: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [Scale, Simplify, and Secure Private Cloud Operations with VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/scale-simplify-and-secure-your-private-cloud-operations-with-vcf-9-1/)

## Measure the real constraint

![Separate analysis of compute, memory, storage and network](/img/vcf-four-domains.png)

Before adding hosts, identify the resource that is actually running out of headroom. In many environments compute appears saturated while CPU and memory still have capacity and the real constraint is storage: usable space, latency, throughput, rebuild time or dependence on an end-of-life array.

Adding servers in this scenario increases compute capacity while leaving the bottleneck unchanged. The result is underused new hardware and an unresolved problem.

The assessment should be split into four domains. Compute analysis covers average and peak utilization, CPU Ready, Co-Stop, NUMA configuration, vCPU-to-core ratio and remaining capacity after a host failure. Memory analysis covers working set, active and consumed memory, ballooning, compression and swapping. Storage analysis requires usable capacity, protection, latency, IOPS, throughput, growth and rebuild times. Network analysis covers bandwidth, redundancy, oversubscription, packet loss, retransmission and latency between domains or sites.

Only after this analysis can the team decide whether it needs more compute, more storage, a network redesign or a better distribution of existing resources.

Broadcom references: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html)

## Recover capacity through rightsizing

![Rightsizing as a continuous governance process](/img/vcf-rightsizing.png)

Rightsizing is not an indiscriminate cut to RAM and vCPU. It is a continuous process of measurement, correction and verification. An oversized VM reduces DRS effectiveness, complicates placement during failover, can create inefficient NUMA configurations and limits the hosts on which the workload can restart.

Decisions must use historical data that includes peaks, seasonality, batch windows, backup activity and expected growth. Reducing a VM on the basis of a daily average is as risky as leaving it at twice the required size for years.

A sound process observes behaviour, prepares a proposal, involves the application owner, applies the change, verifies the result and keeps a practical rollback path. The benefit is not only financial: a more predictable platform is easier to protect, update and expand.

Broadcom references: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [Scale, Simplify, and Secure Private Cloud Operations with VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/scale-simplify-and-secure-your-private-cloud-operations-with-vcf-9-1/)

## Memory Tiering: more capacity, not free memory

![DRAM and an NVMe tier extending memory capacity](/img/vcf-memory-tiering.png)

Memory Tiering is an ESX capability that uses local NVMe devices as a secondary memory tier behind DRAM. The most active pages remain on the faster tier, while colder pages can reside on NVMe. Its purpose is to increase effective capacity when memory is the real constraint and CPU still has headroom. NVMe devices assigned to the memory tier must be dedicated to that function and must not be shared with VM datastores or vSAN.

It is not equivalent to traditional swapping and does not replace rightsizing. VM compatibility, NVMe characteristics and endurance, working set and latency sensitivity must be validated against the actual workload. The benefit must be demonstrated through metrics, testing and operational observation—not inferred from configured memory alone.

Broadcom references: [Memory Tiering considerations and best practices](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/9-1/vsphere-resource-management/memory-tiering-over-nvme/memory-tiering-considerations-and-best-practices.html) · [Using Memory Tiering for Resource Efficiency](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/using-resource-efficiency-for-memory-tiering.html) · [Configuring Memory Tiering in VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/19/more-memory-less-effort-configuring-memory-tiering-in-vcf-9-1/)

## Reuse hardware by assigning the right role

![Different hardware generations assigned to suitable roles](/img/vcf-capacity-analysis.png)

Reusing existing hardware does not mean ignoring compatibility, support or lifecycle. It means separating the server’s remaining technical value from its previous role. A host no longer suitable for demanding workloads may remain appropriate for infrastructure services, test environments or clusters with predictable performance profiles.

The Management Domain is also critical; it is not a natural destination for obsolete hardware. Reuse is sustainable only when capacity, availability, drivers, firmware and support remain aligned with the full expected lifecycle. The Broadcom Compatibility Guide and OEM guidance must come before the financial decision.

Broadcom references: [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html) · [Broadcom Compatibility Guide](https://compatibilityguide.broadcom.com/)

## Do not force uniformity across hardware generations

![Homogeneous clusters and separate lifecycles](/img/vcf-infrastructure-domains.png)

Servers from the same vendor but from different generations may require different firmware, drivers, vendor add-ons and BIOS settings. Treating them as a single homogeneous set increases risk and complicates remediation.

With image-based lifecycle management, the goal is not to build a universal image. It is to define coherent images for homogeneous clusters and maintain clear boundaries between hardware groups with different requirements. Heterogeneity should not be removed on principle; it should be contained, documented and managed.

Broadcom reference: [Lifecycle Management in VMware Cloud Foundation](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/lifecycle-management-in-vmware-cloud-foundation.html)

## Choose storage for the service, not for the trend

![Storage designed around service classes](/img/vcf-storage-service-classes.png)

Adopting VCF does not mean placing every data set on the same type of storage. The decision depends on performance, protection, resilience, operating model, skills, growth, cost and compatibility. Primary storage, supplemental storage, backup repositories and replication targets can coexist as long as each role is explicit.

vSAN ESA is a natural option when the goal is an integrated, policy-driven HCI architecture, but it requires validated hardware, an appropriate network and realistic capacity and resilience calculations. A configuration that installs successfully is not automatically a configuration that can be sustained for years.

Broadcom references: [What’s New in vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html) · [vSAN documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/vsan-deployment-administration-and-monitoring.html)

## vSAN ESA: deduplication and compression without overstating the benefit

![Deduplication and compression as measurable efficiency levers](/img/vcf-vsan-esa-efficiency.png)

VCF 9.1 makes Global Deduplication generally available for vSAN ESA and introduces a new always-on compression algorithm. Deduplication runs as a cluster-wide post-processing activity; compression applies to new data and, after an upgrade, to existing blocks when they are read and rewritten. Results still depend on data type, policy and workload behaviour. Already compressed, encrypted or low-repetition data will not achieve the same reduction ratio as templates, virtual desktops or similar operating systems. In release 9.1, Global Deduplication is not supported on stretched-cluster or 2-node topologies.

Capacity planning should start from raw capacity, subtract protection, operational slack and rebuild space, and only then apply a conservative reduction ratio based on measured data. Deduplication and compression can delay expansion and reduce cost per usable terabyte; they should not remove the safety margin.

Broadcom references: [What’s New in vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html) · [vSAN Compression and Global Deduplication in VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/07/vsan-compression-and-global-deduplication-in-vcf-9-1/) · [Using Deduplication and Compression in vSAN](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/vsan-deployment-administration-and-monitoring/administering-vmware-vsan/increasing-space-efficiency-in-a-vsan-cluster/using-deduplication-and-compression-in-vsan-cluster.html)

## Design the cluster for failure, not for normal operation

![Capacity calculated for maintenance and failure scenarios](/img/vcf-resilience-capacity.png)

A cluster should not be sized only for normal operation. It must maintain service during a host failure, maintenance, an upgrade, storage rebuild or the loss of a fault domain included in the design.

Total capacity cannot therefore be fully allocated. The calculation must subtract N+1 or greater headroom, hypervisor and management overhead, maintenance space, expected growth and capacity required by protection policies.

In stretched clusters, the witness contributes to quorum but does not add workload capacity. The correct question is not how many VMs fit under normal conditions, but how many can remain operational in the failure scenario declared by the architecture.

Broadcom reference: [VMware vSAN Stretched Cluster](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/vsan-deployment-administration-and-monitoring/administering-vmware-vsan/setting-up-a-stretched-cluster.html)

## Consolidate without creating a new monolith

![Cluster boundaries aligned with hardware and requirements](/img/vcf-cluster-boundaries.png)

Reducing the number of clusters limits fragmentation, increases placement freedom and simplifies lifecycle management. Consolidating everything into one cluster, however, can create a monolith that is difficult to govern.

Boundaries should reflect hardware generation, security, application licensing, criticality, performance profile, maintenance windows, location and availability requirements. A cluster is both a technical unit and an operational boundary: it must be large enough to use resources efficiently and coherent enough to manage as one object.

Broadcom reference: [VMware Cloud Foundation 9.1 Design](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/design.html)

## Capacity planning as a FinOps function

![Capacity management and cost as part of the same decision](/img/vcf-operations-finops.png)

Every unused core, unmanaged terabyte and oversized VM has a financial effect. Capacity and cost management should therefore be read together, including the cost of unused capacity, resilience, licensing, hardware renewal and technical debt.

Applying FinOps to the private cloud does not necessarily mean charging every VM to a cost centre. It means making the link between technical choice and financial consequence visible. An oversized VM may bring an investment forward; a fragmented cluster estate may multiply reserves that cannot be shared.

Broadcom references: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [Scale, Simplify, and Secure Private Cloud Operations with VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/scale-simplify-and-secure-your-private-cloud-operations-with-vcf-9-1/)

## VCF Operations as a decision system

![Operations as a continuous cycle of measurement and correction](/img/vcf-operations-decision-cycle.png)

A traditional monitoring system reports what is not working. A mature operations platform should also show what is approaching exhaustion, where capacity can be recovered, which VMs are oversized, which hosts constrain lifecycle and which investment can be deferred.

Fleet Management centralizes the management of instances and components registered in the fleet. Deployment, initial validation and lifecycle belong to distinct but coordinated workflows. Value emerges when dashboards feed a process: data collection, baselining, analysis, corrective action, validation and capacity-plan updates.

Broadcom references: [Fleet Management](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/overview-of-vmware-cloud-foundation-9/what-is-vmware-cloud-foundation-and-vmware-vsphere-foundation/vcf-operations-overview/fleet-management.html) · [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html)

## Buy only after defining the target architecture

![Architectural framework before the bill of materials](/img/vcf-architecture-framework.png)

Before ordering new servers, define the Management and Workload Domains, number and size of clusters, reusable hardware, lifecycle, storage, network, fault domains, N+1 or N+2 capacity, growth, core-to-memory ratio, potential Memory Tiering use and conservative assumptions for vSAN ESA data reduction.

Only then does it make sense to build the bill of materials. This avoids both buying powerful servers that remain underused because the constraint lies elsewhere and buying nodes that are unbalanced for the workload and storage model. The goal is not to buy faster, but to buy with better data and within an established design.

Broadcom references: [VMware Cloud Foundation 9.1 Design](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/design.html) · [Broadcom Compatibility Guide](https://compatibilityguide.broadcom.com/)

## A practical transformation model

![A progressive path from assessment to operations](/img/vcf-evolution-model.png)

A pragmatic path can be organized into five phases. Assessment inventories components, verifies compatibility, measures real capacity, identifies the bottleneck and starts rightsizing. Design defines domains, clusters, storage, network, resilience and lifecycle. Controlled reuse assigns hardware to compatible roles and preserves homogeneous clusters. Targeted investment introduces only what is required to remove a measured constraint. Continuous operations update the capacity plan, thresholds, lifecycle and cost models.

These phases are not a rigid sequence: some activities overlap, and data collected during operations may require earlier decisions to be revisited. The constant is traceability.

Broadcom references: [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html) · [Lifecycle Management](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/lifecycle-management-in-vmware-cloud-foundation.html)

## Decisions that actually save hardware

![From hidden capacity to a governed platform](/img/vcf-data-efficiency.png)

Savings rarely depend on a single feature. They come from a coherent set of decisions: distinguish physical from usable capacity, size for failure, identify the bottleneck, apply rightsizing, consolidate where appropriate, separate different lifecycles, and use Memory Tiering and data reduction only when the data demonstrates a benefit.

The assessment may confirm the need for new servers, redirect investment toward storage or network, or defer the purchase. All are valid outcomes. The objective is not to avoid new hardware, but to avoid buying it to compensate for a problem that has not yet been understood.

Broadcom references: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html)

## Conclusions

![A more efficient, resilient and predictable platform](/img/vcf-governed-platform.png)

VMware Cloud Foundation 9.1 can be the starting point for a more integrated and governable private cloud, but the main benefit does not come from installing the platform. It comes from changing the way infrastructure decisions are made.

Existing hardware should neither be retained indiscriminately nor replaced out of habit. It should be assigned to the role in which it can continue to create value. New hardware must remove a measured constraint, not an assumption. In a mature private cloud, optimization is not a temporary measure while waiting for the next purchase; it is part of the architecture.

Final references: [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html) · [Memory Tiering best practices](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/9-1/vsphere-resource-management/memory-tiering-over-nvme/memory-tiering-considerations-and-best-practices.html) · [vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html)
