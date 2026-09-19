# 08 — Open-Source Bill of Materials

Candidate components for our stack, with licenses (the thing that decides whether we
can build a business on them) and a verdict.

**License rule of thumb for a commercial product:** Apache-2.0 / MIT / BSD = safe to
embed and redistribute. **AGPLv3 = talk to a lawyer before embedding or network-serving
it.** MPL-2.0 = fine, file-level copyleft. BSL = time-delayed, check the grant.

---

## L1–L4 — Metal

| Component | License | Verdict |
|---|---|---|
| **Metal3** (baremetal-operator) | Apache 2 | ✅ **Core dependency.** |
| **Ironic** + IPA | Apache 2 | ✅ via Metal3. Heavy, but the BMC matrix is the moat. |
| **CAPM3** + `ip-address-manager` | Apache 2 | ✅ CAPI glue + static IPAM for metal. |
| **Tinkerbell** (Smee/Tink/Rufio/HookOS) | Apache 2 | 🟡 Keep as the light alternative / for whitebox gear. |
| **bmclib** | Apache 2 | ✅ Multi-vendor BMC abstraction (Go). Use directly for L3. |
| **gofish** | BSD-3 | ✅ Redfish client (Go). |
| **sushy** / **sushy-tools** | Apache 2 | ✅ `sushy-tools` emulates Redfish BMCs → **our virtual lab.** |
| **VirtualBMC** | Apache 2 | ✅ IPMI emulation for libvirt VMs → lab. |
| **openDCIM / NetBox / Nautobot** | Apache 2 | ✅ **Nautobot** (Apache 2) as inventory/IPAM SoT + its job engine. |
| MAAS | **AGPLv3** | ⚠️ Great tool, licensing risk. Integrate-with, don't embed. |
| OpenBMC | Apache 2 | 🟡 Only if we ever do ODM/whitebox firmware. |

## L4 — OS

| Component | License | Verdict |
|---|---|---|
| **bootc** / bootc-image-builder | Apache 2 | ✅ **Primary OS delivery mechanism.** OS as an OCI image. |
| **Talos Linux** | MPL-2.0 | ✅ Hardened opt-in class. API-only, no shell. |
| **Kairos** | Apache 2 | ✅ Alternative: turn any distro immutable + A/B + trusted boot. |
| **Flatcar Container Linux** | Apache 2 | 🟡 Mature, A/B, sysext for drivers. CNCF incubating. |
| **Ignition / Butane** | Apache 2 | ✅ First-boot config for image-based OS. |
| **cloud-init** | Apache 2/GPL dual | ✅ Universal fallback. |
| **nmstate** | Apache 2 | ✅ **Declarative host networking.** Use for L5. |
| **osbuild / image-builder** | Apache 2 | ✅ Image pipeline. |
| **mkosi** | LGPL-2.1 | 🟡 Alternative image builder. |

## L7 — Kubernetes

| Component | License | Verdict |
|---|---|---|
| **Cluster API** + providers | Apache 2 | ✅ The engine (hidden behind our API). |
| **RKE2** | Apache 2 | ✅ **Default distro.** CIS-hardened, FIPS-capable, SELinux, embedded etcd. |
| **k3s** | Apache 2 | ✅ Edge/SNO class. |
| **k0s** | Apache 2 | 🟡 Alternative; pairs with k0smotron. |
| **Talos** (as distro) | MPL-2.0 | ✅ Hardened class. |
| **Kamaji** | Apache 2 | ✅ **Hosted control planes** — most mature OSS option. |
| **k0smotron** | Apache 2 | 🟡 Alternative HCP, CAPI-native. |
| **kube-vip** | Apache 2 | ✅ Control-plane VIP on metal. |
| **etcd-druid** | Apache 2 | 🟡 Gardener's etcd operator — excellent for hosted CP etcd backup. |

## L8 — Network & Storage

| Component | License | Verdict |
|---|---|---|
| **Cilium** | Apache 2 | ✅ CNI + eBPF + BGP control plane + egress GW + cluster mesh + Hubble. One dependency, five features. |
| **MetalLB** | Apache 2 | ✅ Simple L2/BGP LB when Cilium BGP is overkill. |
| **Multus** | Apache 2 | ✅ Secondary NICs (telco/AI). |
| **SR-IOV Network Operator** | Apache 2 | ✅ For SR-IOV/RDMA. |
| **kube-ovn** | Apache 2 | 🟡 If we need real VPC/subnet/QoS multi-tenancy (IaaS-style). |
| **Rook / Ceph** | Apache 2 / LGPL | ✅ DC-scale storage. |
| **Longhorn** | Apache 2 | ✅ Edge/small-cluster storage (CNCF). |
| **OpenEBS / Mayastor** | Apache 2 | 🟡 NVMe-oF performance tier. |
| **TopoLVM**, **local-path-provisioner** | Apache 2 | ✅ Local disks, SNO. |

## L9 — Add-on / state distribution

| Component | License | Verdict |
|---|---|---|
| **Sveltos** | Apache 2 | ✅ **Strong candidate for the addon engine.** Classification + event-driven + CAPI-native. |
| **Flux** | Apache 2 | ✅ Git/OCI source + Helm/Kustomize reconciler. Pairs with Sveltos. |
| **Argo CD** | Apache 2 | 🟡 Best UI; scale ceiling per instance; support as an *option* for customers who already use it. |
| **Fleet** | Apache 2 | 🟡 Best per-target overlay model; couples us to Rancher-shaped thinking. |
| **Helm**, **Kustomize** | Apache 2 | ✅ Obviously. |
| **ORAS** | Apache 2 | ✅ Ship anything as an OCI artifact. |

## L10 — Hub

| Component | License | Verdict |
|---|---|---|
| **Open Cluster Management** (`ocm`) | Apache 2 | ✅ **Registration + ManifestWork + Placement + addon-framework.** Saves ~a year. |
| **rancher/remotedialer** | Apache 2 | ✅ Reverse tunnel library, production-proven. |
| **konnectivity** | Apache 2 | 🟡 K8s-native alternative tunnel. |
| **WireGuard** / **Netbird** / **Headscale** | GPL-2/BSD / MIT | 🟡 Node-level rescue path (SideroLink-style). |
| **kubebuilder / controller-runtime** | Apache 2 | ✅ Our CRDs. |
| **Karmada** | Apache 2 | 🟡 Later — cross-cluster scheduling/failover. |
| **Backstage** | Apache 2 | 🟡 If we want an IDP/self-service portal without building one. |

## L9/L11 — Security, policy, identity

| Component | License | Verdict |
|---|---|---|
| **cert-manager** | Apache 2 | ✅ Mandatory. |
| **Kyverno** | Apache 2 | ✅ Policy (easier for customers than Rego). |
| **Kubewarden** | Apache 2 | 🟡 WASM policy alternative. |
| **SPIRE / SPIFFE** | Apache 2 | 🟡 Unified workload+node identity across the fleet. Powerful, adds complexity. |
| **cosign / sigstore policy-controller** | Apache 2 | ✅ Supply chain; must work offline. |
| **Trivy / Trivy Operator** | Apache 2 | ✅ Scanning + SBOM. |
| **Falco** | Apache 2 | ✅ Runtime detection (CNCF graduated). |
| **NeuVector** | Apache 2 | 🟡 Fuller container security suite. |
| **OpenBao** (or Vault ≤1.14) | MPL-2 / **BSL** | ✅ **OpenBao** — the BSL relicense makes Vault risky to embed. |
| **External Secrets Operator** | Apache 2 | ✅ Secret plumbing to spokes. |
| **Dex** / **Keycloak** | Apache 2 / Apache 2 | ✅ OIDC/SAML broker for hub SSO. |

## L10 — Observability & FinOps

| Component | License | Verdict |
|---|---|---|
| **OpenTelemetry** collector | Apache 2 | ✅ Spoke-side collection. |
| **Prometheus** / **prometheus-agent** | Apache 2 | ✅ |
| **VictoriaMetrics** | Apache 2 | ✅ **Best cost/perf for fleet-scale metrics.** (vmagent → vmcluster) |
| **Mimir** / **Thanos** | AGPLv3 / Apache 2 | ⚠️ Mimir is AGPL — prefer Thanos or VictoriaMetrics. |
| **Loki** | AGPLv3 | ⚠️ Same concern; consider **VictoriaLogs** (Apache 2) or Vector+object store. |
| **Vector** | MPL-2 | ✅ Log shipping. |
| **Grafana** | AGPLv3 | ⚠️ Fine to *deploy alongside*; do not embed/fork into our product. |
| **Perses** | Apache 2 | ✅ Dashboards-as-code, Apache 2 — the embeddable Grafana alternative. |
| **OpenCost** | Apache 2 | ✅ Cost allocation (CNCF). |
| **Robusta / kube-prometheus-stack** | MIT / Apache 2 | 🟡 |

## GPU / AI

| Component | License | Verdict |
|---|---|---|
| **NVIDIA GPU Operator** | Apache 2 | ✅ Drivers, toolkit, device plugin, DCGM, MIG manager. |
| **NVIDIA Network Operator** | Apache 2 | ✅ RDMA/GPUDirect/RoCE. |
| **DCGM Exporter** | Apache 2 | ✅ Per-GPU telemetry → metering. |
| **Kueue** | Apache 2 | ✅ Batch quota/fair-share (K8s SIG). |
| **Volcano** | Apache 2 | 🟡 Gang scheduling (CNCF). |
| **HAMi** | Apache 2 | 🟡 GPU sharing/vGPU (CNCF sandbox). |
| **Dynamic Resource Allocation (DRA)** | K8s core | ✅ The future of GPU/device scheduling. Bet on it. |
| **KubeRay**, **Slinky/Slurm-on-K8s** | Apache 2 | 🟡 Workload layer, ICP-dependent. |

## Virtualization (the VMware-exit motion)

| Component | License | Verdict |
|---|---|---|
| **KubeVirt** | Apache 2 | ✅ VMs as pods (CNCF incubating). |
| **CDI** (Containerized Data Importer) | Apache 2 | ✅ Disk image import/upload. |
| **Harvester** | Apache 2 | 🟡 Whole HCI appliance — use as reference or as a supported "class". |
| **virtctl / KubeVirt Manager** | Apache 2 | ✅ |
| **Forklift** | Apache 2 | ✅ **VMware → KubeVirt migration tooling.** This is the wedge. |

## Registry / air gap

| Component | License | Verdict |
|---|---|---|
| **Zot** | Apache 2 | ✅ Tiny OCI-native registry — perfect regional/site mirror. |
| **Harbor** | Apache 2 | ✅ When the customer wants RBAC, replication, scanning, quotas (CNCF graduated). |
| **Spegel** | MIT | ✅ P2P image sharing *within* a cluster — big win on thin links. |
| **Dragonfly** | Apache 2 | 🟡 P2P distribution at larger scale (CNCF). |

## Backup / DR

| Component | License | Verdict |
|---|---|---|
| **Velero** | Apache 2 | ✅ Cluster backup + CSI snapshots. |
| **etcd snapshot tooling** | Apache 2 | ✅ Built into RKE2/k3s; wrap it. |
| **Kanister** | Apache 2 | 🟡 App-consistent backups. |

---

## Rough count

A credible product here is **~35–45 open-source components** integrated, version-matrixed,
tested together, packaged for air gap, and wrapped in one API + one UI + one support
contract.

**That integration is the product.** It is also why Red Hat charges what it charges,
and why SUSE — who ships the same components without fully integrating them — charges
a tenth as much.
