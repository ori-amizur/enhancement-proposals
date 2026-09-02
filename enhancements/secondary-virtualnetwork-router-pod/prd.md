---
title: secondary-virtualnetwork-router-pod
authors:
  - oamizur@redhat.com
creation-date: 2026-08-25
tracking-link:
  - TBD
see-also:
  - "/enhancements/vn-subnet-connectivity"
replaces:
  - N/A
superseded-by:
  - N/A
---

# Secondary VirtualNetwork: A Router-Pod Networking Model with Bare-Metal Connectivity

| Field       | Value   |
|-------------|---------|
| Author(s)   | Ori Amizur (oamizur@redhat.com) |
| Jira        | TBD |
| Date        | 2026-08-25 |

## 1. Problem Statement

Tenants running VMs across multiple subnets within one VirtualNetwork need those VMs to communicate with each other, reach the internet, and — increasingly — reach bare-metal servers on the physical fabric, all through a single, VN-scoped point of control. OSAC's existing (Primary) VirtualNetwork networking model already provides this for VM-only workloads, but it has no built-in path to bare-metal, and it is not the only architecture worth offering: some deployments specifically want a **single router pod per VirtualNetwork** — one pod that is multi-homed across every subnet in the VN and acts as the gateway for all of them — rather than per-subnet infrastructure. This document proposes exactly that model, as an explicit, opt-in **Secondary** VirtualNetwork type, delivered in two phases: a self-contained in-cluster model first, extended with bare-metal fabric connectivity second.

This router-pod model was described in an earlier design phase of OSAC's networking work — never implemented or shipped — before a different architecture was chosen as the default. Its core mechanism is sound, and is proposed here, refined, as a first-class, selectable option:

### Foundation mechanism: namespace-per-VN with a router pod

- **Namespace-per-VirtualNetwork**: when a VirtualNetwork is provisioned, a single namespace is created for it. Every subnet in the VN shares this one namespace, rather than each subnet getting its own.
- **Secondary CUDNs per subnet**: each subnet creates a ClusterUserDefinedNetwork with `role: Secondary` (not `Primary`), targeting the router pod's namespace via a shared namespace label. Secondary UDNs are required here because the router pod must be multi-homed across every subnet in the VN from within a single namespace — OVN-Kubernetes allows only one Primary UDN per namespace, which rules that type out for this multi-homed role.
- **Router pod**: the osac-operator deploys a single router pod in the VN namespace. It has one interface per subnet and acts as the default gateway (`.1`) for every subnet's VMs. It performs L3 forwarding between subnets, and provides internet egress via SNAT to the cluster network.
- **VM attachment**: VMs attach to their subnet's Secondary UDN (not the pod network), get their IP from OVN IPAM via DHCP, and have a route to the router pod injected via cloud-init (since Secondary UDNs do not provide a DHCP-supplied gateway option). The route's scope depends on the VM: a full default route to `.1` for a VM with only Secondary VN subnets, or a route scoped to just this VN's own CIDR via `.1` for a VM that also has a Primary UDN attachment elsewhere — so the Secondary VN attachment never overrides that attachment's default route, while still reaching every subnet in the VN through the router pod's inter-subnet forwarding. A VM's *namespace*, however, is not necessarily the router pod's namespace — see below.
- **Gateway IP persistence**: the `.1` gateway IP on each subnet is reserved via IPAMClaim with `ipam.lifecycle: Persistent`, so the router pod recovers the same addresses after a restart.
- **Port security**: OVN port security on the router pod's logical switch ports must be patched to allow forwarding of packets with destination IPs outside the router's own address — needed because the router pod forwards traffic on behalf of every VM in the VN, not just itself.

### Delivery Phases

**Phase 1 — In-cluster router pod (no fabric manager required).** Everything needed for a fully functional Secondary VirtualNetwork on a deployment with no fabric manager at all:

- The VirtualNetwork type flag, defaulting to OSAC's existing model.
- The foundation mechanism above: namespace-per-VN, router pod, Secondary CUDNs, `.1` gateway, SNAT-based internet egress.
- The agent, reconciling gateway-IP and routing/port-security state live for interfaces that already exist.
- The two-tier subnet add/remove mechanism: live attachment via `multus-dynamic-networks-controller`, falling back to router pod recreation (Section 9).
- NAT Gateway support via OVN-Kubernetes EgressIP, sourced from the ExternalIPPool.

Phase 1 is a complete, independently shippable capability — a tenant on a fabric-manager-less deployment gets full inter-subnet routing, egress, and NAT Gateway with nothing from Phase 2.

**Phase 2 — Fabric manager integration.** Adds bare-metal fabric connectivity on top of Phase 1, only for deployments that have a fabric manager configured:

- One additional transit interface on the router pod — a Primary CUDN with EVPN transport.
- Bidirectional static route dispatch between the router pod and the fabric manager.
- End-to-end VM ↔ bare-metal connectivity.

Phase 2 requires Phase 1 to already be in place and adds capability without changing Phase 1 behavior for any VirtualNetwork that doesn't have a fabric manager configured.

### Architecture

```text
                         VirtualNetwork Namespace
   ┌───────────────────────────────────────────────────────────────┐
   │                                                                 │
   │   Subnet-A (10.0.1.0/24)          Subnet-B (10.0.2.0/24)        │
   │   ┌───────────────┐               ┌───────────────┐            │
   │   │ VM-A          │               │ VM-B          │            │
   │   │ 10.0.1.5      │               │ 10.0.2.5      │            │
   │   │ gw: .1        │               │ gw: .1        │            │
   │   └───────┬───────┘               └───────┬───────┘            │
   │           │ Secondary UDN                 │ Secondary UDN      │
   │           │ (router pod's .1 interface)   │ (router pod's .1)  │
   │           └───────────────┬───────────────┘                    │
   │                           │                                    │
   │                 ┌─────────▼──────────┐                         │
   │                 │     Router Pod      │                         │
   │                 │                     │                         │
   │                 │  Agent:             │                         │
   │                 │  watches ConfigMap, │                         │
   │                 │  configures gateway │                         │
   │                 │  IPs, routes, port  │                         │
   │                 │  security live      │                         │
   │                 │                     │                         │
   │                 │  cluster-net iface  ├────────► Internet (SNAT,│
   │                 │                     │           or NAT Gateway│
   │                 │                     │           via EgressIP) │
   │                 │  transit iface      │                         │
   │                 │  (Phase 2 only:     │                         │
   │                 │   Primary CUDN,     │                         │
   │                 │   EVPN transport)   │                         │
   │                 └─────────┬───────────┘                         │
   └───────────────────────────┼─────────────────────────────────────┘
                                │  EVPN MAC-VRF (Phase 2 only)
                                │
                     ┌──────────▼───────────┐
                     │   Physical Fabric      │
                     │   (e.g., Netris)       │   Phase 2 only
                     │                        │
                     │   Bare-Metal Server    │
                     │   10.0.3.9             │
                     └────────────────────────┘
```

Everything below the router pod's `transit iface` box exists only in Phase 2 — a Phase 1 router pod has just the per-subnet and cluster-net interfaces.

#### VMs that also need network access outside this VirtualNetwork

OVN-Kubernetes allows only one Primary UDN per namespace. OSAC's existing Primary VirtualNetwork model attaches VMs via a Primary UDN, so a VM that uses that model for one VirtualNetwork *and* needs a subnet in this Secondary VirtualNetwork cannot live in the router pod's own namespace — its namespace is dictated by whatever Primary UDN it already needs.

This means a Secondary VN subnet's Secondary UDN cannot target a single fixed namespace; it must select every namespace containing a VM attached to that subnet via a shared label. Consequently, the router pod's own namespace holds the router pod itself and any VMs that attach *only* to this Secondary VirtualNetwork's subnets — it is not guaranteed to contain every VM that uses it. This applies in both phases.

### Traffic flows

```text
Phase 1 — Inter-subnet (east-west):
VM-A (subnet-a) → default gw .1 (router pod) → router pod forwards → VM-B (subnet-b)

Phase 1 — Internet egress (default, no NAT Gateway):
VM-A (subnet-a) → default gw .1 (router pod) → router pod SNATs to cluster
  network IP → node masquerade → internet

Phase 1 — Internet egress (with NAT Gateway configured):
VM-A (subnet-a) → default gw .1 (router pod) → SNATed to router pod's own IP
  (unchanged) → OVN-Kubernetes EgressIP (matches router pod's namespace) →
  SNATed again to the NATGateway's ExternalIP → internet

Phase 2 — VM → bare-metal (via the fabric transit leg):
VM-A (subnet-a) → default gw .1 (router pod) → router pod forwards via its
  transit interface → EVPN MAC-VRF → physical fabric → bare-metal server

Phase 2 — Bare-metal → VM:
bare-metal server → fabric's local gateway → static route (Section 9) →
  router pod's transit interface → router pod forwards → VM-A
```

Implemented naively, this mechanism would carry real operational costs: the router pod would be a single point of failure per VN, and adding a subnet would require recreating the router pod (causing a brief interruption and requiring OVN port security to be re-patched on the new pod), with the pod's MAC address changing on every recreation and requiring ARP reconvergence across every subnet in the VN. This PRD avoids the recreation cost for route and configuration changes on interfaces that already exist, by design, via the agent-based reconciliation in Section 2. For subnet add/remove itself, this PRD's primary approach (Section 9, Phase 1) attempts live interface attachment, with router pod recreation as an automatic fallback when that isn't available or doesn't succeed (see Risk 6.2). The remaining costs are tracked as risks in Section 6.

### What's new in this PRD

Building on that foundation, this PRD proposes:

1. **(Phase 1)** Making this router-pod/Secondary-UDN model an explicit, selectable **Secondary** VirtualNetwork type, alongside OSAC's existing Primary type as the default value of a new flag.
2. **(Phase 1)** Adding an **agent** to the router pod that reconciles gateway-IP and routing/port-security state live from a ConfigMap, without a pod restart, for interfaces that already exist — and that also watches for a newly-attached interface appearing (Section 9) so a subnet can be added without a pod restart when that mechanism is available. When it isn't available, or the new interface doesn't appear within a bounded time, the operator falls back to recreating the router pod, mitigated by persistent IPAMClaim addressing so existing subnets recover their addresses quickly (see Risk 6.2).
3. **(Phase 1)** Supporting NAT Gateway on fabric-manager-less deployments via OVN-Kubernetes EgressIP, sourced from the existing ExternalIPPool, with no change to the router pod itself.
4. **(Phase 2)** Extending the router pod to reach **bare-metal servers on the physical fabric**, which the foundation mechanism does not address at all — only for deployments that have a fabric manager configured. Since Secondary UDNs do not support native EVPN transport (EVPN is scoped to Primary UDNs only, and Secondary UDN support is listed as future work upstream), the router pod gains one additional transit interface — a Primary CUDN with EVPN transport — to reach the fabric's shared transit MAC-VRF. The fabric-side route to each of the VN's subnets is published via a **static route pushed through the operator's existing dispatcher mechanism** (the same mechanism already used to manage the fabric manager's VPCs, routes, and NAT rules), rather than via a live routing protocol. Running a BGP daemon inside the router pod to publish and learn these routes dynamically is a viable alternative, discussed in Section 8, but is not the primary approach proposed here.

Today OSAC has no VN-scoped, single-router-pod networking option at all, no live reconciliation mechanism for such a router pod, and no bare-metal connectivity path for it.

### User Stories

**Tenant stories:**

- **(Phase 1)** As a tenant, I want to choose whether my VirtualNetwork uses OSAC's Primary connectivity model or this router-pod model, so I can select the approach that fits my workload.
- **(Phase 1)** As a tenant with a Secondary VirtualNetwork, I want VMs in different subnets to communicate and to reach the internet through the router pod.
- **(Phase 1)** As a tenant with a Secondary VirtualNetwork, I want to add or remove subnets without disrupting connectivity for my other subnets, so my network can grow without downtime.
- **(Phase 1)** As a tenant with a Secondary VirtualNetwork on a deployment without a fabric manager, I want to configure a NAT Gateway so my egress traffic uses a known, stable external IP instead of a generic, shared one.
- **(Phase 2)** As a tenant with a Secondary VirtualNetwork, I want VMs on any of my subnets to reach bare-metal servers on the fabric, and vice versa, so I can build applications that span both.

**Provider stories:**

- **(Phase 1)** As a provider, I want the VirtualNetwork type to default to OSAC's existing networking model, so current tenants and deployments are unaffected unless a tenant explicitly opts into the Secondary model.
- **(Phase 1)** As a provider, I want the router pod to reconcile subnet membership automatically as subnets are added or removed, so this doesn't require a pod restart or manual AAP/operator intervention per change.
- **(Phase 2)** As a provider, I want the router pod's fabric connectivity to reuse the fabric manager's existing route-management interface, so bare-metal reachability doesn't require standing up a new peering relationship or routing daemon.

### Goals

#### Phase 1 Goals

- Introduce a VirtualNetwork-level type distinction between OSAC's existing **Primary** model and this **Secondary** (router-pod/Secondary-UDN) model, selectable via a flag on the VirtualNetwork, defaulting to `Primary`.
- Establish the router-pod/Secondary-UDN inter-subnet routing and internet-egress mechanism described above as the Secondary VirtualNetwork's core behavior.
- Add an agent process to the router pod that reconciles gateway-IP and routing/port-security state live for interfaces that already exist, and that detects and configures a newly-attached interface when the operator adds a subnet (Section 9), all without a pod restart.
- Use router pod recreation as an automatic fallback for subnet add/remove when live attachment is unavailable or does not succeed within a bounded time (Risk 6.2), so the feature degrades to a known-working path rather than failing outright.
- Support NAT Gateway for the Secondary VirtualNetwork by giving its egress traffic a dedicated, stable external IP via OVN-Kubernetes EgressIP, sourced from the existing ExternalIPPool — without requiring any change inside the router pod itself.

#### Phase 2 Goals

- Provide bare-metal fabric connectivity for Secondary VirtualNetworks via a Primary CUDN/EVPN transit interface on the router pod and operator-pushed static routes on both the fabric side and the router pod's own routing table — only when the target deployment has a fabric manager configured.

Deployments without a fabric manager remain on Phase 1 behavior indefinitely — provisioning the transit interface, EVPN, and static-route dispatch is conditional on fabric-manager presence, not a staging step every deployment eventually passes through.

### Who knows about which subnets, and who applies that knowledge

Every subnet in a VirtualNetwork — VM-hosted or bare-metal-hosted — is provisioned as an OSAC `Subnet` resource through the same fulfillment-service/osac-operator control plane. The operator is therefore never in a position of needing to *discover* what subnets exist elsewhere on the network the way a BGP-speaking router would be — it is the controller that reconciles every `Subnet` CR in the first place, and so has complete, authoritative knowledge of every subnet's CIDR and type the moment it is created.

This produces a clean division of responsibility:

- **The operator** is the source of truth for subnet membership. On every subnet add/remove in a VirtualNetwork, it computes the full set of routes required — both the VN's own subnets and (Phase 2) any fabric-reachable (bare-metal) subnets — and (a) writes them to the router pod's ConfigMap, and (b) in Phase 2, pushes or withdraws the corresponding static route in the fabric manager.
- **The agent**, running in the router pod, does not independently know about subnet existence and makes no policy decisions. It watches the ConfigMap and reconciles the pod's local interfaces and routing table to match whatever the operator has written.

### Non-Goals

- Changes to OSAC's existing (Primary) VirtualNetwork networking model — it is unaffected by this work.
- High availability or multiple router pod instances per VirtualNetwork (single instance only, in either phase).
- The non-EVPN transit alternative (a plain macvlan/SR-IOV attachment with external BGP/VRF-Lite) — deferred as a possible future alternative, not part of Phase 2 as proposed.
- Running a BGP daemon inside the router pod — a viable Phase 2 alternative (Section 8), but not part of this scope.
- Migration tooling to convert an existing VirtualNetwork between the Primary and Secondary type.
- NAT Gateway integration for Secondary VirtualNetworks **that have a fabric manager configured** — for those, NAT Gateway is delegated entirely to the fabric manager's own native SNAT mechanism, the same as for any other VirtualNetwork, and is unaffected by this work. (NAT Gateway for the no-fabric-manager case is Phase 1 — see Section 2.1.)
- VPC peering integration for Secondary VirtualNetworks.

## 2. In Scope / Out of Scope

### 2.1 In Scope

#### Phase 1

- A `VirtualNetwork` type/mode field distinguishing **Primary** and **Secondary**, immutable after creation, defaulting to `Primary`.
- For `Secondary` VirtualNetworks: the osac-operator provisions a single namespace for the VN and a single router pod within it, with one Secondary UDN attachment per subnet, per the foundation mechanism in Section 1.
- Router pod behavior as the default gateway (`.1`) per subnet, L3 forwarding between subnets, and SNAT-based internet egress, matching the foundation mechanism.
- Gateway IP reservation via IPAMClaim with persistent lifecycle, and cloud-init-injected routes on VMs (a full default route, or a route scoped to the VN's own CIDR for VMs with a Primary UDN attachment elsewhere), matching the foundation mechanism.
- DNS resolver configuration for VMs with only Secondary VN subnets, injected via cloud-init to mirror the router pod's own DNS configuration; VMs with a Primary UDN attachment elsewhere are unaffected, since their DNS is already provided by that attachment.
- Support for VMs that also need a Primary UDN elsewhere: each subnet's Secondary UDN targets namespaces via a shared label rather than a single fixed namespace, so such a VM can still attach to a Secondary VN's subnet from its own namespace.
- A router pod agent that watches a ConfigMap and, for interfaces that already exist on the router pod, applies gateway-IP and routing/port-security state changes at runtime without a pod restart. The agent applies whatever the operator has written to the ConfigMap; it does not independently query or discover subnet existence.
- When a subnet is added to a Secondary VirtualNetwork, the operator first attempts live attachment: it patches the router pod's Multus network-attachment annotation with the new Secondary UDN, relying on `multus-dynamic-networks-controller` (Section 9) to inject the interface into the running pod. The agent watches for the new interface and, once it appears, configures it (link up, IPAMClaim-reserved gateway IP, routes, port security) — no pod restart in this path.
- If the new interface does not appear within a bounded time, or `multus-dynamic-networks-controller` is not installed/available in the target cluster, the operator falls back to recreating the router pod with the updated set of Secondary UDN attachments. The same fallback applies to subnet removal. Persistent IPAMClaim addressing ensures existing subnets recover their addresses without re-provisioning in either path.
- NAT Gateway support for the Secondary VirtualNetwork, implemented by the operator creating an OVN-Kubernetes EgressIP resource selecting the router pod's namespace, using the address from the ExternalIP the tenant's NATGateway resource references. The router pod requires no changes for this — its existing default-egress path (VM → router pod's cluster-net interface, SNATed to the pod's own IP) is unchanged; the EgressIP mechanism performs a second SNAT to the dedicated external address at the point traffic leaves the node, entirely outside the pod.

#### Phase 2

- One additional transit interface on the router pod: a Primary CUDN with EVPN transport, used solely to reach the fabric's shared transit MAC-VRF. This is provisioned only when the target deployment has a fabric manager configured, at VirtualNetwork creation time; a fabric-manager-less VirtualNetwork never gains this interface.
- Static addressing for the transit interface, reserved through the operator's existing dispatcher mechanism via IPAMClaim with `ipam.lifecycle: Persistent` (consistent with the constraint that pod network interfaces cannot use DHCP) — so the router pod recovers the same transit address across restarts and reschedules.
- A static route, pushed by the operator through the fabric manager's existing route-management API whenever a VM subnet is added or removed, publishing that subnet's CIDR to the fabric with the router pod's transit-interface address as next-hop.
- The mirror-image static route, written by the operator to the router pod's ConfigMap and applied by the agent, whenever a bare-metal (fabric-reachable) subnet becomes relevant to the VirtualNetwork: `<bare-metal subnet CIDR> via <fabric peer's transit address> dev <transit interface>`. The operator determines this route's content from its own authoritative Subnet CR state, not from any discovery mechanism.

### 2.2 Out of Scope

- Any change to OSAC's existing (Primary) VirtualNetwork networking model.
- Router pod high availability.
- Conversion between VirtualNetwork types after creation.
- The macvlan/SR-IOV + external BGP/VRF-Lite transit alternative.
- Running a BGP daemon inside the router pod (see Section 8 for this alternative).

## 3. Acceptance Criteria

### 3.1 Phase 1

- [ ] A VirtualNetwork can be created with the type explicitly set to Secondary; omitting the field defaults to the existing model.
- [ ] The type field is immutable after VirtualNetwork creation.
- [ ] A Secondary VirtualNetwork provisions exactly one namespace and one router pod, with one Secondary UDN attachment per subnet in the VN.
- [ ] VMs in different subnets of a Secondary VirtualNetwork can communicate via the router pod's `.1` gateway on each subnet.
- [ ] VMs in a Secondary VirtualNetwork can reach the internet via the router pod's SNAT egress path.
- [ ] A VM with only Secondary VN subnets successfully resolves both external and cluster-internal DNS names using a resolver configuration identical to the router pod's own; a VM with a Primary UDN attachment elsewhere retains that attachment's DNS configuration unaffected.
- [ ] A VM with a Primary UDN attachment elsewhere and a Secondary VN subnet reaches every other subnet in that VN via a cloud-init-injected route scoped to the VN's own CIDR through `.1`, while its Primary attachment's default route is left untouched.
- [ ] When the target deployment has no fabric manager configured, the router pod is provisioned without a transit interface and without any fabric-side static route dispatch — it provides in-cluster inter-subnet routing and internet egress only.
- [ ] When a NATGateway is configured for a Secondary VirtualNetwork with no fabric manager, VM egress traffic is source-NATted to the NATGateway's ExternalIP via an OVN-Kubernetes EgressIP selecting the router pod's namespace, with no changes to the router pod's own configuration or agent behavior.
- [ ] Without a NATGateway configured, a fabric-manager-less Secondary VirtualNetwork's egress behavior is unaffected — traffic continues to use the router pod's existing default SNAT-to-cluster-network path.
- [ ] When `multus-dynamic-networks-controller` is available, adding a subnet to a Secondary VirtualNetwork results in the new interface being injected into the running router pod, detected and configured by the agent (gateway IP, routes, port security), with no pod restart.
- [ ] When live attachment is unavailable, or the new interface does not appear within the defined timeout, adding a subnet results in the router pod being recreated with the new Secondary UDN attachment; existing subnets recover their addresses via persistent IPAMClaim.
- [ ] Removing a subnet follows the same two-tier approach: live detachment via `multus-dynamic-networks-controller` where available, falling back to router pod recreation otherwise; remaining subnets recover their addresses via persistent IPAMClaim in the fallback path.
- [ ] Route, gateway-IP, or port-security changes on an interface that already exists on the router pod are always applied live by the agent, without a pod restart, regardless of which path was used to attach the interface.
- [ ] The router pod's agent never independently queries or discovers subnet existence — all routing state it applies originates from the operator via the ConfigMap.
- [ ] Existing VirtualNetworks using OSAC's Primary networking model are unaffected by this work.

### 3.2 Phase 2

- [ ] When the target deployment has a fabric manager configured, the router pod is provisioned with the transit interface and the full bare-metal-connectivity mechanism described below.
- [ ] Adding a VM subnet also results in the operator pushing a static route for that subnet's CIDR into the fabric manager, with the router pod's transit address as next-hop; removing a subnet results in that route being withdrawn.
- [ ] When a bare-metal subnet becomes relevant to a Secondary VirtualNetwork, the operator writes the corresponding route to the router pod's ConfigMap, and the agent applies it to the router pod's routing table without a pod restart.
- [ ] A VM on any subnet of a Secondary VirtualNetwork can reach a bare-metal server on the fabric, and a bare-metal server can reach the VM, once the corresponding static routes are in place on both sides.
- [ ] Deploying Phase 2 does not change behavior for any existing Phase-1-only (fabric-manager-less) Secondary VirtualNetwork.

## 4. Assumptions

- The fabric manager's existing route-management API can accept a static route pointing at the router pod's transit-interface address as next-hop, the same way it already accepts routes for other purposes (e.g., SNAT default routes, NAT rules) — this is a much lower bar than establishing a new BGP peering relationship, and is assumed to already be within the fabric manager's existing capability. (Phase 2)
- Secondary UDN support for native EVPN transport will not become available in the relevant timeframe; if it does, this static-route approach (and the BGP alternative in Section 8) may become unnecessary for future Secondary VirtualNetworks, though existing ones would still need one of the two, or a migration path. (Phase 2)
- A single, non-VRF-isolated routing table inside the router pod is sufficient, since the pod's network namespace is already single-tenant (see the companion design document for the reasoning). (Both phases)
- Static, operator-pushed routing is an acceptable trade-off for this use case because subnet add/remove is already a fully operator/controller-orchestrated event — there is no independent discovery problem for a dynamic protocol to solve here. (Phase 2)
- The operator (fulfillment-service/osac-operator) is the authoritative source of truth for every subnet's existence, CIDR, and type in a VirtualNetwork, since it is the controller that provisions each `Subnet` resource. It therefore never needs to discover remote subnets the way a BGP-speaking router would — it already knows them, and can compute and push routes for both the fabric side and the router pod's own routing table from that existing state. (Both phases)
- Whether a Secondary VirtualNetwork's target deployment "has a fabric manager configured" is something the operator can determine unambiguously from the NetworkClass at VirtualNetwork creation time. This is assumed but not yet confirmed against the current NetworkClass API — OSAC's existing Unified Networking model generally treats `fabricManager` as a required field, which is in tension with the premise that some deployments have none at all (Open Question 7.6). This determination gates whether Phase 2 provisioning happens at all, so it needs to be resolved before Phase 2 begins, though it does not block Phase 1.
- ExternalIPPool CIDRs for a fabric-manager-less NetworkClass are provisioned by the admin with node-level EgressIP assignment in mind, the same way CIDRs for a fabric-manager-backed NetworkClass are already provisioned to suit that fabric's routing/DNAT mechanism — this is an existing provisioning pattern applied to a new mechanism, not a new capability the software must independently guarantee (Open Question 7.7 covers documenting the specific topology needed). (Phase 1)
- A namespace-scoped EgressIP selecting the router pod's namespace cannot inadvertently affect VM traffic, since VMs attach only to their subnet's Secondary UDN and never to the pod/cluster network — the router pod is assumed to be the only entity in its namespace with a cluster-network presence. (Phase 1)

## 5. Dependencies

- **(Phase 1)** `multus-dynamic-networks-controller` (Section 9) as the primary subnet-attachment mechanism — an unsupported upstream Network Plumbing Working Group project, not bundled or documented by Red Hat for non-VM pods (Risk 6.6). It is plausibly already present on every cluster relevant to this feature, since OSAC requires OpenShift Virtualization for VMaaS and OpenShift Virtualization's own VM interface hot-plug feature is built on this same controller — so OSAC likely does not need to deploy it separately. What is not yet confirmed is whether that instance is scoped to react only to KubeVirt-managed pods rather than arbitrary pods like the router pod (Risk 6.7); the osac-operator must detect whichever of these is true and fall back to router pod recreation (Risk 6.2) when live attachment is absent, scoped away from the router pod, or does not succeed.
- **(Phase 1)** osac-operator changes to dispatch provisioning based on VirtualNetwork type, and to support Secondary UDN-based router pod provisioning.
- **(Phase 2)** Fabric manager support for accepting a static route via its existing route-management API, scoped to the router pod's transit address.
- **(Phase 2)** osac-operator changes to push/withdraw the fabric-side static route alongside each subnet add/remove.
- The companion design document that specifies the router pod's agent behavior, live-attachment/fallback logic (Phase 1), and EVPN transit attachment/static-route dispatch (Phase 2) in technical detail.

## 6. Risks

### 6.1 Router pod is a single point of failure with no HA in this phase

- **Phase:** Both
- **Owner:** Networking / osac-operator team
- **Mitigation:** Kubernetes restarts the pod on failure; document the expected reconvergence window. HA is explicitly out of scope for this phase and may be revisited later.

### 6.2 When live attachment isn't available, adding or removing a subnet falls back to recreating the router pod, changing its MAC addresses and requiring ARP reconvergence across every subnet in the VN

- **Phase:** 1
- **Owner:** Networking team
- **Mitigation:** This PRD's primary mechanism (Section 9) avoids this cost by attaching the new interface to the running router pod via `multus-dynamic-networks-controller`, detected and configured by the agent. This risk applies specifically to the fallback path — triggered when that mechanism is not installed/available in the target cluster, or the interface does not appear within the defined timeout. Persistent IPAMClaim addressing (Section 2.1) keeps IP addresses stable across the recreation in the fallback path; MAC-based ARP reconvergence still applies, the same as it would on pod failure. May be revisited alongside HA.

### 6.6 The primary subnet-attachment mechanism depends on unsupported upstream behavior (`multus-dynamic-networks-controller`)

- **Phase:** 1
- **Owner:** Networking / osac-operator team, with Red Hat support/product sign-off
- **Mitigation:** This is not an officially documented or supported OpenShift capability for non-VM pods (Section 9), so its behavior across OpenShift versions is not guaranteed by Red Hat's support SLA. The recreation fallback (Risk 6.2) ensures subnet add/remove still works if this mechanism regresses, is removed upstream, or is unavailable in a given cluster — but the *no-restart* property this PRD aims for depends on it. Product/support should explicitly sign off on this dependency before this design ships as the default; monitor upstream `multus-dynamic-networks-controller` status as part of ongoing maintenance.

### 6.7 The controller may be present in the cluster (via OpenShift Virtualization) but scoped to KubeVirt pods only, never triggering for the router pod

- **Phase:** 1
- **Owner:** Networking / osac-operator team
- **Mitigation:** OSAC already requires OpenShift Virtualization for VMaaS, and OpenShift Virtualization's own VM interface hot-plug feature depends on `multus-dynamic-networks-controller` — so the controller is plausibly already running wherever this feature would be used, without OSAC deploying it separately. However, that instance may be scoped (via RBAC or its own configuration) to react only to virt-launcher pods, not to the router pod. If so, the primary mechanism would silently never engage — every subnet change would take the fallback (recreation) path even though the controller is technically "installed." This needs to be verified directly against the controller's actual deployment/RBAC configuration (Open Question 7.5); the fallback keeps the feature correct either way, just without the intended no-restart benefit if scoping rules it out.

### 6.3 Cloud-init dependency for the VM's route to the router pod

- **Phase:** Both
- **Owner:** Networking team
- **Mitigation:** Document the cloud-init requirement; VMs without cloud-init have no route to `.1` — a full default route for Secondary-VN-only VMs, or a VN-CIDR-scoped route for dual-attached VMs — and lose inter-subnet and (for Secondary-VN-only VMs) egress connectivity.

### 6.4 Static routes can drift from actual subnet membership if the operator's push/withdraw step fails independently of the router pod agent's own reconciliation

- **Phase:** 2
- **Owner:** osac-operator team
- **Mitigation:** Treat the fabric-side route push/withdraw as part of the same reconciliation loop as the ConfigMap update, with retry/requeue on failure, so the two stay consistent.

### 6.5 Fabric-side route changes are not instantaneous and are not reflected back to the router pod dynamically

- **Phase:** 2
- **Owner:** Networking team
- **Mitigation:** Acceptable for this phase since subnet changes are already operator-paced events, not something requiring sub-second propagation; revisit if requirements change.

### 6.8 EgressIP requires its address to be assignable on some node's own network, which the admin must account for when provisioning the ExternalIPPool for a fabric-manager-less NetworkClass

- **Phase:** 1
- **Owner:** Networking / osac-operator team
- **Mitigation:** This is not a new requirement — ExternalIPPool CIDRs are already chosen per NetworkClass by the admin to suit whatever mechanism makes that NetworkClass's addresses reachable (today, the fabric manager's BGP/DNAT). A fabric-manager-less NetworkClass using EgressIP is the same pattern with a different mechanism: the admin provisions its ExternalIPPool with CIDRs valid for node-level address assignment (e.g., a subnet on the nodes' own external-facing NIC) instead. This should be captured in the provisioning documentation for fabric-manager-less NetworkClasses, not treated as an open technical question.

## 7. Open Questions

### 7.1 After a router pod restart, how long does it take for the fabric to relearn the transit interface's new MAC via EVPN, and is that gap acceptable?

- **Phase:** 2
- **Owner:** Networking / SRE
- **Impact:** The transit address itself is stable across restarts (via persistent IPAMClaim, per Section 2.1), but a new pod instance still gets a new MAC, so the fabric's EVPN Type-2 (MAC/IP) advertisement for that address must be relearned before the static route's next-hop is reachable at L2 again. This is the same underlying issue as Risk 6.2, applied to the transit leg specifically — determines whether any additional mitigation (e.g., gratuitous ARP on startup) is needed for this phase.

### 7.2 How does label-based namespace targeting for Secondary UDNs scale and stay consistent as VMs with dual (Primary-model + Secondary) attachments are created and deleted across many different namespaces?

- **Phase:** 1
- **Owner:** osac-operator team
- **Impact:** Each Secondary VN subnet's Secondary UDN must track a potentially growing and changing set of namespaces (one per dual-attached VM's other namespace), rather than a single fixed namespace. Affects how the operator maintains the shared namespace label and how quickly a new namespace becomes selectable when a dual-attached VM is created.

### 7.3 How does the operator detect that live attachment failed or is unavailable, to trigger the recreation fallback (Risk 6.2)?

- **Phase:** 1
- **Owner:** osac-operator team
- **Impact:** Affects whether the operator probes for `multus-dynamic-networks-controller`'s presence once (e.g., checking for its deployment/CRDs at VirtualNetwork-type-Secondary provisioning time) versus attempting live attachment per subnet change and timing out on failure. A one-time capability probe avoids a timeout delay on every subnet change once it's known the mechanism isn't present, but needs to handle the controller being installed or removed after the initial check.

### 7.4 What is an acceptable timeout for the agent to wait for a live-attached interface to appear before the operator falls back to recreation?

- **Phase:** 1
- **Owner:** Networking / SRE
- **Impact:** Too short a timeout risks falling back to recreation (and its disruption) when live attachment would have succeeded slightly later; too long a timeout delays the subnet becoming usable when live attachment was never going to work for that cluster.

### 7.5 Is `multus-dynamic-networks-controller`, when deployed as part of OpenShift Virtualization, scoped to react only to virt-launcher pods, or will it also act on the router pod's own Multus annotation changes?

- **Phase:** 1
- **Owner:** Networking / osac-operator team
- **Impact:** Determines whether the primary mechanism actually engages for the router pod out of the box (if the controller reacts to any pod), or whether it silently never triggers and every subnet change takes the fallback path (if scoped to KubeVirt pods only) — in which case OSAC would need to deploy its own instance of the controller, or an equivalent, as new infrastructure. See Risk 6.7.

### 7.6 How is "no fabric manager configured" represented on a NetworkClass?

- **Phase:** Gates Phase 2 (must be resolved before Phase 2 begins)
- **Owner:** Networking / osac-operator team
- **Impact:** OSAC's existing Unified Networking model generally treats `fabricManager` as a required NetworkClass field, which is in tension with the premise that some deployments have none at all. This needs to be reconciled against the current NetworkClass API — whether this corresponds to a distinct NetworkClass configuration (e.g., a `udn-net`-style class predating that model) or something else — before the operator can reliably decide, at VirtualNetwork creation time, whether to provision the transit interface and bare-metal-connectivity machinery for a given Secondary VirtualNetwork. Note this determination is also needed in Phase 1, to decide when *not* to provision Phase 2 machinery — but it only has a functional consequence once Phase 2 exists.

### 7.7 What node-level network topology should be documented as the provisioning requirement for a fabric-manager-less NetworkClass's ExternalIPPool?

- **Phase:** 1
- **Owner:** Networking / infrastructure team
- **Impact:** Not a question of feasibility — ExternalIPPool CIDRs are already admin-provisioned per NetworkClass to match whatever mechanism makes them reachable, and this is simply that same step applied to EgressIP instead of fabric BGP/DNAT. The remaining work is to specify, in provisioning documentation, what node-level topology (e.g., a dedicated L2 segment or secondary NIC) a fabric-manager-less NetworkClass needs so its admin-chosen ExternalIPPool CIDRs are valid for EgressIP assignment.

## 8. Alternative Considered: BGP Daemon in the Router Pod (Phase 2)

Rather than operator-pushed static routes, the router pod could instead run its own BGP process (e.g., FRR) over the transit interface, peering directly with a fabric-side counterpart to advertise the VN's subnet CIDRs and learn the fabric's routes dynamically, without per-change operator involvement.

This was the initially favored approach during design discussion, and remains viable, but was set aside as the primary approach in this PRD because:

- It requires the fabric manager to accept a second, independent BGP neighbor relationship for the router pod, distinct from whatever it already uses for native EVPN attachment on the same transit network — a materially larger ask than accepting a static route through an API it already exposes.
- It introduces AS-numbering and route-target coordination that the static-route approach avoids entirely.
- It adds a live routing protocol and its associated packaging, configuration-reload, and convergence/restart-time behavior inside the router pod, none of which the static-route approach needs.
- Since subnet add/remove is already a fully operator-orchestrated event (creating the Secondary CUDN, updating the router pod's ConfigMap), the main advantage of BGP — publishing changes without explicit operator action — has limited value here, as the operator is already in the loop for every such change.

If a future requirement emerges for the fabric to learn routes independently of the operator's orchestration (for example, if subnet membership could change through a path the operator does not control), this alternative should be revisited.

## 9. Primary Mechanism: Live Interface Attachment via `multus-dynamic-networks-controller`, with Router Pod Recreation as Fallback (Phase 1)

To add a subnet to a Secondary VirtualNetwork without a router pod restart, the operator patches the router pod's Multus network-attachment annotation with the new Secondary UDN, relying on the upstream [`multus-dynamic-networks-controller`](https://github.com/k8snetworkplumbingwg/multus-dynamic-networks-controller) (a Kubernetes Network Plumbing Working Group project) to inject the corresponding interface directly into the running pod's network namespace. The same mechanism is used in reverse to remove an interface when a subnet is removed.

```text
Subnet added
     │
     ▼
Operator patches router pod's Multus annotation
     │
     ▼
multus-dynamic-networks-controller injects the interface   ──── not available / times out ────┐
     │                                                                                         │
     ▼ (within timeout)                                                                        ▼
Agent detects new interface, configures it live                          Operator recreates router pod
(gateway IP, routes, port security) — no restart                         with updated Secondary UDN set
     │                                                                    (Risk 6.2 — known, bounded disruption)
     ▼                                                                                         │
Subnet usable                                                            Subnet usable ◄────────┘
```

This is viable specifically for the router pod, for a reason that doesn't hold for generic application pods: the reason Red Hat does not document or support this mechanism for ordinary pods is that a standard application (e.g., an unmodified Nginx container) has no process watching for a newly-appeared network device and configuring it — the interface is injected at the infrastructure level, but nothing inside the container notices or uses it. The router pod's own agent (Section 2) is purpose-built software, not a generic application — it watches for a newly-injected interface and configures it (brings the link up, applies the IPAMClaim-reserved gateway IP, adds routes and port security) the moment it appears, filling exactly the gap that makes this unsupported for ordinary workloads.

**This mechanism is not an officially documented or supported OpenShift capability for non-VM pods**, which is why it is paired with an automatic fallback rather than being relied on unconditionally:

- Red Hat only documents and supports this class of behavior for VM pods via OpenShift Virtualization's own tooling (which additionally requires a live migration to attach the interface at the guest level — a different mechanism, not evidence that this upstream controller itself is production-supported for plain pods).
- Depending on it exclusively would mean depending on behavior outside Red Hat's OpenShift support SLA, with no guarantee of compatibility across OpenShift versions or continued upstream maintenance.

**Fallback:** if `multus-dynamic-networks-controller` is not installed/available in the target cluster, or the agent does not observe the new interface appear within a bounded timeout (Open Question 7.4), the operator recreates the router pod with the updated set of Secondary UDN attachments — the same recreation behavior described in Risk 6.2, using only officially supported mechanisms, at the cost of a brief, bounded interruption mitigated by persistent IPAMClaim addressing.

This gives the design a "best effort no-restart, always-works-eventually" shape: subnet add/remove is fast and non-disruptive where the upstream mechanism is present and functioning, and degrades gracefully to the known-safe recreation path everywhere else. Product/support should sign off on depending on the primary path given its unsupported status (Risk 6.6) before this ships as the default behavior.
