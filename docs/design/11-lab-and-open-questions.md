# 11 — Learning Lab + The Questions We Must Answer

## Part 1 — Build the lab before building the product

You said you're not in a hurry to write code. Good. But **reading docs about Ironic is
nothing like watching a `BareMetalHost` get stuck in `inspecting` for 40 minutes.**
Everything in this repo is secondhand until you've run it.

The good news: you can run *all* of this with zero physical hardware.

### The trick: emulated BMCs

| Tool | Emulates | Use for |
|---|---|---|
| **`sushy-tools`** (`sushy-emulator`) | **Redfish**, incl. virtual media | Metal3, OpenShift, SUSE Edge, anything Redfish |
| **VirtualBMC** (`vbmc`) | IPMI 2.0 | Tinkerbell/Rufio, older paths |
| **`libvirt` + KVM** | the "servers" | everything |
| **`virtualbmc-for-vsphere`** | IPMI over vSphere | if you have ESXi lying around |

A libvirt VM + sushy-tools gives you a machine that PXE-boots, honours Redfish power
commands, and **mounts virtual media** — which is enough to exercise 90% of Metal3.

### Hardware budget, in order

1. **Phase 0 — one workstation.** 64GB RAM (128 is much better), 12+ cores, 2TB NVMe.
   Run libvirt + sushy-tools + 6–8 VMs. Cost: a used Threadripper/EPYC box, or your
   existing machine.
2. **Phase 1 — rented metal.** Hetzner (cheapest in EU, has IPMI on some lines), OVH,
   **Latitude.sh** or **Equinix Metal** (real Redfish + API, and Equinix's heritage
   *is* Tinkerbell). Rent 3 servers for a week at a time to validate real BMC quirks.
   Cost: low hundreds of dollars per experiment.
3. **Phase 2 — owned gear.** 2–3 used Dell R640/R650 or Supermicro off eBay
   (~$500–1500 each) — you need **real iDRAC/IPMI quirks** to build the quirks DB, and
   you cannot fake those.
4. **Phase 3 — vendor lab access.** Dell/HPE/Supermicro partner programs and the
   NVIDIA Inception / NCP programs give remote lab access to current-gen gear. Apply
   early; approval is slow and free.

### Lab exercise list (in order, with what to learn)

| # | Exercise | What you're actually learning |
|---|---|---|
| 1 | libvirt + sushy-tools: 4 VMs with Redfish BMCs | the substrate |
| 2 | Metal3 standalone: `BareMetalHost` → inspect → provision Ubuntu | Ironic's state machine and where it hangs |
| 3 | Add CAPM3 + CAPI + RKE2 providers → a real cluster from metal | how CAPI actually composes; how bad the errors are |
| 4 | Break it deliberately: wrong BMC creds, wrong boot MAC, full disk, mid-deploy power loss | **the failure modes are the product** |
| 5 | Tinkerbell: same 4 VMs, `Hardware`/`Template`/`Workflow` | the alternative model; why its simplicity is seductive |
| 6 | Talos + Omni (free tier) | what best-in-class UX feels like; SideroLink |
| 7 | Rancher + Fleet + Elemental | registration-first onboarding; bundle overlays |
| 8 | SUSE Edge full quickstart (management cluster + Metal3 + Turtles) | how an integrated stack is *packaged* |
| 9 | k0rdent: `ClusterTemplate` → `ClusterDeployment` → `MultiClusterService` | the closest thing to our intended API |
| 10 | OCM: hub + 2 spokes, `ManifestWork`, `Placement`, an addon | the hub substrate we'd build on |
| 11 | OpenShift ZTP with SNO in VMs (if you can get a dev sub) | the gold standard; TALM semantics |
| 12 | Kamaji: 5 hosted control planes + metal workers | HCP economics, hands-on |
| 13 | bootc: build a custom OS image, push to Zot, `bootc upgrade`, roll back | the OS supply chain bet |
| 14 | Air gap: disconnect the lab, mirror everything, provision a cluster | the hardest thing to retrofit |
| 15 | On rented metal: Dell iDRAC — read/write BIOS attrs via Redfish, flash firmware | **the moat.** Where the quirks live. |

Keep a **lab journal** in `docs/lab/` — one file per exercise: what you did, what broke,
how long it took, what the error message was, and what the product *should* have said
instead. That journal is your requirements document, and it will be more valuable than
anything in this repo.

---

## Part 2 — Questions only you can answer

These fork the design. I've given my recommendation for each, but they're yours.

### Q1. Which beachhead — GPU/AI infrastructure, or VMware replacement?
Both use the same engine, but they imply different *first* features, different demos,
different design partners, and a different first hire.
- **GPU/AI:** first features = firmware baselining, RoCE fabric, GPU metering,
  multi-tenant isolation with secure erase. Faster money, hotter competition.
- **VMware exit:** first features = KubeVirt + Forklift migration + storage + a VM-ish
  UI. Bigger TAM, slower, and "cheap" is the battlefield.

*My recommendation:* **GPU/AI first**, because Gap 1 (firmware) is most acutely painful
there, the buyers move fast, and the VMware-exit door can open later on the same engine.

### Q2. Geography and go-to-market. India-first, or US/EU-first?
This changes pricing, compliance, and sales motion dramatically. India has a real,
funded, underserved sovereign-AI and GPU build-out, it's where you can get in front of
buyers, and local SIs are hungry for a stack they can resell. But ACVs are lower and
procurement is relationship-driven. US/EU pays 3–5× more per deal and is far harder to
reach solo.

*My recommendation:* **build for global, sell India-first** (design partners, references,
revenue), then use those references to sell into EU sovereign programs. Do not build
India-specific product.

### Q3. Open core, or closed?
*My recommendation:* **open core, Apache 2.** A solo founder cannot buy distribution.
The OSS core *is* the marketing budget. Draw the line at "one team, one site, one
tenant" (free) vs "an organisation" (paid) — never cripple the engine.

### Q4. How far do we really go on L6 (network fabric)?
It's the biggest differentiator and the biggest risk. Network teams are conservative and
will not let a new vendor write to their switches.
*My recommendation:* **three stages.** (a) read-only: ingest LLDP + port state, verify
the cable map, show the diff — pure value, zero risk, wins the network team over;
(b) dry-run: generate the config they'd apply; (c) apply, opt-in per switch. Stage (a)
alone is already a feature nobody else has.

### Q5. Do we ship our own OS image, or only consume others'?
*My recommendation:* **ship a bootc-based image** (Ubuntu and RHEL-derived variants) as
the default, plus support Talos as a hardened class and "bring your own image" for the
stubborn. You need to own the image to guarantee the driver/firmware matrix — which is
the whole promise.

### Q6. Do we build on OCM, or write our own hub?
*My recommendation:* **build on OCM.** Registration, mTLS identity, per-cluster
namespaces, `ManifestWork`, `Placement`, and the addon framework are a year of work,
Apache 2, and battle-tested at Red Hat scale. Wrap it; don't expose it. Revisit only if
its object model becomes a ceiling.

### Q7. What's the first thing you *sell*, before the platform exists?
Realistically a solo founder gets to revenue via **services + a design partner**: "I'll
build and run your GPU cluster's provisioning stack, and the tooling I build stays open
source." That funds the product and buys you the hardware access and the failure modes
you can't get any other way.

### Q8. Name, domain, and whether you incorporate now.
Genuinely not urgent, but the name shows up in every CRD group, every binary, and every
import path. Changing it after 6 months is annoying. Pick something boring and
available.

---

## Part 3 — What I'd suggest we do next, in this order

1. **You pick Q1 and Q2.** Everything downstream depends on them.
2. **I build the lab-in-a-box**: scripts/compose to stand up libvirt + sushy-tools +
   Metal3 + CAPM3 + RKE2 so exercise 1–3 is one command. (~a day of work for me, and
   it makes everything after it faster.)
3. **We run exercises 1–5** and keep the journal. Expect surprises that invalidate parts
   of doc 10. That's the point.
4. **We write the `Machine` / `HardwareProfile` API for real**, informed by the journal,
   and validate it by hand against a rented Dell.
5. **Then** we decide what the first 3 months of code is.

Ordering matters: **do not write product code until exercise 15 is done.** The quirks
you find flashing a real iDRAC will reshape the `HardwareProfile` API, and everything
else depends on that API being right.
