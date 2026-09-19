# 10 — Our Reference Architecture, v0 (a strawman to argue with)

Working name: **`forge`** (placeholder). Everything here is a proposal to be attacked,
not a decision. The point of writing it down is to make the disagreements concrete.

---

## 1. Design principles

1. **Integrate, don't rebuild.** Every layer we can buy from open source, we buy. Our
   code is the API, the opinions, the glue, the UX, and the three gaps nobody fills.
2. **CAPI/Metal3/OCM are engines, never the user-facing API.** Users see our CRDs.
3. **Three tiers from day one**, collapsible to one for small deployments.
4. **Immutable OS delivered as OCI artifacts.** One supply chain for OS + addons +
   workloads.
5. **Pull for state, tunnel for ops, WireGuard for rescue.** Never store spoke
   kubeconfigs on the hub.
6. **Multi-tenancy in the data model from commit #1.** Retrofitting tenancy is a rewrite.
7. **Air gap is a build artifact**, not a documentation page.
8. **Every failure must produce one human-readable sentence** on our CR status, not
   six controllers' logs.

---

## 2. Topology

```
┌─────────────────────────────────────────────────────────────────────────┐
│  GLOBAL HUB  (RKE2, 3 nodes, HA — customer-hosted or our SaaS)          │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ forge-api        Tenant/Project/Catalog/RBAC/Audit/Billing (CRDs)  │  │
│  │ forge-console    React UI  ·  forge CLI  ·  Terraform provider     │  │
│  │ OCM hub          registration, Placement, ManifestWork, addons     │  │
│  │ forge-tunnel     audited kubectl/exec proxy (remotedialer-based)   │  │
│  │ forge-rollout    RolloutCampaign controller (TALM-class)           │  │
│  │ forge-inventory  Sites, Racks, Machines, IPAM, fabric SoT          │  │
│  │ catalog store    ClusterTemplates + AddonTemplates as OCI artifacts│  │
│  │ Dex/Keycloak · cert-manager · OpenBao · VictoriaMetrics · Perses   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │  OCM registration (mTLS, spoke-initiated)
                            │  + on-demand reverse tunnel
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ SITE CONTROLLER  │ │ SITE CONTROLLER  │ │ (collapsed: hub  │
│ DC-Mumbai        │ │ DC-Frankfurt     │ │  == site, small  │
│                  │ │                  │ │  deployments)    │
│ Metal3 BMO+Ironic│ │ …same…           │ └──────────────────┘
│ CAPI + CAPM3     │ │                  │
│ forge-firmware   │ │                  │   ← L3 engine (our code)
│ forge-fabric     │ │                  │   ← L6 engine (our code)
│ Zot mirror +     │ │                  │
│   Spegel         │ │                  │
│ Kamaji (hosted   │ │                  │
│   control planes)│ │                  │
│ vmagent buffer   │ │                  │
│ DHCP/HTTP boot   │ │                  │
│   (optional)     │ │                  │
└────────┬─────────┘ └────────┬─────────┘
         │                    │
   ┌─────┴──────┬─────────────┴──┬──────────────┐
   ▼            ▼                ▼              ▼
 dc-standard  hosted-cp        edge-sno       virtual
 (3+ nodes,   (CP in Kamaji,   (1 node,       (vCluster on
  local CP)    metal workers)   local CP)      shared nodes)
```

**Why the site controller exists:** Ironic must be near the BMCs; multi-GB OS images
must not cross the WAN N times; a hub outage must not stop a DC from replacing a dead
node; telemetry should aggregate before it egresses; and a sovereign customer may
require that a region never phones home.

---

## 3. Our API (the part users and the UI actually touch)

### Tenancy
```yaml
Organization           # the customer
  └── Project          # tenant / team / end-customer. HARD RBAC + cost + quota boundary
        ├── MachinePool      # metal allocated to this project
        ├── Cluster          # clusters in this project
        └── Quota            # nodes, GPUs, vCPU, storage
```

### Inventory & metal
```yaml
apiVersion: forge.io/v1alpha1
kind: Site                    # a datacenter / building / edge location
spec:
  region: ap-south-1
  controllerRef: site-mumbai  # which site controller owns it
  networks:                   # IPAM + VLAN plan for this site
    - name: provisioning
      vlan: 100
      cidr: 10.10.0.0/24
    - name: workload
      vlan: 200
      cidr: 10.20.0.0/22
      gateway: 10.20.0.1
  ntp: [...]
  registryMirror: zot.mumbai.internal
---
kind: Machine                 # ONE object for the server AND its fabric attachment
spec:
  siteRef: dc-mumbai
  rack: R07
  position: U12
  bmc:
    address: redfish-virtualmedia://10.10.7.12/redfish/v1/Systems/1
    credentialsRef: {name: bmc-r07, namespace: ...}
  hardwareProfileRef: gpu-h100-8x     # ← L3: desired firmware/BIOS/RAID state
  fabric:                             # ← L6: the switch side, declared here
    - nic: eno1
      switch: leaf-r07-a
      port: Ethernet1/12
      mode: trunk
      nativeVlan: 200
      allowedVlans: [200, 300]
      lag: po12
    - nic: eno2
      switch: leaf-r07-b
      port: Ethernet1/12
      lag: po12
  networkProfileRef: bond-lacp-vlan   # ← L5: nmstate template, rendered w/ IPAM
status:
  phase: Available            # Discovered|Inspecting|Remediating|Available|Allocated|Provisioned|Draining|Cleaning|Failed
  inventory: {cpu, memory, disks[], nics[], gpus[], numaTopology, tpm, bootMode}
  firmwareCompliance:
    compliant: false
    drift:
      - attribute: ProcC1E
        desired: "Disabled"
        actual: "Enabled"
        remediation: RequiresReboot
  conditions:
    - type: Ready
      status: "False"
      reason: FirmwareDrift
      message: "3 BIOS attributes differ from profile gpu-h100-8x; run `forge machine remediate r07-u12`"
```

### The L3 object — our first differentiator
```yaml
kind: HardwareProfile
metadata: {name: gpu-h100-8x}
spec:
  vendors: [dell-r760xa, supermicro-sys-821ge]   # per-vendor rendering
  firmware:
    bios:   {minVersion: "2.19.1", target: "2.19.1"}
    bmc:    {minVersion: "7.10.30.00"}
    nic:    {mellanox-cx7: "28.42.1000"}
  biosSettings:
    LogicalProc: Disabled          # HT off
    ProcC1E: Disabled
    ProcCStates: Disabled
    SysProfile: PerfOptimized
    SriovGlobalEnable: Enabled
    PcieAspmL1: Disabled           # matters for GPUDirect
    IommuSupport: Enabled
    MemoryOperatingMode: OptimizerMode
    BootMode: Uefi
    SecureBoot: Enabled
  storage:
    - devices: [disk0, disk1]
      raid: RAID1
      usage: boot
    - devices: [nvme*]
      raid: JBOD
      usage: data
  validation:                      # post-remediation proof
    - name: numa-gpu-affinity
      type: nvidia-topo
    - name: pcie-link-width
      expect: "x16 Gen5"
  enforcement: DriftDetectAndReport   # or AutoRemediateInMaintenanceWindow
```
This CR, plus a signed compliance report per node, is the thing we can demo that
nobody else can.

### Clusters (k0rdent-style template/instance split)
```yaml
kind: ClusterTemplate           # versioned, in the catalog, distributed as OCI
metadata: {name: dc-standard, version: 1.4.0}
spec:
  class: dc-standard            # edge-sno | edge-ha | dc-standard | hosted | virtual
  distro: rke2
  kubernetesVersion: "1.34.x"
  osImage: oci://registry/forge/os/ubuntu-24.04-bootc:1.4.0
  controlPlane: {replicas: 3, placement: local}
  addons: [cilium, metallb, rook-ceph, monitoring-agent, kyverno-baseline]
  variables:
    - {name: podCidr,  default: "10.42.0.0/16"}
    - {name: apiVip,   required: true}
---
kind: Cluster
metadata: {name: prod-mumbai-01, namespace: project-acme}
spec:
  templateRef: {name: dc-standard, version: 1.4.0}
  siteRef: dc-mumbai
  values: {apiVip: 10.20.0.10}
  machineSelector:
    matchLabels: {forge.io/pool: acme-gpu}
  addonOverlayRefs: [acme-baseline, mumbai-region]
status:
  phase: Ready
  message: "Ready. 3 CP / 12 workers. Template dc-standard 1.4.0. Drift: none."
```

### Day-2 — our third differentiator
```yaml
kind: RolloutCampaign
spec:
  target:
    clusterSelector: {matchLabels: {tier: edge, region: ap-south-1}}
  change:
    kind: ClusterTemplateUpgrade
    to: {name: edge-ha, version: 1.5.0}
  strategy:
    canaries: [prod-pune-01, prod-nashik-03]
    batchSize: 10
    maxConcurrent: 10
    maxFailures: 2                 # abort the campaign after this many
    preCache: true                 # pull OS + images to sites BEFORE the window
    maintenanceWindows:
      - {cron: "0 2 * * SUN", duration: 4h, timezone: site}   # site-local time
    preflight: [DiskSpace, EtcdHealthy, NoActiveAlerts, BackupComplete]
    verify:   [NodesReady, AddonsHealthy, WorkloadsHealthy]
    onFailure: PauseCampaign
status:
  phase: InProgress
  progress: {total: 187, succeeded: 42, inProgress: 10, failed: 1, pending: 134}
```

### Addons
```yaml
kind: AddonTemplate       # versioned Helm/kustomize/raw, OCI-distributed
kind: AddonOverlay        # per-project / per-region / per-site value overrides
```
Implemented via **Sveltos + Flux** underneath; the user never sees a `ClusterProfile`.

---

## 4. Provisioning flows

**Flow A — DC, BMC reachable (the Metal3 path)**
```
1. Import inventory (Nautobot / CSV / discovery scan of the mgmt subnet)
2. Machine CR created → forge-inventory validates BMC creds
3. Ironic inspects  → full hardware inventory populated
4. forge-fabric configures the switch ports (gNMI/NETCONF/eAPI)       ← L6
5. forge-firmware compares to HardwareProfile → remediates → verifies ← L3
6. Ironic cleans disks (secure erase if the pool is multi-tenant)
7. Deploy bootc OS image via Redfish VIRTUAL MEDIA (no DHCP needed)
8. nmstate applies the network profile; node comes up addressable
9. CAPI/CAPM3 joins it to the Cluster; RKE2 bootstraps
10. Sveltos applies the addon stack
11. OCM klusterlet registers the cluster with the hub
12. Cluster appears in the console; audited kubectl available
```

**Flow B — Edge, no BMC / no network path (the registration path)**
```
1. Create a Site + RegistrationToken
2. forge-imagebuild produces a site-specific installer (bootc/Kairos/Talos) with
   the token, network config, and pre-pulled images baked in
3. Ship the USB / serve the ISO / pre-image the appliance at the factory
4. Technician powers it on; optional local browser UI to pick the site ID
5. Agent phones home over HTTPS → Machine CR appears, Ready
6. Steps 9–12 as above
```

Both paths converge on the same `Machine` object. This is essential — one inventory,
one lifecycle, two on-ramps.

---

## 5. The three engines we actually write

Everything else is integration. These are the products.

### `forge-firmware` (L3)
Go service on the site controller. Uses `bmclib` + `gofish` + vendor CLIs
(`racadm`, `ilorest`, `sum`, `OneCLI`) behind a driver interface.
- Reconciles `HardwareProfile` → BMC.
- Maintains a **vendor/generation quirks database** (YAML, versioned, shipped as an
  OCI artifact so it updates independently of releases). *This is the compounding moat.*
- Ordered, safe remediation: settings-only (no reboot) → settings+reboot → firmware
  flash → verify → rollback on regression.
- Emits a **signed compliance attestation** per node (in-toto/cosign): "at time T,
  node X matched profile P at digest D." Auditors and AI-cluster buyers both want this.

### `forge-fabric` (L6)
Go service. Drivers for SONiC, Arista (eAPI/gNMI), Cisco NX-OS (NX-API/gNMI),
Juniper (NETCONF), Nokia SR Linux (gNMI); OpenConfig where possible.
- Renders per-port intent from `Machine.spec.fabric` + `Site.spec.networks`.
- **Dry-run diff first, always.** Network engineers will not accept a black box.
- Handles the provisioning-VLAN → production-VLAN transition during install.
- Reconciles LLDP neighbor data back into inventory to *verify the cable map*, which
  catches the single most common DC mistake: a miscabled server.
- Ships as **opt-in**, read-only by default. This is how we earn trust with the network
  team rather than getting banned by them.

### `forge-rollout` (L11)
Go controller implementing `RolloutCampaign`. Distro-agnostic, air-gap-aware,
resumable across hub restarts, with the pre-cache + backup + verify + abort semantics
above. Covers OS, Kubernetes, addons, and firmware as campaign types.

---

## 6. MVP — what to build first

**Demo target (the thing that gets a meeting):**
> "Here are 3 racked servers we've never touched. In 30 minutes, unattended and
> air-gapped, they become a compliant 3-node RKE2 cluster with Cilium and
> monitoring — and here is a signed report proving all three have identical firmware,
> BIOS and NUMA topology."

**Milestone 0 — virtual lab (weeks 1–3).** No hardware, no product code.
Build the lab (doc 11), run Metal3+CAPM3 end-to-end against `sushy-tools` emulated
Redfish BMCs, and separately stand up k0rdent, SUSE Edge, and Omni to feel the UX.
*Goal: earn the right to have opinions.*

**Milestone 1 — the spine (months 1–3).**
`Site`, `Machine`, `HardwareProfile` (detect-only), `ClusterTemplate`, `Cluster` CRDs;
Metal3 virtual-media provisioning of a bootc image; RKE2 via CAPI; Cilium; one CLI.
Single tier (hub == site). *Goal: the 30-minute demo, in the virtual lab.*

**Milestone 2 — the differentiator (months 3–5).**
`forge-firmware` remediation + quirks DB for **one** real vendor (Dell iDRAC9 is the
best-documented) + signed attestation. Rent real hardware. *Goal: the demo on metal,
with the compliance report.*

**Milestone 3 — the fleet (months 5–8).**
OCM registration, audited kubectl tunnel, Sveltos addons, `RolloutCampaign`, the
console, a second site, hub/site tier split. *Goal: it's a fleet product, not a
provisioner.*

**Milestone 4 — commercial (months 8–12).**
Projects/tenancy/quota, metering, air-gap bundle CLI, GPU stack (GPU Operator +
DCGM + MIG + GPU-hour metering), KubeVirt+Forklift for door B. *Goal: first paying
design partner.*

**Explicitly NOT in v1:** cross-cluster scheduling (Karmada), service mesh, a
marketplace, our own Kubernetes distro, our own CNI, our own OS from scratch,
Windows nodes, and anything telco-RAN-specific.

---

## 7. Licensing plan (open core)

| Tier | Contents | License |
|---|---|---|
| **forge-core** (OSS) | CRDs, provisioning (Metal3/Tinkerbell), CAPI integration, OS images, addons, CLI, single-site, single-tenant, `HardwareProfile` **detection** | **Apache 2.0** |
| **forge-enterprise** | Multi-tenancy + quota + metering, SSO/SAML, audited tunnel, `RolloutCampaign`, firmware **remediation** + attestation, `forge-fabric`, air-gap bundles, FIPS/STIG builds, multi-site, support | commercial |

The OSS tier must be **completely sufficient for one team running one site.** That's
what drives adoption and gives us the distribution a solo founder otherwise can't buy.
The paid tier is everything an *organisation* needs. Same split that works for
vCluster, Grafana, and Rancher-Prime.
