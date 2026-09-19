# 04 — Rafay: the SaaS control plane (and the GPU pivot)

**One-line summary:** the best *control-plane product* in the space, with almost no
bare-metal depth. They start where Red Hat/SUSE stop. Closed source. If we build a
better hub, Rafay is our closest commercial comparable — and their GPU pivot tells us
where the money currently is.

---

## 1. What Rafay actually is

Rafay is a **multi-tenant SaaS control plane for Kubernetes fleets**, also available
self-hosted and air-gapped. It is explicitly *not* a Kubernetes distribution company
(though they ship MKS), and until recently was not a bare-metal company at all.

Tenancy model — and this is the part most engineering-led competitors get wrong:

```
Organization (customer)
  └── Project  (tenant / team / business unit / end-customer of an MSP)
        ├── Clusters        (imported OR provisioned)
        ├── Blueprints      (addon stacks)
        ├── Workloads / GitOps pipelines
        ├── Users, Groups, Roles  (SSO/OIDC/SAML mapped)
        ├── Secrets, Cost, Audit
        └── Environments (self-service templates)
```

**Projects are a hard RBAC + visibility + cost boundary.** For an MSP or a GPU cloud
reselling to end customers, this is the product. Rancher's "Projects" (namespace
groupings) and Red Hat's `ManagedClusterSet` are both weaker.

---

## 2. Connectivity: Zero Trust Kubectl Access

Every managed cluster runs a Rafay agent (relay/connector) that establishes a
**persistent outbound TLS tunnel** to the controller. There are **no inbound ports, no
VPN, no bastion, and the cluster API server is never exposed.**

The flow for a user running `kubectl`:

```
user kubectl ──▶ Rafay Access Proxy (controller)
                   │  authenticate user (SSO/OIDC)
                   │  authorize against project RBAC
                   │  issue/stamp a short-lived identity
                   │  LOG THE FULL REQUEST (verb, resource, body)
                   ▼
              reverse tunnel ──▶ relay agent in cluster ──▶ kube-apiserver
                                   (impersonating the mapped identity)
```

Consequences worth internalising:
- **Every kubectl command in the fleet is centrally audited**, including `exec` session
  capture. Auditors love this; it closes a real compliance gap (normal K8s audit logs
  are per-cluster and easily lost).
- **No standing credentials.** Nobody holds a long-lived kubeconfig for prod.
- **No network engineering required** to onboard a cluster anywhere — behind CGNAT,
  in a customer's DC, at a retail store.

**This is table stakes for us.** Rancher's `remotedialer` gives the transport for free;
the audit/identity layer on top is where the product value is.

---

## 3. Cluster Blueprints

A **Blueprint** is a versioned, named stack of add-ons + policies applied to a set of
clusters:

- base blueprint (platform-provided: monitoring, ingress, logging, policy, CSI, CNI) +
  custom add-ons (your Helm charts, versioned independently)
- **versioned and promotable** — `blueprint:v3` in dev, soak, then promote to prod
- **continuously enforced**: drift is detected and reverted. Ad-hoc `kubectl edit` on a
  managed component gets undone.
- v4.0 (Nov 2025) added **draft mode** (test blueprint changes before publish) and
  faster sync.

This is the same concept as Spectro's Cluster Profile, Fleet's Bundle, k0rdent's
ServiceTemplate. Rafay's version is the most "product-y": good UI, clear versioning,
explicit promotion workflow.

---

## 4. The rest of the platform

- **MKS (Rafay's upstream K8s)** — for provisioning onto VMs/bare metal you already
  have. This is the extent of their metal story historically: *you* provision the OS,
  Rafay installs Kubernetes on it.
- **GitOps pipelines** — multi-stage with approvals, system-sync + agent-based.
- **Environment Manager** — self-service catalog of Terraform/OpenTofu-backed
  environments (not just clusters: VPCs, databases, whole app stacks). An IDP feature.
- **Cost management** — per-project/per-namespace chargeback and showback.
- **Backup & restore**, secrets management, OPA/Kyverno policy, image scanning.
- **Air-gapped self-hosted** controller for gov/regulated buyers.

---

## 5. The GPU pivot — read this as market intelligence

Since ~2024 Rafay has aggressively repositioned around **"GPU PaaS"** for:
- **Neoclouds / NCPs** (NVIDIA Cloud Partners) — companies buying H100/H200/GB200 racks
  and reselling them,
- enterprises building internal **AI factories**.

What they added:
- multi-tenant **GPU partitioning** (MIG, time-slicing, fractional), quotas per project
- **dynamic GPU scheduling** and pooling across clusters
- **AI workbenches** (Jupyter/VS Code on demand) and **inference endpoints** as
  self-service products — the tenant never sees Kubernetes
- **metering + billing hooks** for GPU-hours, so an NCP can invoice from day one
- NVIDIA partnership/validation (GPU Operator, Network Operator, AICR reference
  architectures)
- and, notably, **bare-metal provisioning and VM lifecycle** finally appearing in their
  platform description by v4.0 — because neoclouds hand you racks, not VMs.

**The lesson for us:** the money in 2025–2027 for this category is in GPU
infrastructure, and the buyer is not "an enterprise platform team" but "a company whose
business model is selling compute". They need metal→K8s→multi-tenant→metering as one
product, and they need it in months, not years. Rafay is converging on that from the
top (control plane down); Red Hat/SUSE approach it from the bottom (metal up).
**The middle is open.**

---

## 6. Business model

- **SaaS subscription**, priced per cluster / per node / per GPU depending on the deal;
  also OEM/white-label for MSPs and cloud providers (they'll let a neocloud brand it).
- Self-hosted and air-gapped tiers at a premium.
- Land-and-expand from a few clusters to a fleet.

## 7. Where it hurts

1. **Closed source.** For sovereign, defense, and "no vendor lock" buyers this is
   disqualifying. Also means no community, no ecosystem, no free on-ramp.
2. **SaaS-first heritage.** The self-hosted/air-gapped product is heavier and less
   loved. Buyers in regulated/sovereign markets start from "it must run entirely in my
   DC".
3. **No real L1–L4.** Firmware, BIOS, RAID, BMC, inspection, network fabric — nothing.
   They assume the metal is racked, imaged, and network-configured. That is precisely
   the part that takes customers *weeks*.
4. **Kubernetes-only worldview.** Limited VM story compared to Harvester/OpenShift Virt
   /OpenStack — a problem when the same buyer also needs VMs.
5. **Control-plane dependency.** Lose the controller and you lose access and day-2
   automation to the whole fleet (workloads keep running).

## 8. What we steal

- **The tenancy model**: Org → Project → resources, with Project as the RBAC + cost +
  visibility boundary. Design this in from commit #1; retrofitting tenancy is brutal.
- **Audited zero-trust kubectl/exec through the hub.** Highest perceived-value-per-
  engineering-hour feature in the entire product.
- **Blueprint versioning + promotion + drift enforcement** (not just "apply").
- **Metering as a platform primitive**, not a reporting afterthought.
- **The GPU-tenant UX layer**: workbenches and inference endpoints, so the end user
  never learns Kubernetes. That's what turns infrastructure into a product an NCP can
  sell.
