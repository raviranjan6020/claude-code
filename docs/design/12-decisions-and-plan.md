# 12 — Decisions & Revised Plan

## Decisions taken (2026-09-19)

| # | Decision | Choice |
|---|---|---|
| D-1 | Beachhead vertical | **Deferred** — build the generic engine; the first design partner picks the vertical |
| D-2 | Go-to-market | **Services-first with a design partner** — paid engagement, tooling stays open source |
| D-3 | Network fabric (L6) | **Verify, never configure.** Host-side LLDP cable-map verification only. No switch write path, ever. Partner/integrate (Netris) when a customer needs the fabric configured. |

These two are one strategy, not two: **the design partner chooses the beachhead, and
pays us to discover it.** That is a legitimate solo-founder path — it buys hardware
access, real failure modes, revenue, and a reference logo, all of which are otherwise
unaffordable.

It also has one specific, well-known failure mode.

> ### ⚠️ The failure mode we are explicitly guarding against
> Services-first + no vertical focus degenerates into **"a consultancy with a side
> repo."** Eighteen months later there is revenue, no product, and a codebase that is
> three client forks with the client names changed.
>
> **The discipline that prevents it, stated as a rule:**
> Every hour of client work must land in **the spine** or in **a pack** (defined below).
> Nothing lands in a client-specific fork. If a client needs something that fits
> neither, it is either (a) refactored until it fits, or (b) delivered as throwaway
> glue we explicitly agree not to maintain. There is no third option.

---

## D-3 in full: why we do not build the network fabric

### What changed
The first draft of doc 09 listed "rack-to-workload in one object" (L6) as a co-equal
differentiator. That was wrong, for two reasons, and correcting it is the most
consequential change in this plan.

**Reason 1 — the niche is occupied.** Netris has spent years building exactly this for
exactly our intended market: a controller + per-switch agents + an XDP-accelerated
multi-tenant VPC gateway (SoftGate), covering NVIDIA/Cumulus and Dell/EdgeCore/SONiC,
now extended to BlueField DPUs for per-GPU tenant isolation, with top-tier neocloud
references. **And Mirantis partners with them** rather than building it into k0rdent.
When the closest architectural analog to our product chooses to partner on this layer,
that is a finding, not an opinion.

**Reason 2 — founder fit.** L6's write path demands BGP/EVPN/VXLAN/MLAG fluency, a
switch-vendor certification matrix, a per-NOS agent, a line-rate data plane, and the
trust of network teams who gatekeep hard. A non-network founder starting here is
starting with their weakest hand against an incumbent's strongest.

### Why L3 (firmware) is the correct wedge for *this* founder

| | **L3 — firmware/BIOS/RAID** | **L6 — fabric write path** |
|---|---|---|
| Nature of the problem | API + data: Redfish, vendor CLIs, a quirks table | Protocol + topology + distributed state |
| Prerequisite expertise | read vendor docs carefully | years of network engineering |
| Feedback loop | minutes, in a lab, on one node | needs a real fabric and a maintenance window |
| Blast radius of a bug | one node won't boot; reflash it | a rack or a fabric goes dark |
| Gatekeeper to get started | none — you own the server | the network team must approve you |
| What it rewards | **persistence and rigour** | deep domain intuition |
| Incumbent | **none** | Netris, well-funded, NVIDIA-aligned |
| Moat shape | a quirks DB that **compounds** per vendor/generation | protocol coverage + certifications |

L3 is the rare layer where the moat is built by *grinding* — every Dell generation,
every iDRAC firmware revision, every Supermicro quirk you encode is permanent value
nobody can shortcut. That is exactly the kind of moat a solo founder with an AI pair
can actually build, and exactly the kind a large vendor won't staff because it's
unglamorous.

### What we keep from L6 (and it is genuinely cheap)

**Host-side LLDP cable-map verification.** During Ironic/discovery-ramdisk inspection,
`lldpd` already reports each NIC's switch neighbour and port. We record it on the
`Machine`, diff it against `spec.fabric`, and surface a condition:

```
  FabricMismatch: eno2 reports leaf-r07-b/Ethernet1/19, inventory expects Ethernet1/12.
  Likely miscabled. (LLDP observed 2026-09-19T11:04Z)
```

No switch credentials. No network-team approval. No protocol drivers. No writes. It is
parsing, roughly two weeks of work, and it catches the most common and most
time-wasting error in datacenter bring-up. The `Machine.spec.fabric` block in doc 10
**stays in the API** — it just becomes *asserted intent we verify and can hand to
someone else*, rather than something we apply.

### The integration path
When a customer needs the fabric actually configured, we export the verified topology
(machine → NIC → switch → port → intended VLAN/LAG) over an API to **Netris**, to their
existing Ansible, or to Nautobot as source of truth. Being the system that *knows the
truth about the physical topology* is a good position; being the system that writes to
their switches is a fight we would lose.

**Revisit trigger:** only if a design partner has no fabric automation, explicitly asks
us to own it, and will fund a network engineer to do it. Not before.

---

## What "stay unfocused" actually means in code

Unfocused at the **pack** level. Ruthlessly focused on the **spine**.

### The spine — identical for every vertical, ~80% of the work

This is what we build regardless of who the first customer turns out to be. Nothing
here is GPU-specific, VM-specific or sovereign-specific:

| Layer | Spine component |
|---|---|
| L1–L2 | `Site` / `Rack` / `Machine` CRDs; BMC drivers; Ironic inspection |
| **L3** | `HardwareProfile` — firmware/BIOS/RAID desired state, drift, attestation |
| L4 | bootc image build + sign + mirror; Redfish virtual-media deploy |
| L5 | `NetworkProfile` — nmstate rendered from IPAM |
| L6 | LLDP cable-map **verification only** (host-side). No switch writes — see D-3 |
| L7 | `ClusterTemplate` / `Cluster` → CAPI → RKE2 (+ k3s, Talos classes) |
| L8 | Cilium, MetalLB, Rook/Longhorn as default addons |
| L9 | `AddonTemplate` / `AddonOverlay` via Sveltos + Flux |
| L10 | OCM registration, audited kubectl tunnel, WireGuard rescue path |
| L11 | `RolloutCampaign` |
| L12 | Air-gap bundle CLI; OCI-artifact distribution for everything |

### Packs — deferred, pluggable, built only when a client pays for one

A pack is a versioned bundle of `AddonTemplate`s + `HardwareProfile` presets +
dashboards + docs. It must be installable on a stock spine with no core changes.
**If building a pack requires changing the spine, the spine's API is wrong — fix the
API, don't special-case the pack.**

| Pack | Contents | Triggered by |
|---|---|---|
| `pack/gpu` | GPU Operator, Network Operator, DCGM, MIG manager, Kueue, GPU-hour metering; `HardwareProfile` presets for H100/H200/GB200 (PCIe ACS, IOMMU, C-states, NUMA/GPU affinity validation) | an AI/neocloud partner |
| `pack/virt` | KubeVirt, CDI, Forklift (VMware migration), storage tuning, VM-centric UI views | a VMware-exit partner |
| `pack/sovereign` | FIPS-mode builds, STIG/CIS profiles, offline CVE feeds, compliance attestation reporting | a gov/regulated partner |
| `pack/telco` | SR-IOV, PTP, NUMA-aware scheduling, PerformanceProfile equivalent | (avoid for now) |

**The test for whether a feature belongs in the spine:** would at least two of the four
packs need it? If yes, spine. If no, pack.

---

## How a services engagement must be structured

The contract determines whether this strategy produces a company or a job. Non-negotiables:

**Must have:**
- **We own the IP.** Client receives a broad, perpetual, irrevocable license to use and
  modify — not ownership. Work-for-hire with IP assignment kills the product; walk away.
- **Explicit right to open-source** anything we write, at our discretion.
- **Named reference + case study rights** (can be time-delayed 6–12 months if they're
  sensitive).
- **Access to their lab/fleet** for development and testing. This is a large part of
  what we're actually being paid in.
- **Fixed scope, fixed deliverable, time-boxed.** Not a retainer that becomes a job.

**Should have:**
- A **support/subscription tail** after the build — this is the transition from services
  revenue to product revenue, and it should be in the first contract, not negotiated
  later.
- A second phase priced as product + support rather than time.

**Red flags to decline:**
- "We need it to work with our existing [bespoke thing]" as the core of the scope.
- Anything where the deliverable is a document or a migration rather than running
  software.
- A client who wants the code but not us — that's a one-time payment and no compounding.

**Pricing frame:** we are replacing a Red Hat subscription that would cost them
$300k–$2M/yr on the same fleet. Price the engagement against that, not against
contractor day rates.

---

## What wins the first engagement (in leverage order)

1. **Publish the landscape research.** The corpus in this repo, cleaned up, is a
   credible *"State of Bare-Metal Kubernetes, 2026"* report. For a solo founder with no
   logo, a genuinely useful public artifact is the cheapest credibility that exists, and
   it generates inbound from exactly the people who have this problem. Cost: a week.
2. **The 30-minute demo, recorded.** Racked-to-cluster in a virtual lab, unattended,
   air-gapped, ending in a signed firmware-compliance report. Cost: milestone M1.
3. **Two narrow technical posts with real numbers** — e.g. measured NCCL bandwidth
   delta from a single wrong BIOS setting, or the cost of firmware drift across a pool.
   Specific numbers travel; opinions don't.
4. **Direct outreach** to GPU cloud providers and mid-size DC operators. They are
   findable, they are not enterprise-procurement-slow, and they know they have this
   problem.

---

## Revised milestones

| # | When | Goal | Output |
|---|---|---|---|
| **M0** | weeks 1–4 | **Credibility + competence.** Build the virtual lab; run exercises 1–5 from doc 11; publish the landscape report. | Lab-in-a-box repo; lab journal; published report |
| **M1** | months 1–3 | **The spine, demoable.** `Site`/`Machine`/`HardwareProfile`/`ClusterTemplate`/`Cluster`; Metal3 virtual-media → bootc → RKE2 → Cilium; one CLI. Single tier. | The recorded 30-minute demo |
| **M2** | months 3–5 | **The differentiator, on real metal.** `forge-firmware` remediation + quirks DB for Dell iDRAC9 + signed attestation. Rent hardware. | Demo on physical servers with the compliance report |
| **M3** | months 4–6 | **Win the engagement.** Outreach on the back of M0–M2. | Signed design-partner contract |
| **M4** | months 6–12 | **Spine hardening + the pack they need.** OCM registration, audited tunnel, `RolloutCampaign`, air-gap bundle, hub/site tier split — plus whichever pack the partner's vertical demands, built *as a pack*. | Running in a customer's DC; support tail begins |

**M3 is deliberately parallel to M2, not after it.** Outreach starts the moment the
virtual-lab demo exists; the physical-metal version closes the deal.

**Still explicitly NOT in scope:** Karmada/cross-cluster scheduling, service mesh, a
marketplace, our own distro/CNI/OS-from-scratch, Windows nodes, telco RAN.

---

## Immediate next action

Build the **lab-in-a-box**: one command → libvirt + `sushy-tools` (emulated Redfish
BMCs *with virtual media*) + Metal3 + CAPM3 + RKE2, so doc 11's exercises 1–5 are cheap
to repeat and to hand to anyone else later.

It is also the first artifact of the product — the spine's dev environment and its CI
substrate are the same thing.
