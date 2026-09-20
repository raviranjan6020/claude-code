# Ingot

A B2B platform that turns racked servers into multi-tenant Kubernetes (and VMs),
managed as a fleet from a central hub, connected or air-gapped.

*An ingot is metal that has been refined, assayed, and cast to a known standard —
which is what this does to a rack of unknown servers.*

**Status:** research complete; starting the platform-layer build.
Current plan → [13 — P2 build plan](docs/design/13-p2-build-plan.md).

## Contents

### Research — understanding the field
| Doc | What it covers |
|---|---|
| [00 — Problem decomposition](docs/research/00-problem-decomposition.md) | The 13 layers every vendor is a subset of. The scoring rubric. **Read first.** |
| [01 — Red Hat](docs/research/01-redhat.md) | OpenShift + RHACM + Metal3 + Assisted Installer + ZTP + TALM + HyperShift |
| [02 — SUSE / Rancher](docs/research/02-suse-rancher.md) | Rancher + RKE2 + Elemental + Fleet + Harvester + SUSE Edge + Turtles |
| [03 — Mirantis](docs/research/03-mirantis.md) | Container Cloud's 3-tier model, k0s, k0smotron, **k0rdent**, Sveltos |
| [04 — Rafay](docs/research/04-rafay.md) | SaaS control plane, zero-trust kubectl, Blueprints, the GPU PaaS pivot |
| [05 — vCluster / Loft](docs/research/05-vcluster.md) | Virtual clusters, the syncer, vNode, and where density fits |
| [06 — The rest of the field](docs/research/06-other-players.md) | Spectro Cloud, Sidero/Omni, Canonical MAAS, Tinkerbell, Gardener, Kubermatic, Platform9, Nutanix, Arc/EKS-A/GDC, NVIDIA BCM — plus a provisioning-engine comparison table |
| [07 — Cross-cutting patterns](docs/research/07-cross-cutting-patterns.md) | The 10 real design decisions: connectivity, CP placement, tiering, OS model, CAPI-or-not, templating, day-2, air gap, scale limits, business models |
| [08 — OSS bill of materials](docs/research/08-oss-building-blocks.md) | ~60 candidate components with licenses and verdicts |
| [09 — Market & whitespace](docs/research/09-market-whitespace.md) | Competitive scorecard, the 4 gaps, ICP analysis, risks, the pitch |

### Design — our own product
| Doc | What it covers |
|---|---|
| [10 — Reference architecture v0](docs/design/10-our-architecture-v0.md) | Topology, our CRDs, the three engines we'd actually write, licensing |
| [11 — Lab & open questions](docs/design/11-lab-and-open-questions.md) | Zero-hardware lab (libvirt + sushy-tools), 15 graded exercises — now the opening of P1 |
| [12 — Decisions](docs/design/12-decisions-and-plan.md) | D-1…D-6, and why each was taken |
| [13 — **P2 build plan**](docs/design/13-p2-build-plan.md) | **The executable plan.** Done test, architecture, scope + NOT-list, 6 phases, cost |

## The short version

**Everyone is a subset of the same 13 layers.** Red Hat covers nearly all of them and
prices itself out of half the market. SUSE/Rancher ships excellent open-source parts
without integrating them. Mirantis/k0rdent has the best architecture and the thinnest
product. Rafay has the best control plane and no metal at all. Spectro Cloud is the
closest competitor. Sidero has the best engineering and the narrowest scope.

**Three layers are empty across the entire industry:**
1. **Firmware / BIOS / RAID desired state** (L3) — everyone punts to vendor tools
2. **Network fabric co-provisioning** (L6) — everyone punts to the network team
3. **Day-2 rollout campaigns outside OpenShift** (L11) — only Red Hat's TALM exists

Those three, plus table-stakes tenancy and audited access, is the product.

## Decisions

| | Decision |
|---|---|
| **D-1** | **Beachhead deferred** — build the generic engine; the first design partner picks the vertical |
| **D-2** | **GTM: services-first** with a design partner; tooling stays open source |
| **D-3** | **Network fabric: verify, never configure.** Host-side LLDP cable-map verification only. Netris owns the write path and Mirantis partners with them rather than building it — that is the market answering the question |
| **D-4** | **Name: Ingot.** `github.com/<org>/ingot`, API group `ingot.sh/v1alpha1` |
| **D-5** | **P2 (platform) first, then P1 (metal + firmware)** — reverses the earlier recommendation; risks accepted and mitigated by a hard 10-week timebox |
| **D-6** | **Tiers: global / regional / site.** Deploy two, model three. One binary, `--tier=` selects controller sets, so collapsing is a flag not a rewrite |

Structurally: a **spine** (everything below the vertical, ~80% of the work) plus
**packs** (`gpu`, `virt`, `sovereign`) built only when a client pays for one.
Guard rail: every hour of client work lands in the spine or a pack — never a client fork.
