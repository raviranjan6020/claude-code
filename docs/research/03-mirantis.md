# 03 — Mirantis: Container Cloud → k0s → k0rdent

**One-line summary:** the best *architecture* in the market and the weakest *product*.
k0rdent is, structurally, almost exactly the thing we want to build — which is both a
gift (read their code) and a warning (someone with more resources is already here).

---

## 1. Lineage matters

Mirantis's history explains everything about the current product:

- **2010s**: the largest pure-play OpenStack distributor (Mirantis OpenStack, Fuel —
  a bare-metal OpenStack deployer). **They have deeper real bare-metal operational scars
  than anyone except Red Hat.**
- **2019**: acquired **Docker Enterprise** → MKE (Mirantis Kubernetes Engine), MSR
  (registry), MCR (runtime). Inherited a Swarm-era codebase and an enterprise customer
  list.
- **2020**: acquired **Lens** (the desktop K8s IDE) — a distribution channel with
  millions of developers they never fully monetized.
- **2021+**: built **k0s** — a single-binary, zero-dependency Kubernetes distro.
- **2022+**: **Mirantis Container Cloud** + **MOSK** (OpenStack on Kubernetes).
- **2025**: **k0rdent**, open sourced, Apache 2 — the strategic reset.
- Also picked up Banzai Cloud's Pipeline assets and amazee.io (Lagoon PaaS).

The company is **services-led**. Mirantis sells outcomes and people; the products are
often the vehicle. That's why architecture quality outruns product polish.

---

## 2. Mirantis Container Cloud — the explicit three-tier hub-spoke

This is the most literal implementation of "hub and spoke" in the market, and worth
studying because it's the only one that introduces an explicit **regional tier**:

```
 Management Cluster  (the global hub: IAM/Keycloak, UI, release controller,
        │             provider controllers, the "single pane")
        │
        ├── Regional Cluster (per-datacenter / per-region / per-provider)
        │        │   runs the actual provisioning machinery locally:
        │        │   Ironic, DHCP/PXE, image cache, LCM controllers
        │        │
        │        ├── Child Cluster (workload)
        │        ├── Child Cluster
        │        └── Child Cluster
        │
        └── Regional Cluster (another DC, another cloud)
```

**Why the regional tier is the right idea:** bare-metal provisioning is *latency- and
L2-sensitive*. PXE/DHCP needs to be near the servers. Image transfer of a multi-GB OS
image from a global hub to 40 datacenters over WAN is a disaster. Ironic conductors
want to be close to the BMCs. So you push a **provisioning satellite** into each site
and keep only *intent + identity + inventory* in the global hub.

Red Hat has no equivalent (their hub does everything, which is why huge ZTP
deployments get painful). Rancher has no equivalent. **This is an architectural idea
worth adopting as a first-class concept in our design.**

Container Cloud's bare metal provider: Ironic (+ their own `BareMetalHost`-style CRDs,
`BareMetalHostProfile` for disk/partition layout, `IpamHost`/`Subnet`/`L2Template` for
a genuinely declarative L5 network model), Ceph via Rook, LCM controller + agent per
node for ordered upgrades.

**`L2Template` deserves a callout** — a reusable template describing bonds, VLANs,
bridges, and addresses, rendered per host from IPAM. It's the most complete declarative
host-networking model any of these vendors shipped. Most products bolt nmstate on as an
afterthought.

**MOSK** runs full OpenStack (Nova/Neutron/Cinder/Octavia) *as workloads on Kubernetes*
on bare metal — the "we need real IaaS with VMs, VPCs, floating IPs and quotas, not
just KubeVirt" answer. Relevant if our ICP is a sovereign/regional cloud provider, who
almost always needs VM-IaaS semantics, not just containers.

---

## 3. k0s and k0smotron

**k0s**: single static binary, no host dependencies (no need for a specific systemd,
no packages), embeds etcd or can use kine→Postgres/MySQL, ships its own containerd.
Apache 2. Very attractive for heterogeneous/edge fleets where you can't guarantee the
host distro. Comparable to k3s; a bit more "vanilla upstream" in behaviour.

**k0smotron**: a Kubernetes operator that runs **k0s control planes as pods** in a
management cluster, and is a first-class **CAPI control-plane + bootstrap provider**.

So: `Cluster` (CAPI) + `K0smotronControlPlane` (pods on hub) + any infra provider for
workers (including bare metal) = a cluster whose control plane costs you zero dedicated
servers. Control plane upgrade = change the image tag on a StatefulSet.

This is Mirantis's answer to HyperShift, and unlike HyperShift it isn't OpenShift-only.
Compare with **Kamaji** (Clastix) which does the same for vanilla kubeadm-compatible
control planes and is more mature/adopted.

---

## 4. k0rdent — study this closely

Launched 2025 as an open-source "**Distributed Container Management Environment**"
(their coinage; read it as "fleet platform"). Three pillars:

### KCM — k0rdent Cluster Manager
Cluster lifecycle. Built on **Cluster API** + k0s + a pluggable set of CAPI providers.

Core CRDs:
- **`ClusterTemplate`** — a reusable, versioned, parameterized cluster definition
  (which infra provider, which K8s version, which CP provider, which defaults).
  Essentially a productized `ClusterClass`.
- **`ClusterDeployment`** — an instance of a template with values + a credential.
  *"Give me template `aws-standalone-cp-1-2-0` with these 3 values."*
- **`Credential`** — the infra credential, decoupled from the template so the same
  template is reused across tenants/regions.
- **`ServiceTemplate`** — a versioned addon (a Helm chart + values contract).
- **`MultiClusterService`** — "apply these `ServiceTemplate`s to every cluster matching
  this selector." Fleet-wide addon targeting.
- Templates are distributed as **OCI artifacts in a registry** — so air-gap is the same
  problem as mirroring container images. Clean.

### KSM — k0rdent State Manager
Add-on and runtime state distribution. Provider-based; the default provider is
**Flux + Sveltos**.

**Sveltos** (`projectsveltos`, Apache 2) is worth its own study:
- `ClusterProfile` / `Profile` — "clusters matching this selector get these Helm
  charts / kustomizations / raw YAML, in this order, with these values".
- **Cluster classification** — `ClusterHealthCheck`, `Classifier` — clusters
  auto-labelled based on their *actual* state (K8s version, installed CRDs, resources),
  and profiles target those labels. Self-organizing fleet.
- **Event-driven add-ons** — `EventSource` + `EventTrigger`: "when a Service of type X
  appears in any cluster, deploy Y". Genuinely novel; nothing in Argo/Flux does this.
- **Templating with cross-cluster data** — a profile can template values from resources
  in the management cluster or the target cluster.
- Agentless or agent (drift detection) mode.
- Handles dependencies/ordering between profiles.

**For our design, Sveltos is a strong candidate for L9** — more purpose-built for
"fleet of clusters" than Argo CD, lighter than Fleet, and CAPI-native.

### KOF — k0rdent Observability and FinOps
OpenTelemetry collectors + Prometheus/VictoriaMetrics + Grafana + **OpenCost**,
deployed as ServiceTemplates, rolling up to regional/central storage. Cost visibility
per cluster/tenant.

**Tested providers**: AWS EC2/EKS, Azure/AKS, vSphere, OpenStack, Docker (for dev),
plus "bring your own CAPI provider". **Bare metal is notably not the headline** — you
can wire CAPM3 or CAPT in, but it is not the polished path. *That's a gap.*

---

## 5. Business model

- **k0rdent** free/Apache 2; **k0rdent Enterprise** = supported build, hardened,
  validated provider matrix, LTS.
- MKE / MSR / MOSK / Container Cloud: per-node subscriptions.
- Heavy professional services attach. Mirantis will happily run it for you (ops-as-a-
  service), which is where much of the revenue lives.

## 6. Where it hurts

1. **Product polish and UX.** k0rdent is CRDs and a thin UI. Compared to Rafay or
   Spectro it feels like a toolkit for people who already love CAPI.
2. **Bare metal is the weak leg** of k0rdent specifically — ironic, given Container
   Cloud's strength. The strategic energy went to "any cloud + vSphere + OpenStack".
3. **Brand damage/fatigue.** Mirantis has pivoted publicly several times (OpenStack →
   Docker Enterprise → k0s → k0rdent). Enterprise buyers price that in.
4. **Two overlapping product lines** (Container Cloud/MKE vs k0rdent) with an unclear
   migration story.
5. **Small ecosystem.** Sveltos, k0s, k0smotron are all excellent but low-adoption;
   hiring for them is hard, which enterprises care about.

## 7. What we steal

- **The three-tier management → regional → child topology.** Adopt it explicitly.
- **`ClusterTemplate` + `ClusterDeployment` + `Credential` separation.** This is a
  better product API than raw `ClusterClass`, and it's the right shape for multi-tenant
  self-service ("here is a catalog of cluster flavours; pick one").
- **`ServiceTemplate` + `MultiClusterService`.** Versioned addon catalog + selector-based
  fleet targeting.
- **Templates as OCI artifacts.** Everything — cluster templates, addons, OS images —
  distributed through a registry makes air-gap one problem instead of five.
- **Sveltos** as the state engine (or at minimum its classification + event-driven ideas).
- **`L2Template`/`IpamHost`/`Subnet`** from Container Cloud as the model for declarative
  host networking + IPAM.
- **k0smotron/Kamaji** for the hosted-control-plane tier.
