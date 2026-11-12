---
title: "VCF 9 Holodeck LAB: Making the Most of 128 Cores for a Full-Stack Lab"
date: 2026-11-12T00:00:00+01:00
draft: true
description: "How I adjusted CPU topology across seven nested ESX hosts in Holodeck to stay within my 128-core personal-use VCF entitlement obtained through VCP-VCF Architect and VMUG Advantage."
slug: "vcf-9-holodeck-128-core-cpu-topology"
translationKey: "vcf-9-holodeck-128-core-cpu-topology"
tags: ["VMware", "VCF 9", "Holodeck", "Homelab", "Licensing", "Nested ESX", "VMUG"]
categories: ["VMware"]
showToc: true
TocOpen: false
cover:
  image: "/img/vcf-holodeck-128-cores/00-cover.webp"
  alt: "VMware Cloud Foundation 9 lab with four Management hosts and three Workload hosts, totaling 128 licensed cores"
  caption: "VCF 9 in Holodeck: Full Stack in 128 Cores"
  relative: true
---

## The Problem Wasn't the Number of Cores. Or Was It?

Building a complete VMware Cloud Foundation lab is a useful exercise: infrastructure requirements, available compute resources, and licensing all come into play.

I was working with **Holodeck and VMware Cloud Foundation 9**, using a nested deployment with **seven ESX hosts**: four in the **Management Domain** and three in a separate **Workload Domain**. That was exactly the layout I wanted to preserve for hands-on learning and experimentation.

The environment was running, but a licensing check revealed a problem: **the four Management Domain hosts alone consumed all 128 cores in my personal-use VCF entitlement**. Nothing was left for the three Workload Domain hosts.

<!-- IMAGE 01 — Save your own inventory screenshot as static/img/vcf-holodeck-128-cores/01-holodeck-inventory.webp -->
![Holodeck inventory showing four ESX hosts in the Management Domain and three in the Workload Domain](/img/vcf-holodeck-128-cores/01-holodeck-inventory.webp)
*Figure 1 — The seven nested ESX hosts in the Holodeck lab, split across two domains.*

Before removing hosts or abandoning the second domain, I wanted to understand **how VCF was actually counting licensed cores**.

## VCF Licensing: A Minimum of 16 Cores per CPU

VMware Cloud Foundation (VCF) and VMware vSphere Foundation (VVF) licensing use a **minimum of 16 licensed cores per physical CPU (socket)**, even when a processor has fewer cores. On a multi-socket host, the minimum applies to each CPU independently.

In simplified form, for each host:

`Licensed cores = sum, across all sockets, of max(16, cores on that socket)`

| CPU topology | Available cores | Licensed cores |
|---|---:|---:|
| 1 socket × 8 cores | 8 | 16 |
| 1 socket × 12 cores | 12 | 16 |
| 1 socket × 20 cores | 20 | 20 |
| 2 sockets × 12 cores | 24 | 32 |
| 2 sockets × 16 cores | 32 | 32 |
| 2 sockets × 24 cores | 48 | 48 |

On a physical host, the number of CPU sockets is determined by the hardware. In a **nested ESX environment**, however, ESX itself runs as a virtual machine. The CPU sockets and cores seen by the ESX guest depend on the VM's virtual hardware configuration.

This is not just a theoretical distinction. [Broadcom KB 431120](https://knowledge.broadcom.com/external/article/431120/resolving-unexpected-license-capacity-re.html) specifically addresses unexpectedly high license consumption caused by the virtual CPU topology of nested ESX hosts in VCF Operations 9.0.

## Four Hosts Were Enough to Consume All 128 Cores

Initially, each Management Domain ESX VM was configured with **two virtual sockets and 16 cores per socket**, or **32 vCPUs per host**.

| Management Domain | Initial topology | Required cores |
|---|---|---:|
| ESX 01 | 2 sockets × 16 cores | 32 |
| ESX 02 | 2 sockets × 16 cores | 32 |
| ESX 03 | 2 sockets × 16 cores | 32 |
| ESX 04 | 2 sockets × 16 cores | 32 |
| **Total** | **4 hosts** | **128** |

I did not even need to add the Workload Domain requirements to identify the issue: **128 out of 128 cores were already accounted for**.

<!-- IMAGE 02 — Save an original screenshot or a clearly labeled reconstructed diagram as static/img/vcf-holodeck-128-cores/02-mgmt-cpu-before.webp -->
![Original nested ESX CPU configuration with 32 vCPUs, two sockets, and 16 cores per socket](/img/vcf-holodeck-128-cores/02-mgmt-cpu-before.webp)
*Figure 2 — The original Management Domain host configuration: 2 sockets × 16 cores.*

It is worth noting that this initial configuration was aligned with a sound performance-oriented setup: in scenarios like this, ESX typically exposes a CPU topology that remains as consistent as possible with the underlying hardware, also taking **NUMA** and memory locality into account.

That said, **moving from 2 × 16 to 1 × 32 would not have reduced license consumption**: in both cases the total would still have been 32 cores. To achieve a real licensing benefit, I had to change not only the virtual socket layout, but also **the total number of vCPUs assigned**.

## The New Layout: 80 Cores for Management, 48 for Workloads

My goal was to **make the most of the 128 cores granted for learning and experimentation**, while keeping all seven hosts and accepting a conscious trade-off in virtual compute resources.

For each Management Domain host, I changed the configuration from **2 sockets × 16 cores** to **1 socket × 20 cores**. The licensing requirement dropped from 32 to 20 cores per host: **4 × 20 = 80 cores**.

<!-- IMAGE 03 — Save the final Management ESX VM CPU settings as static/img/vcf-holodeck-128-cores/03-mgmt-cpu-after.webp -->
![Management ESX VM CPU topology after the change: 20 vCPUs, one socket with 20 cores](/img/vcf-holodeck-128-cores/03-mgmt-cpu-after.webp)
*Figure 3 — Management Domain: 20 vCPUs presented as a single 20-core socket.*

That left **48 cores**, enough to configure each of the three Workload Domain hosts with **1 socket × 16 cores**.

<!-- IMAGE 04 — Save the final Workload ESX VM CPU settings as static/img/vcf-holodeck-128-cores/04-wld-cpu-after.webp -->
![Workload ESX VM CPU topology: 16 vCPUs presented as one 16-core socket](/img/vcf-holodeck-128-cores/04-wld-cpu-after.webp)
*Figure 4 — Workload Domain: one 16-core socket on each of the three hosts.*

| Domain | Hosts | Final CPU topology | Licensed cores |
|---|---:|---|---:|
| Management Domain | 4 | 1 socket × 20 cores | 80 |
| Workload Domain | 3 | 1 socket × 16 cores | 48 |
| **Total** | **7** | | **128** |

**128 cores available, 128 cores used.** No hosts removed and no workload domain sacrificed.

## How to Change CPU Topology on a Nested ESX 9 Host

The modification **is not performed from within the nested ESX**. Instead, you must adjust the settings of the **ESX VM—which is itself running on an ESX instance installed on the physical server**—by connecting to the vSphere Client of the underlying infrastructure that hosts Holodeck.

### 1. Check CPU Settings and Virtual Hardware

In the outer vSphere Client, select the ESX VM and open **Edit Settings**. Record the current total vCPU count, VM hardware compatibility, and cores-per-socket value.

The setting's location depends on the outer vSphere version and the VM's virtual hardware version:

- **vSphere 8 with virtual hardware version 20 or later:** `Edit Settings → VM Options → CPU Topology`.
- **Earlier virtual hardware versions:** the setting may be under `Edit Settings → Virtual Hardware → CPU`.

See [Broadcom KB 407648](https://knowledge.broadcom.com/external/article/407648/cores-per-socket-setting-location-has-be.html). The fact that the guest runs ESX 9 does not change where the VM setting is managed; the outer hypervisor and VM compatibility determine the available UI.

### 2. Prepare for Host Shutdown

Before making changes, check where the cluster's VMs and services reside. If vSAN is in use, evaluate the appropriate maintenance mode and any impact on data accessibility or availability. Do not modify every host in a domain simultaneously without checking dependencies.

Shut down nested ESX cleanly and **power off the ESX VM** at the outer layer.

### 3. Apply the New CPU Topology

With the VM powered off, open **Edit Settings** again and apply the appropriate configuration:

| Setting | Management hosts | Workload hosts |
|---|---:|---:|
| **Total CPUs (vCPUs)** | **20** | **16** |
| **Cores per Socket** | **20** | **16** |
| Resulting virtual sockets | 1 | 1 |

If the VM is using automatic CPU topology assignment (*Assigned at power on* / *Automatic vTopology*), select the desired cores-per-socket value manually and confirm the resulting virtual socket count.

Save the settings, power the ESX VM back on, and wait for the host to become healthy in its cluster. Repeat **one host at a time**, taking service status into account.

### 4. Verify the Topology Seen by ESX 9

In the vSphere Client managing the nested VCF environment, select the host and review its **Hardware / Processors** information.

The expected result is:

- **Management:** 1 socket, 20 cores.
- **Workload:** 1 socket, 16 cores.

<!-- IMAGE 05 — Save the nested ESX Hardware / Processors view as static/img/vcf-holodeck-128-cores/05-esx9-guest-topology.webp -->
![CPU topology detected by nested ESX 9 after changing the underlying VM configuration](/img/vcf-holodeck-128-cores/05-esx9-guest-topology.webp)
*Figure 5 — The topology seen inside ESX 9 must match the configuration applied on the outer hypervisor.*

### 5. Confirm the Core Count in VCF Operations

Finally, verify what the licensing system actually reports. VCF 9 uses centralized licensing through VCF Operations and the VCF Business Services Console. Correct VM settings are not enough unless the updated topology is reflected in license consumption.

The navigation path differs between releases:

- **VCF Operations 9.0:** `License Management → Registration`.
- **VCF Operations 9.1:** `Manage → Licensing → Licenses & Registration`.

Depending on whether the environment runs in connected or disconnected mode, license information may need to be refreshed or imported. Check the license capacity allocated to each vCenter and the cores required by all seven hosts.

<!-- IMAGE 06 — Save the VCF Operations licensing panel showing the total 128-core capacity as static/img/vcf-holodeck-128-cores/06-vcf-ops-128-cores.webp -->
![VCF Operations license capacity and host consumption totaling 128 cores](/img/vcf-holodeck-128-cores/06-vcf-ops-128-cores.webp)
*Figure 6 — The final figure to validate in VCF Operations: 80 Management cores plus 48 Workload cores.*

## This Wasn't a Performance Optimization

The purpose of this change was clear: **use my personal-use licensing entitlement as effectively as possible for lab work**.

By reducing the virtual CPU resources assigned to nested hosts, I gave up some of the original virtual compute capacity. Under load, this configuration could increase contention or affect response times, although I do not yet have measurements that quantify the real impact.

Nested virtualization also adds its own variables, including vCPU scheduling on the outer hypervisor and interactions with the underlying physical NUMA topology.

For a personal lab, this is an acceptable trade-off. **It is not a sizing recommendation for production systems**, where hardware, workloads, supportability, and licensing must be assessed together.

## Where the 128 Cores Came From: VCP-VCF Architect and VMUG Advantage

These 128 cores did not come from a standard commercial license. After earning my **VCP-VCF Architect** certification and joining **VMUG Advantage**, I obtained access to a **128-core VMware Cloud Foundation personal-use entitlement**.

It is a valuable opportunity for anyone who wants to study the platform on a real deployment rather than relying exclusively on guided exercises.

Under the program described by VMware and VMUG, the license is **for non-production use, does not include support, and is initially valid for one year**. Annual renewal is possible for up to three years subject to program rules and maintaining the required membership. As eligibility and program terms can change, always check the current conditions before planning a lab.

Giovanni Dominoni provides a useful explanation of the benefits and access process in [*Homelab VMware License: a great opportunity for VVF and VCF certified people*](https://www.giovannidominoni.it/homelab-vmware-license-a-great-opportunity-for-vvf-and-vcf-certified-people/). For the latest program terms, refer to [VMUG Advantage](https://www.vmug.com/advantage).

<!-- OPTIONAL IMAGE 07 — To show the entitlement, save a sanitized screenshot as static/img/vcf-holodeck-128-cores/07-vmug-entitlement.webp and uncomment the line below. NEVER expose license keys, tokens, IDs, or personal details. -->
<!-- ![Broadcom portal showing a personal-use 128-core VCF entitlement](/img/vcf-holodeck-128-cores/07-vmug-entitlement.webp) -->

## Next Step: Measure What Happens Under Load

The CPU topology change solved the licensing problem, but it also introduced another learning opportunity: **how does VCF behave when CPU resources are constrained?**

In future tests, I want to examine CPU utilization, CPU Ready, co-stop, contention, and appliance behavior, separating the effects of host sizing from those introduced by nested virtualization as far as possible.

Results from a nested lab should not be treated as production benchmarks. They can, however, help us understand architectural trade-offs and ask better questions when sizing a real environment.

## Conclusions

This exercise reminded me of the need to distinguish **physical resources, virtual resources, and licensed capacity**. They are related, but not interchangeable.

By adjusting the CPU topology of the ESX VMs, I was able to use all 128 cores of my personal VCF entitlement while keeping all seven hosts and the separation between Management and Workload Domains.

The trade-off was a smaller vCPU allocation, which I plan to evaluate through targeted testing. In return, I kept a complete, correctly licensed environment that I can continue to learn from.

**Sometimes, a lab's limitations are exactly what make it useful.**

## References

- [Broadcom KB 313548 — Counting Cores for VMware Cloud Foundation and vSphere Foundation](https://knowledge.broadcom.com/external/article/313548).
- [Broadcom KB 431120 — Resolving unexpected License capacity requirements in VCF Operations](https://knowledge.broadcom.com/external/article/431120/resolving-unexpected-license-capacity-re.html).
- [Broadcom KB 407648 — Cores per Socket setting location in vSphere](https://knowledge.broadcom.com/external/article/407648/cores-per-socket-setting-location-has-be.html).
- [Broadcom KB 453634 — Adding ESX host to vCenter 9.x: license capacity and VCF Operations paths](https://knowledge.broadcom.com/external/article/453634/adding-esx-host-to-vcenter-9x-fails-with.html).
- [VMUG — VCP-VCF License Access for Personal Use (program details and requirements)](https://www.vmug.com/wp-content/uploads/2025/03/VCP-VCF-License-Access-for-Personal-Use-Advantage-Benefit_March_2025.pdf).
- [Giovanni Dominoni — Homelab VMware License: a great opportunity for VVF and VCF certified people](https://www.giovannidominoni.it/homelab-vmware-license-a-great-opportunity-for-vvf-and-vcf-certified-people/).
