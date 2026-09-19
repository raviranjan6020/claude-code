# 02 — SUSE / Rancher: the fully open-source toolkit

**One-line summary:** the best-value, genuinely 100% open-source stack — but it is a
*toolkit*, not a product. SUSE hands you eight excellent components and a documentation
site; integrating them is your job.

This is the most important vendor for us to understand, because **almost every piece is
Apache 2.0 and directly reusable**, and because their weakness (fragmentation) is a
product opportunity.

---

## 1. The component map

| Layer | SUSE/Rancher component | License |
|---|---|---|
| L1–L4 metal | **Metal3** (BMO + Ironic) | Apache 2 |
| L2/L4 alt path | **Elemental** (registration-based onboarding) | Apache 2 |
| L4 OS | **SL Micro** (SLE Micro) — immutable, btrfs snapshots | Proprietary-ish/SUSE |
| L4 image build | **Edge Image Builder (EIB)** | Apache 2 |
| L7 distro | **RKE2** (hardened) / **k3s** (edge) | Apache 2 |
| L7 orchestration | **Rancher Turtles** (CAPI ↔ Rancher bridge) | Apache 2 |
| L8 CNI/LB | Canal/Calico/Cilium, **MetalLB**, kube-vip | Apache 2 |
| L8 storage | **Longhorn** (CNCF), Ceph via Rook | Apache 2 |
| L9/L10 GitOps | **Fleet** | Apache 2 |
| L10 hub | **Rancher Manager** | Apache 2 |
| HCI | **Harvester** (KubeVirt + Longhorn + RKE2) | Apache 2 |
| Security | **NeuVector**, **Kubewarden** (WASM policy) | Apache 2 |
| OS patching | **SUSE Multi-Linux Manager** (Uyuni) | GPL |
| Observability | **SUSE Observability** (StackState) | Proprietary |

---

## 2. Rancher Manager: the hub, and its famous tunnel

Rancher runs on a "local" cluster (usually RKE2 or k3s). Every downstream cluster gets:

- **`cattle-cluster-agent`** — a Deployment that **dials outbound** to Rancher and holds
  open a websocket.
- **`cattle-node-agent`** — a DaemonSet for node-level operations.

The websocket runs **`remotedialer`** (`rancher/remotedialer`, Apache 2) — a reverse
tunnel that lets Rancher initiate connections *through* the spoke's outbound session.
Rancher then acts as a **proxy for the downstream API server**: your `kubectl` talks to
`rancher.example.com/k8s/clusters/c-m-xxxxx`, Rancher authenticates you against its own
RBAC, impersonates you downstream, and forwards over the tunnel.

**Why this matters:** no inbound firewall rules, no VPN, no public API servers, one
auth/audit choke point. This is the same idea Rafay sells as "Zero Trust Kubectl" and
Sidero implements as SideroLink over WireGuard.

**The catch:** the hub becomes a single point of failure for *access* (not for
workloads — spokes keep running), and a traffic concentrator. Rancher's practical
ceiling is commonly cited around **2,000 downstream clusters** per Rancher install
before the tunnel fan-in, websocket churn, and `steve`/`norman` API aggregation layers
become the bottleneck. This is why Fleet exists separately.

**Two ways clusters get into Rancher:**
1. **Imported/registered** — run one `kubectl apply -f <url>` on any existing cluster.
   Dead simple, works with EKS/GKE/AKS/anything. This low-friction on-ramp is a huge
   part of Rancher's adoption.
2. **Provisioned** — Rancher creates the cluster. "Provisioning v2" is built on **CAPI**
   under the hood (`cluster-api-provider-rke2`, plus Rancher's own machine providers
   for vSphere/Harvester/AWS/Azure/DigitalOcean/custom).

**"Custom" clusters** deserve a mention: Rancher gives you a `curl | sh` registration
command with a token; you run it on any Linux box you've already provisioned and it
joins. This deliberately **skips L1–L5 entirely** — and it's how the overwhelming
majority of real Rancher bare-metal clusters are actually built. Customers use
Ansible/Foreman/MAAS/their own kickstart for the OS, then paste a Rancher command.

---

## 3. Elemental: the "registration-first" metal path

Elemental is SUSE's alternative to BMC-driven provisioning, and conceptually it's
closer to what an edge customer actually wants.

```
1. Admin creates a MachineRegistration CR on Rancher
      → produces a registration URL + token
2. Build an installer ISO/image with that URL embedded (via EIB or elemental-operator)
3. Ship/USB/PXE the image to the site; someone powers on the box
4. Elemental agent boots, calls home, registers
      → a MachineInventory CR appears on the hub with full hardware inventory
5. MachineInventorySelectorTemplate + CAPI (cluster-api-provider-elemental)
      selects inventory into a cluster
6. RKE2/k3s installed onto immutable SL Micro; node joins; Rancher imports it
```

CRDs: `MachineRegistration`, `MachineInventory`, `MachineInventorySelector`,
`MachineInventorySelectorTemplate`, `ManagedOSImage`, `ManagedOSVersion`,
`ManagedOSVersionChannel`, `SeedImage`.

**The insight:** this inverts the control flow. Metal3 is *hub-reaches-out* (needs BMC
access, needs network path, needs credentials in the hub). Elemental is
*spoke-reaches-in* (needs only outbound HTTPS). For 5,000 retail stores with no BMC and
no VPN, Elemental wins trivially. For a datacenter with 500 Dells on a management VLAN,
Metal3 wins.

**A serious product needs both paths.** Red Hat learned this too (Metal3 *and*
Assisted Installer).

The OS is **SL Micro**: immutable, read-only root, `transactional-update` with btrfs
snapshots → atomic A/B upgrade and rollback. `ManagedOSImage` drives fleet-wide OS
upgrades as a Fleet bundle.

---

## 4. SUSE Edge: the integrated stack (3.x → 3.5)

SUSE Edge is the *curated bundle* of the above, with tested version combinations. The
current shape:

- **Management cluster**: RKE2 + Rancher + Fleet + Metal3 (BMO + Ironic + `media-server`)
  + **Rancher Turtles** + MetalLB + cert-manager, often bootstrapped by **EIB**.
- **Downstream provisioning**: `BareMetalHost` CRs → CAPM3 → `Metal3MachineTemplate` →
  RKE2 bootstrap/control-plane providers → cluster comes up → **Turtles auto-imports it
  into Rancher** so it appears in the UI alongside everything else.
- **Rancher Turtles** is the important glue: an operator that makes CAPI clusters
  first-class Rancher citizens (`CAPIProvider` CR to install providers, auto-import via
  namespace/cluster labels). It is Rancher's admission that CAPI won, and its path off
  the bespoke provisioning v1 code.
- **Edge Image Builder**: declarative YAML → a customized SL Micro ISO/raw image with
  RKE2/k3s embedded, Helm charts pre-loaded, container images pre-pulled, network config,
  users, certs, and Fleet/Elemental registration baked in. **This is the air-gap answer**:
  the image contains everything, the site needs no registry access on day 1.

**Gaps in this path** (as of the 3.x line): Metal3 support is comparatively young here,
the supported hardware/BMC guidance is narrower than OpenShift's, the management cluster
has fairly specific networking prerequisites (the provisioning network, MetalLB VIPs for
Ironic endpoints), and multi-tenancy across the Metal3 inventory is not really a thing.

---

## 5. Fleet: GitOps built for absurd scale

Fleet is the most under-appreciated project in this whole landscape, and the closest
thing to a purpose-built answer for L9 at fleet scale.

Model:
- **`GitRepo`** on the hub → scanned → produces **`Bundle`** objects (one per path/
  chart in the repo).
- **`Bundle`** → matched against target clusters via `targets` (cluster selectors,
  `ClusterGroup` selectors, label selectors) → produces a **`BundleDeployment`** per
  cluster, placed in that cluster's hub-side namespace.
- The **Fleet agent** on each downstream cluster watches *only its own*
  `BundleDeployment`s and applies them. Pull model, same virtue as OCM's `ManifestWork`.
- **Per-target customization**: the same `Bundle` carries overlays — `helm.values`,
  `valuesFrom`, `kustomize.dir`, `yaml.overlays` — selected per target. **One bundle
  definition, 1,000 site-specific renderings.** This is the answer to "how do I express
  1000 sites without 1000 YAML files" and it's cleaner than Argo's ApplicationSet for
  this shape.
- Supports raw YAML, Kustomize, Helm, and Helm-with-Kustomize-postrender.
- `Bundle` chunking + content resources handle large payloads.

Rancher claims Fleet targets **~1 million clusters**; realistic validated numbers in
public material are in the tens of thousands. Either way it's an order of magnitude
beyond a single Argo CD instance managing per-cluster `Application`s.

**Fleet vs Argo CD vs Sveltos** (this choice matters for us):

| | Fleet | Argo CD | Sveltos |
|---|---|---|---|
| Model | push bundle → spoke agent pulls | hub pushes via stored kubeconfig | hub pushes, classification-driven |
| Scale shape | 10k+ small clusters | 100s of clusters | 1000s |
| Per-cluster variance | first-class overlays | ApplicationSet generators | templated ClusterProfile |
| Needs spoke agent | yes | no | no (agentless) or agent mode |
| Git required | yes | yes | no (can be event/CRD driven) |
| UX maturity | weak UI | best-in-class UI | CLI/CRD only |

---

## 6. Harvester: the VMware-replacement play

Harvester = RKE2 + KubeVirt + Longhorn + kube-vip + Multus/Cilium, packaged as an
installable HCI appliance ISO. It is simultaneously:
- a hyperconverged virtualization platform (VMs with a VMware-ish UI),
- a **CAPI infrastructure provider** (Rancher provisions guest K8s clusters onto it),
- managed from Rancher as just another cluster.

**Strategically this is SUSE's strongest card right now.** Post-Broadcom, every
enterprise with a VMware estate is shopping. Harvester's story — "bare metal in, VMs
and Kubernetes out, one open-source stack, per-node pricing" — lands hard.

Its weaknesses: Longhorn's performance ceiling versus vSAN/Ceph for demanding
workloads, thinner live-migration/DRS-equivalent features, and no real multi-tenant
IaaS control plane (no projects/quotas/VPC networking the way OpenStack has).

---

## 7. Business model

- **Rancher Prime** — the supported build, priced **per managed node** (commonly cited
  in the low hundreds to ~$1k/node/yr range depending on volume and support tier).
- SLES / SL Micro subscriptions separate.
- **Everything is Apache 2 and usable for free.** SUSE monetizes support, hardened
  builds, the curated version matrix, and FIPS/CC certifications — *not* feature
  gating. This is unusually honest and it constrains their revenue per customer.

## 8. Where it hurts

1. **Fragmentation.** Rancher UI, Fleet (different UI, different model), Elemental
   (another set of CRDs), Harvester (another cluster), Metal3 (raw CRDs, no UI),
   Multi-Linux Manager (a whole separate product), NeuVector (another console). A
   customer experiences seven products, not one.
2. **No unified inventory or day-2 rollout engine.** Nothing equivalent to TALM. There
   is no "upgrade these 200 sites in batches of 10 with a canary and automatic abort".
3. **Metal3 path is comparatively immature** in this stack and undocumented at the
   edges; the practical path most customers use is "provision the OS yourself, then
   run the Rancher registration command".
4. **Weak multi-tenancy.** Rancher's Projects are namespace groupings, not a real
   tenancy model with per-tenant metal pools, quotas, chargeback, or isolated control
   planes.
5. **Thin observability/FinOps** without buying SUSE Observability.
6. **Strategic wobble.** Rancher has rewritten its provisioning engine twice (v1 →
   v2 → Turtles/CAPI). Customers notice.

## 9. What we steal

- **`remotedialer`** (Apache 2) — a production-proven reverse tunnel we can embed
  instead of inventing one.
- **Fleet's per-target overlay model** — the cleanest expression of fleet-wide config
  with per-site variance in the ecosystem.
- **Elemental's registration-first flow** — the "phone home" onboarding path.
- **Edge Image Builder's philosophy** — bake everything into the image; air-gap becomes
  a build-time problem instead of a runtime one.
- **RKE2 as our default distro**: CIS-hardened by default, FIPS-capable, single binary,
  SELinux-aware, embedded etcd, Apache 2, no vendor lock. It is the most "enterprise
  RFP-ready" free distro.
- **Harvester/KubeVirt** as the VMware-exit motion.
