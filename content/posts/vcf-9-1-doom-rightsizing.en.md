---
title: "DOOM on VCF 9.1: a nerdy workload for Rightsizing"
date: 2026-10-21T10:15:00+02:00
draft: true
description: "Alpine Linux, DOSBox and DOOM in a VCF 9.1 Workload Domain: a minimal-footprint experiment with Rightsizing and What-If Analysis in VCF Operations."
translationKey: "vcf-9-1-doom-rightsizing"
tags:
  - "VMware Cloud Foundation"
  - "VCF Operations"
  - "Rightsizing"
  - "Reclaim"
  - "Capacity Planning"
  - "Holodeck"
  - "Linux"
  - "DOSBox"
slug: "doom-on-vcf-9-1-rightsizing"
cover:
  image: "/img/vcf-9-1-doom-rightsizing.webp"
  alt: "DOOM running on Alpine Linux in a VMware Cloud Foundation 9.1 lab"
  relative: true
---

**Minimal footprint, minimal effort, maximum fun.**

In my **VMware Cloud Foundation 9.1** lab, built with **Holodeck**, I wanted a tiny but continuously running workload inside the *Workload Domain*. The goal was not to benchmark a game; it was to give **VCF Operations** real activity to observe while exploring **Rightsizing**, capacity analysis and **What-If** scenarios.

I could have used a stress tool or a script. Instead, I went for something more fun: **running DOOM**.

## Architecture: just enough to get the job done

I chose **Alpine Linux**, with no full desktop environment: only Xorg, a graphical terminal and **DOSBox Staging** to run the shareware edition of DOOM.

| Component | Lab configuration |
|---|---|
| Infrastructure | VCF 9.1 on Holodeck |
| VM | `vm-retro`, in the Workload Domain |
| Operating system | Alpine Linux 3.24.1 x86_64 |
| Compute | 1 vCPU, 512 MB RAM |
| Storage | 8 GB, thin provisioned |
| Display | VMware SVGA / Xorg |
| Emulator | DOSBox Staging 0.82.2 |
| Application | DOOM Shareware 1.9 |

The design decision was straightforward: **keep the footprint and operational complexity low while retaining a measurable application process**.

## From a Linux console to DOOM

With Alpine repositories configured correctly (including `community`), the essential components can be installed with:

```sh
setup-xorg-base
apk update
apk add dosbox-staging sdl2 xterm
```

One unexpected issue came up during the build: the SDL3-based `sdl2-compat` layer caused startup problems with the emulator. Using **native SDL2** resolved them in this lab.

After configuring Xorg and a session with `xterm`, starting the application is simple:

```sh
# Start the graphical session as an unprivileged user
startx

# From xterm
dosbox
```

DOSBox needs a Linux directory mapped to its DOS `C:` drive. For example, **adjusting the path to your own installation**:

```dos
MOUNT C /home/retro/dos
C:
CD \GAMES\DOOMS
DOOM.EXE
```

In my lab the game directory was `C:\GAMES\DOOMS`, and the executable I used was `DOOMS.EXE`; the standard shareware archive normally uses `DOOM.EXE`.

DOOM has a built-in *attract mode*: when left at its title screen, it cycles through recorded gameplay demos without user input. It provides a small, real, not perfectly flat but **repeatable and observable** workload.

## From telemetry to decisions: Reclaim, Rightsize and What-If

DOOM is not a benchmark; it simply creates observable activity. The next step is to use the **VCF Operations Capacity Engine**, which combines metrics, historical data and policies to support operational decisions. The screenshots below cover **the lab cluster's VMs as a whole**, not just the VM running DOOM.

### Reclaim — what can I get back?

**Reclaim** identifies unused capacity, including powered-off or idle VMs, snapshots and orphaned disks. In `cluster-wld01-01a`, the analysis found **two powered-off VMs** (`retro-vm-3` and `retro-vm-bck`) and **one orphaned disk**: **3.15 GB** reclaimable in total, with **€6/month in estimated potential savings**. This is not realized cash savings; it depends on the configured cost model. Ownership, dependencies and retention policies should be checked before removing anything.

![VCF Operations Reclaim: reclaimable capacity in the lab cluster](/img/vcf-9-1-reclaim.webp)

### Rightsize — how much does a VM really need?

**Rightsize** compares allocated resources against projected requirements, identifying *oversized* and *undersized* VMs. Here `retro-vm-2` has **4 vCPUs and 4 GB RAM**, while VCF Operations recommends reducing it by **2 vCPUs and 2 GB** — resulting in a proposed **2 vCPU / 2 GB** configuration. No VM is flagged as *undersized*. This is a **recommendation, not a change already applied**; application peaks and risk policies still matter.

![VCF Operations Rightsize: CPU and memory reduction recommendation](/img/vcf-9-1-rightsize.webp)

### What-If — what happens when the next workload arrives?

With **What-If Analysis**, I modeled *New-Retro-Quake*: **one hypothetical new VM**, separate from the tiny DOOM VM, placed on the same cluster. The scenario distinguishes **Allocation** (**2 vCPUs, 6 GB RAM, 80 GB disk**) from estimated **Demand** (**600 MHz CPU, 614.4 MB RAM, 8 GB disk**). Provisioned resources are not the same as resources actually needed.

The result is **“The workload will fit”**: a modeled cost of **€27/month** and **Time Remaining greater than one year**. Forecast peaks with the workload added are **39.11 / 48.24 GHz CPU**, **110.01 / 160.79 GB RAM**, and **256.12 GB / 1.98 TB storage** (projected demand / available capacity). This is a **simulation**, not a resource reservation or a performance guarantee.

![VCF Operations What-If: simulating an additional HCI workload](/img/vcf-9-1-what-if.webp)

For a closer look at the model's output, see the [CPU, memory and storage forecast detail](/img/vcf-9-1-what-if-forecast.webp).

**The point of the experiment:** move beyond monitoring to actionable decisions — *reclaim what is unused, rightsize what exists, and plan what comes next*. All estimates depend on **historical data, policies, the Demand/Allocation model and workload behavior**.

## Closing thoughts

An Alpine VM with 512 MB of RAM, an open-source emulator and a game from 1993 were all it took to turn a capacity-management test into a small retrocomputing lab.

**Minimal footprint, minimal effort, excellent value for the exercise.** And seeing DOOM run inside VMware Cloud Foundation is undeniably satisfying.

### References and repositories

- [VMware Holodeck – repository and documentation](https://github.com/vmware/Holodeck)
- [Alpine Linux](https://alpinelinux.org/)
- [DOSBox Staging – source repository](https://github.com/dosbox-staging/dosbox-staging)
- [DOOM v1.9 Shareware – Doomworld /idgames archive](https://legacy.doomworld.com/idgames/idstuff/doom/doom19s)
- [VMware – What's New in Capacity Management for VCF 9.0](https://blogs.vmware.com/cloud-foundation/2025/08/06/whats-new-in-capacity-management-for-vmware-cloud-foundation-9/)
- [VMware – Rightsizing VMs with VMware Cloud Foundation](https://blogs.vmware.com/cloud-foundation/2025/08/12/vmware-cloud-foundation-helps-you-optimally-size-vms-and-keep-your-infrastructure-and-workloads-running-smoothly/)

*Licensing and usage note:* DOSBox Staging is licensed under **GPL-2.0-or-later**; **DOOM Shareware** remains proprietary id Software software and is subject to the terms included with its original distribution. This is a personal, non-commercial lab; this post **does not redistribute game files** or imply that the shareware edition can be freely used in commercial demonstrations. Use of VCF must be covered by the applicable licenses and authorizations.
