# 00 — The Problem, Decomposed

Before comparing vendors, we need a shared vocabulary. Every product in this space is
some subset of the same 13 layers. When a vendor says "we do bare metal Kubernetes",
the only useful question is: *which of these layers do you actually own, and which do
you assume someone else already did?*

This decomposition is the scoring rubric used in every vendor teardown that follows.

---

## L0 — Physical / Facility

Racking, cabling, power, cooling, serial console, PDU control, asset tags.

Mostly out of scope for software, **except**: a product that can't reconcile
"what the DCIM says is in rack 7" with "what actually answered on the management
VLAN" will generate endless support tickets. Cable-map awareness (which NIC goes to
which switch port) is the difference between a 20-minute bring-up and a 2-day one.

**Who owns it:** DCIM tools (NetBox, Nautobot, Device42), or nobody, or a spreadsheet.

---

## L1 — Out-of-Band Control (BMC)

The server's lights-out controller: iDRAC (Dell), iLO (HPE), XCC (Lenovo), IMM,
BMC on Supermicro, OpenBMC on hyperscale/ODM gear.

Protocols, in order of increasing modernity:
- **IPMI 2.0** — universal, ancient, insecure (cipher zero, RAKP hash leaks), UDP 623.
  Still the lowest common denominator on older fleets.
- **Redfish** — HTTPS + JSON, DMTF standard. Power, boot order, firmware inventory,
  BIOS attributes, **virtual media** (mount an ISO over HTTPS — no PXE required),
  event subscriptions, telemetry.
- **Vendor OEM extensions** — where the actual useful stuff lives (RAID config, secure
  erase, NIC partitioning, GPU health). Redfish "standard" coverage is thin here.

**This layer is where multi-vendor pain concentrates.** Redfish implementations differ
in firmware-revision-specific ways. A production-grade product needs a per-vendor,
per-generation quirks database. Open source helpers: `bmclib` (Go, Equinix), `gofish`
(Go, Redfish client), `sushy` (Python, OpenStack), `python-redfish`, `racadm`/`ilorest`
CLIs as fallbacks.

---

## L2 — Discovery & Inventory

Finding what's out there and describing it truthfully.

- **Passive**: DHCP fingerprinting on the provisioning VLAN, LLDP from switches,
  Redfish enumeration of a BMC IP range.
- **Active ("inspection" / "commissioning")**: boot an ephemeral agent (IPA, HookOS,
  MAAS commissioning kernel) that runs `lshw`/`dmidecode`/`lsblk`/`lspci`, benchmarks
  disks, reads NIC MACs and LLDP neighbors, and reports back.

Output should be a first-class object: CPU model/count/NUMA topology, RAM DIMM layout,
disks (model, serial, size, SMART, NVMe namespaces), NICs (MAC, speed, PCI addr, LLDP
neighbor + port), GPUs, BMC firmware version, TPM presence, boot mode (UEFI/Legacy),
Secure Boot state.

**Everyone's inventory model is too shallow.** Most stop at "4 CPUs, 128GB, 2 disks".
For AI/telco you need NUMA/PCIe topology (which GPU is on which root complex, which NIC
is NUMA-local to which GPU) or scheduling is garbage.

---

## L3 — Firmware, BIOS & RAID Desired State

Declaring: "every node in this pool runs BIOS 2.19.1, with SR-IOV enabled, C-states
disabled, HT off, boot mode UEFI, Secure Boot on, RAID1 on the boot pair, JBOD on the
rest, NIC firmware X".

Then: drift detection, safe ordered remediation (firmware flash → reboot → verify),
and rollback when a flash bricks a setting.

**This is the single most underserved layer in the whole market.** Metal3 has
`HostFirmwareSettings` / `HostFirmwareComponents` / `FirmwareSchema`; it's real but
thin. MAAS barely touches it. Tinkerbell has nothing native. Everyone else says
"use the vendor tool" (Dell OME, HPE OneView, Lenovo XClarity) — which means a second
management plane, a second UI, and a manual step in your "zero touch" workflow.

---

## L4 — OS Provisioning

Getting bits on disk and a bootable, addressable machine.

**Boot methods:**
- PXE/TFTP → iPXE chainload → HTTP(S) — classic, requires L2 adjacency or DHCP relay,
  requires owning DHCP (political fight in most enterprises).
- **HTTP Boot (UEFI)** — no TFTP, no PXE, DHCP option 60/67 with a URL.
- **Redfish Virtual Media** — mount ISO over HTTPS directly from the BMC.
  *No DHCP ownership required, works across routed networks, works at the edge.*
  This is strategically the most important boot path for a modern product.
- USB / factory-preinstalled image — the edge reality (ship an appliance).

**Install methods:**
- **Package-based**: kickstart/preseed/autoyast/AutoYaST + curtin. Flexible, slow,
  non-reproducible, drifts from day one.
- **Image-based**: write a prebuilt disk image, then customize with cloud-init/ignition.
  Fast, reproducible.
- **Container-native OS**: the image *is* an OCI artifact (bootc, Talos installer,
  Kairos, Flatcar with sysext). This is where the industry is converging.

---

## L5 — Host Network Configuration

Bonds (LACP vs active-backup), VLANs, MTU/jumbo frames, bridges, SR-IOV VFs, DPDK,
multiple routing tables, IPv6, RDMA/RoCE for AI fabrics.

Getting this wrong is the #1 cause of "the node provisioned but never joined".

Tools: nmstate (declarative, used by Metal3/OpenShift), netplan (Ubuntu/MAAS),
systemd-networkd, Talos machine config, cloud-init's (weak) network config.

---

## L6 — Network Fabric (Switches)

The leaf/spine side: access VLAN on the server port, MLAG/bond peer config, BGP
underlay, EVPN/VXLAN overlay, PXE-VLAN vs prod-VLAN switching during install,
per-tenant VRFs.

**Almost nobody in the Kubernetes-on-metal space touches this.** It's why "zero touch"
provisioning still takes two weeks: the servers are automated, the switches are a
change ticket. Adjacent tooling: SONiC, Nokia SR Linux, Arista CloudVision, Cisco NDFC,
Juniper Apstra, plus OpenConfig/gNMI/NETCONF and Netbox-as-source-of-truth.

**This is a credible differentiation wedge.**

---

## L7 — Kubernetes Bootstrap

Turning provisioned hosts into a cluster.

- **Distro choice**: kubeadm (vanilla), RKE2, k3s, k0s, Talos, MicroK8s, OpenShift,
  EKS-D/EKS-A, Kubespray (Ansible).
- **Control plane topology**: 3-node stacked etcd, external etcd, single-node (SNO),
  or **hosted control plane** (runs as pods elsewhere — HyperShift, Kamaji, k0smotron,
  Gardener shoots, vCluster).
- **The bootstrap chicken-and-egg**: something must exist before the cluster does.
  Patterns: an ephemeral kind/k3s "bootstrap cluster" that pivots into the target
  (CAPI's `clusterctl move`), or a permanent hub that provisions everything else.
- **HA VIP for the API server**: kube-vip, keepalived, MetalLB L2, or a hardware LB.

---

## L8 — Cluster Networking & Storage

CNI (Cilium, Calico, Flannel, kube-ovn, Multus for secondary NICs), LoadBalancer
(MetalLB, Cilium BGP, kube-vip), Ingress/Gateway API, and CSI (Rook/Ceph, Longhorn,
OpenEBS/Mayastor, local-path, TopoLVM, vendor arrays via CSI).

On metal there is no cloud LB and no cloud EBS, so these are *mandatory*, not optional.
This is exactly the gap between "kubeadm finished" and "a platform".

---

## L9 — Platform Add-ons ("the blueprint")

Monitoring, logging, policy, secrets, service mesh, backup, registry, cert-manager,
GPU operator, image scanning. Every vendor has a name for the bundle:
**Blueprint** (Rafay), **Cluster Profile** (Spectro), **Bundle** (Fleet),
**ClusterProfile/Profile** (Sveltos), **ServiceTemplate/MultiClusterService** (k0rdent),
**Policy + addon** (RHACM/OCM), **App/Chart** (Rancher).

Requirements that separate toys from products: **versioned**, **layered**
(base + overlay per site), **drift-detected and re-enforced**, **dependency-ordered**,
**canary-able**, and **air-gap-able**.

---

## L10 — Fleet Control Plane (the "hub")

The actual product surface:
- **Registration**: how a spoke proves identity and joins.
- **Connectivity**: pull agent, reverse tunnel, or direct API access.
- **Placement/targeting**: label-based selection of which clusters get what.
- **Multi-tenancy**: org → project → fleet → cluster; RBAC projection; SSO.
- **Interactive access**: audited kubectl/exec/logs through the hub without VPN.
- **Observability rollup**: metrics/logs/events from N clusters, per tenant.
- **Inventory of clusters**, health, versions, compliance, cost.

---

## L11 — Day-2 Lifecycle

Where products are actually judged, and where most die:

- **Upgrades**: OS, kubelet, control plane, addons — each with its own rollout policy,
  batching, canary, maintenance window, and rollback. Red Hat's TALM
  (`ClusterGroupUpgrade`) is the most developed model here.
- **Certificate rotation** — the silent killer of disconnected edge fleets. A site
  that's been offline 370 days has expired kubelet client certs and cannot rejoin.
- **etcd backup/restore**, node replacement, scale-out, cordon/drain semantics,
  disk replacement, GPU RMA workflows.
- **Drift**: someone SSH'd in and edited a file. Detect and correct, or make it
  impossible (immutable OS).
- **CVE remediation SLA** across a fleet you can't reach interactively.

---

## L12 — Commercial Wrapper

Not optional for a B2B product:

- **Air-gap / disconnected** as a first-class artifact, not a doc page.
- **Compliance builds**: FIPS 140-3, CIS benchmarks, STIG, Common Criteria.
- **Metering & chargeback**: per-tenant node-hours, GPU-hours, egress.
- **Licensing/entitlement** enforcement that works offline.
- **Support model**: what do you do at 3am when a customer's site is bricked and you
  have no network path to it?

---

## How to read the vendor teardowns

For each vendor, ask:

| Question | Why it matters |
|---|---|
| Which layers do they own vs. assume? | Reveals the real product boundary |
| What's the hub→spoke connectivity mechanism? | Determines scale ceiling & security posture |
| Where does the control plane physically run? | Determines cost per site |
| Mutable or immutable OS? | Determines day-2 cost and drift story |
| CAPI or bespoke? | Determines ecosystem leverage vs. control |
| What's genuinely open source? | Determines what we can reuse |
| What's the pricing unit? | Determines who they can and can't sell to |
| Where does it hurt in production? | That's our opening |
