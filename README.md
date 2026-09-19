# Bare-Metal → Kubernetes, Hub-and-Spoke: Research & Design

Working notes for a B2B platform that turns racked servers into multi-tenant Kubernetes
(and VMs), managed as a fleet from a central hub, connected or air-gapped.

**Status:** research phase. No product code yet, by design.

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
| [10 — Reference architecture v0](docs/design/10-our-architecture-v0.md) | Topology, our CRDs, the three engines we'd actually write, MVP milestones, licensing |
| [11 — Lab & open questions](docs/design/11-lab-and-open-questions.md) | How to run all of this with zero hardware, 15 lab exercises, and the 8 decisions that are yours |

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
