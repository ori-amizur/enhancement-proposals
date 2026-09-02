---
title: secondary-virtualnetwork-router-pod
authors:
  - oamizur@redhat.com
creation-date: 2026-08-26
last-updated: 2026-08-26
tracking-link:
  - TBD
prd: "prd.md"
see-also:
  - "/enhancements/vn-subnet-connectivity"
replaces:
  - N/A
superseded-by:
  - N/A
---

# Secondary VirtualNetwork: Router Pod Model with Bare-Metal Connectivity

## Summary

This enhancement adds a **Secondary** VirtualNetwork type: a single router pod, multi-homed across every subnet in the VirtualNetwork, that provides inter-subnet routing and internet egress from one VN-scoped point of control. It is delivered in two phases. **Phase 1** is fully self-contained and requires no fabric manager: the router pod, live subnet attach/detach with automatic fallback to recreation, and NAT Gateway support via OVN-Kubernetes EgressIP. **Phase 2** adds bare-metal fabric connectivity — a dedicated EVPN transit interface and operator-managed static routing — only for deployments that have a fabric manager configured. Tenants opt in via a type field on the VirtualNetwork that defaults to OSAC's existing networking model, so no current deployment is affected.

## Motivation

OSAC's existing VirtualNetwork networking model has no VN-scoped, single-router-pod option. Some deployments specifically want one router per VirtualNetwork — simpler to reason about and operate than per-subnet infrastructure. That capability is valuable on its own, with no dependency on a physical fabric (Phase 1), and becomes more valuable still when it can also reach bare-metal workloads on a fabric where one exists (Phase 2). This enhancement delivers both, in that order.

### User Stories

* **(Phase 1)** As a tenant, I want to choose a VN-scoped router-pod networking model for my VirtualNetwork instead of OSAC's existing Primary model, so I can use the architecture that best fits my workload.
* **(Phase 1)** As a tenant, I want VMs in different subnets of my VirtualNetwork to communicate with each other and reach the internet through a single router.
* **(Phase 1)** As a tenant, I want to add or remove subnets from my VirtualNetwork without disrupting connectivity for my other subnets.
* **(Phase 1)** As a tenant on a deployment without a fabric manager, I want to configure a NAT Gateway so my egress traffic uses a known, stable external IP.
* **(Phase 2)** As a tenant, I want VMs on my VirtualNetwork's subnets to reach bare-metal servers on the physical fabric, and vice versa.
* **(Phase 1)** As a provider, I want this new model to default to off, so existing tenants and deployments see no behavior change unless they explicitly opt in.
* **(Phase 1)** As a provider, I want subnet membership changes on the router pod to be reconciled automatically, without manual AAP/operator intervention per change.
* **(Phase 1)** As an SRE, I want to be able to tell, from pod status and events, whether a subnet was attached live or required a router pod recreation, so I can distinguish expected behavior from a regression.

### Goals

#### Phase 1

- Provide a VirtualNetwork type flag, defaulting to OSAC's existing model, that opts a VirtualNetwork into the router-pod model described here.
- Provide inter-subnet routing and internet egress through a single router pod per Secondary VirtualNetwork.
- Reconcile router pod state (gateway IPs, routes, port security) live from a ConfigMap, without a pod restart, for interfaces that already exist.
- Attach and detach subnets on the router pod without a pod restart when possible, falling back automatically to router pod recreation when not.
- Support NAT Gateway for fabric-manager-less Secondary VirtualNetworks via OVN-Kubernetes EgressIP, sourced from the existing ExternalIPPool, with no change to the router pod itself.

#### Phase 2

- Provide bare-metal fabric connectivity from the router pod via a dedicated EVPN transit interface and operator-managed static routing, only for deployments that have a fabric manager configured.

### Non-Goals

- Changes to OSAC's existing (Primary) VirtualNetwork networking model.
- Router pod high availability or multiple router pod instances per VirtualNetwork (either phase).
- A non-EVPN transit alternative (macvlan/SR-IOV attachment with external BGP/VRF-Lite) for Phase 2.
- A BGP daemon running inside the router pod for Phase 2 (see Alternatives).
- Migration tooling to convert a VirtualNetwork between networking types after creation.
- NAT Gateway integration for Secondary VirtualNetworks **that have a fabric manager configured** — delegated entirely to the fabric manager's own native SNAT mechanism, unaffected by this work. (NAT Gateway for the no-fabric-manager case is Phase 1, not a non-goal.)
- VPC peering integration for Secondary VirtualNetworks.

## Proposal

A Secondary VirtualNetwork is implemented as:

### Phase 1 (no fabric manager required)

1. **One namespace per VirtualNetwork**, shared by every subnet in it (as opposed to one namespace per subnet).
2. **One Secondary ClusterUserDefinedNetwork (CUDN) per subnet**, targeting the router pod's namespace — and any other namespace containing a VM that also attaches to that subnet — via a shared namespace label. Secondary UDNs, not Primary, are required because the router pod must be multi-homed across every subnet in the VN from a single namespace, and OVN-Kubernetes allows only one Primary UDN per namespace.
3. **One router pod per VirtualNetwork**, deployed by the osac-operator in the VN's namespace, with one interface per subnet acting as that subnet's default gateway (`.1`) and one interface on the cluster network for internet egress via SNAT.
4. **An agent running in the router pod** that watches a ConfigMap written by the operator and reconciles gateway IPs, routes, and port security on interfaces that already exist, and that detects and configures a newly live-attached interface when a subnet is added.
5. **A two-tier mechanism for subnet add/remove**: live interface attachment via `multus-dynamic-networks-controller` when available, falling back automatically to router pod recreation when it is not.
6. **NAT Gateway support** implemented via an OVN-Kubernetes **EgressIP** resource selecting the router pod's namespace, using the address from the tenant's NATGateway's ExternalIP — rather than via any fabric-side mechanism or a change to the router pod itself.

### Phase 2 (fabric manager required)

7. **One additional transit interface on the router pod** — a Primary CUDN with EVPN transport — used solely to reach the physical fabric's shared transit MAC-VRF, provisioned only when the target deployment has a fabric manager configured.
8. **Operator-managed static routing** between the router pod and the fabric manager for bare-metal reachability, rather than a routing protocol running inside the router pod.

When no fabric manager is configured, items 7–8 are never provisioned: the router pod remains an in-cluster-only gateway providing inter-subnet routing, internet egress, and NAT Gateway, and nothing else.

### Architecture

The diagram below shows a Secondary VirtualNetwork with **Phase 2 deployed** (a fabric manager configured). On **Phase 1 alone** (no fabric manager), the router pod has only its cluster-net and per-subnet interfaces — no transit interface, no EVPN, no physical fabric box — everything from the transit interface down does not exist.

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
   │           └───────────────┬───────────────┘                    │
   │                           │                                    │
   │                 ┌─────────▼──────────┐                         │
   │                 │     Router Pod      │                         │
   │                 │                     │                         │
   │                 │  Agent:             │                         │
   │                 │  watches ConfigMap, │                         │
   │                 │  configures gateway │                         │
   │                 │  IPs, routes, port  │                         │
   │                 │  security live, and │                         │
   │                 │  detects newly      │                         │
   │                 │  live-attached      │                         │
   │                 │  interfaces         │                         │
   │                 │                     │                         │
   │                 │  cluster-net iface  ├────────► Internet (SNAT,│
   │                 │                     │           or NAT Gateway│
   │                 │                     │           via EgressIP, │
   │                 │                     │           Phase 1)      │
   │  Phase 2 only ─▶│  transit iface      │                         │
   │                 │  (Primary CUDN,     │                         │
   │                 │   EVPN transport)   │                         │
   │                 └─────────┬───────────┘                         │
   └───────────────────────────┼─────────────────────────────────────┘
                                │  EVPN MAC-VRF (fabric transit network)
                                │           Phase 2 only, below this line
                     ┌──────────▼───────────┐
                     │   Physical Fabric      │
                     │   (e.g., Netris)       │
                     │                        │
                     │   Bare-Metal Server    │
                     │   10.0.3.9             │
                     └────────────────────────┘
```

### Workflow Description

**Actors:** tenant (creates VirtualNetworks/Subnets/ComputeInstances), osac-operator (reconciles all of the above, provisions and updates the router pod), router pod agent (applies ConfigMap-driven state inside the pod), fabric manager (Phase 2 only; e.g., Netris; hosts bare-metal subnets and accepts static routes).

#### VirtualNetwork creation (Phase 1, with a Phase 2 branch)

1. Tenant creates a VirtualNetwork with its type field set to Secondary; omitting the field defaults to OSAC's existing model.
2. The osac-operator resolves the target NetworkClass and checks whether it has a fabric manager configured.
3. The operator creates the VN's namespace and deploys the router pod into it, with the cluster-network interface and no subnet interfaces yet.
4. **(Phase 2) If a fabric manager is configured**, the operator additionally provisions the EVPN transit interface (see "Bare-metal connectivity setup" below) and reserves its address via IPAMClaim with `ipam.lifecycle: Persistent`. **(Phase 1) If no fabric manager is configured**, this step is skipped entirely — the router pod never gains a transit interface for this VirtualNetwork's lifetime, and none of the Phase 2 workflows below apply to it.

#### Subnet creation (Phase 1, live-attach path)

1. Tenant creates a Subnet referencing the VirtualNetwork.
2. The osac-operator creates the Secondary CUDN for the subnet, targeting the router pod's namespace (and any other namespace with a dual-attached VM, via the shared label).
3. The operator reserves the subnet's `.1` gateway IP via a persistent IPAMClaim, and patches the router pod's Multus network-attachment annotation to add the new Secondary UDN.
4. `multus-dynamic-networks-controller` injects the new interface into the running router pod's network namespace.
5. The router pod's agent detects the new interface (via periodic link inspection or netlink events) and configures it: brings the link up, assigns the reserved `.1` address, and updates its own routing table per the ConfigMap. Port security is patched separately by the operator's `cudn_net` role, not the agent — see Implementation Details ("Port security") and Open Question 10; this live-attach path itself remains blocked on `multus-dynamic-networks-controller` (Open Questions 1, 5, 6) and is not yet built.
6. The Subnet is marked Ready. No router pod restart occurred.

#### Subnet creation (Phase 1, fallback path)

1. Steps 1–3 as above.
2. If `multus-dynamic-networks-controller` is not installed/available in the target cluster (detected via a one-time capability probe at VirtualNetwork provisioning time — see Open Questions), or the agent does not observe the new interface within the configured timeout, the operator recreates the router pod with the full, updated set of Secondary UDN attachments.
3. Because gateway IPs and (Phase 2) the transit address are IPAMClaim-persistent, the recreated pod recovers the same addresses for all existing subnets; only MAC addresses change, requiring ARP reconvergence.
4. The Subnet is marked Ready once the recreated pod is configured.

#### Subnet removal (Phase 1)

Mirrors subnet creation: live detachment via `multus-dynamic-networks-controller` where available (agent removes the corresponding routing state as the interface disappears), falling back to router pod recreation without the removed attachment otherwise — the only path currently implemented (see Implementation Details, "Port security"). Either way, the operator's `cudn_net` role repatches OVN port security for every remaining attachment after detachment, since recreation invalidates every existing logical switch port, not just the removed one.

#### ComputeInstance (VM) creation (Phase 1)

1. Tenant creates a ComputeInstance with a subnet attachment.
2. The VM is created on the subnet's Secondary UDN, receiving its IP from OVN IPAM via DHCP.
3. If the VM also requires a Primary UDN elsewhere, it is created in that namespace instead of the router pod's namespace; the subnet's Secondary UDN already selects that namespace via the shared label (Implementation Details). Because the Primary attachment already owns the default route, cloud-init instead injects a route scoped to this Secondary VirtualNetwork's own CIDR via `.1` — reaching every subnet in the VN through the router pod's inter-subnet forwarding without overriding the Primary attachment's default route. DNS resolution for such a VM is provided entirely by its Primary attachment's existing mechanism — no DNS configuration is injected via the Secondary VN subnet.
4. Otherwise (no Primary UDN elsewhere), cloud-init instead injects a full default route to `.1`, plus DNS resolver configuration mirroring the router pod's own `/etc/resolv.conf` — the same file the router pod itself uses, read by the operator/agent and passed through unchanged (see "DNS resolution" in Implementation Details).

#### NAT Gateway configuration (Phase 1, no fabric manager)

1. Tenant allocates an ExternalIP from an ExternalIPPool, as usual.
2. Tenant creates a NATGateway resource referencing the Secondary VirtualNetwork and the ExternalIP.
3. The osac-operator, having already determined at VirtualNetwork creation time that this deployment has no fabric manager, implements the NATGateway by creating an OVN-Kubernetes `EgressIP` resource with the allocated address, selecting the router pod's namespace (and, for defense in depth, the router pod's own label) via `namespaceSelector`/`podSelector`.
4. No change is made to the router pod or its agent. The router pod's existing default-egress behavior (VM → `.1` → cluster-net interface, SNATed to the pod's own IP) is untouched.
5. OVN-Kubernetes assigns the EgressIP to a node capable of hosting that address and performs a second SNAT — from the router pod's own IP to the NATGateway's external IP — as traffic actually leaves the node. The NATGateway is marked Ready once the EgressIP reports `Assigned`.

```text
VM-A → router pod (.1) → out cluster-net interface, SNATed to router pod's own IP
  (existing default-egress behavior, unchanged)
  → OVN-Kubernetes EgressIP (matches router pod's namespace) → SNATed again to
    the NATGateway's ExternalIP → internet
```

Without a NATGateway configured, this second SNAT step does not exist — traffic uses the router pod's default egress path with a generic, shared source address. Only one NATGateway per VirtualNetwork is supported, consistent with the existing NATGateway resource's semantics elsewhere in OSAC.

#### Bare-metal connectivity setup (Phase 2)

1. When a bare-metal subnet on the fabric becomes relevant to the VirtualNetwork, the operator writes a route into the router pod's ConfigMap: `<bare-metal subnet CIDR> via <fabric peer's transit address> dev <transit interface>`. The agent applies it live.
2. The operator also pushes a static route into the fabric manager's existing route-management API: `<VN subnet CIDR> via <router pod's transit address>`, so bare-metal-originated traffic can reach back.
3. Both directions rely on the router pod's transit interface being a genuine EVPN member of the fabric's shared transit MAC-VRF — established once, at VirtualNetwork creation, and unaffected by individual subnet changes.

```text
VM → bare-metal:
VM → router pod (.1) → transit interface → EVPN MAC-VRF → fabric → bare-metal server

Bare-metal → VM:
bare-metal server → fabric's local gateway → static route (pushed by operator)
  → router pod's transit interface → router pod → VM
```

### API Extensions

- **VirtualNetwork**: adds a type/mode field (name TBD, e.g. `spec.networkingType`), enum with `Primary` and `Secondary`, immutable after creation, defaulting to `Primary`. This is the only tenant-facing API surface change, and is delivered in Phase 1. Note: `Primary` here names OSAC's existing VirtualNetwork type — it is unrelated to OVN-Kubernetes's own Primary/Secondary UDN distinction, which this feature's router pod uses internally regardless of which VirtualNetwork type a tenant chooses (Section 1's foundation mechanism always uses Secondary UDNs for the router pod, whether the VirtualNetwork itself is `Primary` or `Secondary`).
- **Subnet, ComputeInstance**: no spec changes. Existing fields (`network_attachments`, etc.) work unchanged; provisioning-side behavior differs based on the parent VirtualNetwork's type.
- **NATGateway**: no spec changes — the existing `virtual_network`/`external_ip` fields are reused as-is (Phase 1). Only the operator's internal implementation differs for a fabric-manager-less target VirtualNetwork (EgressIP instead of a fabric-side SNAT rule).
- The router pod, its ConfigMap, the transit CUDN (Phase 2), the Secondary CUDNs' shared-label targeting, and the EgressIP resource backing a fabric-manager-less NATGateway (Phase 1) are internal resources managed by the operator — not exposed in the tenant API.
- No new CRDs. `multus-dynamic-networks-controller`'s own resources (Phase 1) and OVN-Kubernetes's existing `EgressIP` CRD (Phase 1) are external dependencies, not OSAC API extensions.

### Implementation Details/Notes/Constraints

#### Phase 1

- **Namespace-per-VN and shared-label targeting**: every Secondary CUDN for a subnet in the VN targets a `namespaceSelector` matching a shared label, not a single fixed namespace, so that a VM requiring a Primary UDN elsewhere (and therefore living in a different namespace) can still attach to a Secondary VN's subnet. The operator maintains this label set as VMs with such dual attachments are created and deleted (Open Questions).
- **Gateway IP persistence**: uses IPAMClaim with `ipam.lifecycle: Persistent`, so the router pod (whether reconciled live or recreated) recovers the same addresses. MAC addresses are not persisted and change on recreation, requiring ARP reconvergence (Risks).
- **Port security (implemented as a temporary stopgap, see Open Question 10)**: OVN port security on the router pod's logical switch ports must be patched to allow forwarding of packets with destination IPs outside the router's own address, since the router pod forwards on behalf of every VM in the VN. Investigated during implementation: OVN-Kubernetes's UDN/CUDN API exposes no per-pod or per-port mechanism to relax this declaratively (the only related field, `ipam.mode: Disabled`, disables port security for the entire logical switch, not just the router pod's port, and conflicts with this design's DHCP-based VM addressing). The only mechanism is direct OVN northbound database manipulation (`ovn-nbctl` against the `nbdb` container in `ovnkube-node`, `openshift-ovn-kubernetes` namespace), which requires cluster-admin-equivalent `pods/exec` access into a system namespace and has no tenant scoping at the OVN layer (a single credential can read/write every tenant's logical switches, not just the router pod's own) — a real, accepted, and deliberately temporary tradeoff, not a resolved problem. Implemented in the `cudn_net` Ansible role (`patch_router_pod_port_security.yaml`/`patch_router_pod_lsp.yaml`), reusing the same credential that role already uses for all other subnet/router-pod provisioning (`osac-sa`, already cluster-admin in the default single-cluster deployment; a new, explicitly documented RBAC prerequisite for multi-cluster deployments — see `osac-operator/docs/vmaas-dedicated-cluster/README.md`), rather than a new isolated component: a narrowly-scoped isolated implementation was designed and then rejected once it became clear `osac-sa`'s existing cluster-admin binding — and the fact that the multi-cluster remote-cluster kubeconfig is entirely deployer-supplied with no OSAC-defined scope — meant that kind of isolation would not reliably contain anything AAP itself can already do, making the added complexity not worth it for something explicitly expected to be replaced. Runs after every subnet create/delete (router pod recreation invalidates every existing logical switch port, not just the changed one, so every current attachment is repatched each time), and never trusts a computed logical switch port name outright: OVN-Kubernetes's UDN naming formula is an internal, unexported implementation detail with no compatibility guarantee, so the implementation queries for ports by the stable `<namespace>_<podname>` name suffix, cross-checks the count against the router pod's current attachments, and only patches ports it can independently confirm via the pod's own Multus `network-status` annotation — anything that doesn't cleanly match is a hard failure, not a best-effort guess. Not verifiable against a live cluster in this repo's test environment (no OVN-Kubernetes control plane available) — the `ovnkube-node` pod label/selector and the matching logic must be confirmed against a real cluster before this is considered production-ready.
- **Cloud-init dependency and route scope**: Secondary UDNs do not provide a DHCP-supplied gateway option, so the VM's route to `.1` is injected via cloud-init, and its scope depends on whether the VM is dual-attached. A VM with only Secondary VN subnets gets a full default route to `.1`. A VM with a Primary UDN attachment elsewhere keeps that attachment's default route untouched and instead gets a route scoped to this Secondary VirtualNetwork's own CIDR via `.1` — since Subnets are non-overlapping carve-outs of the VN's CIDR, this single route reaches every subnet in the VN through the router pod's inter-subnet forwarding, while the VM's own directly-attached subnet is still reached via its normal connected route (more specific, so it always takes precedence over the VN-CIDR route). VMs without cloud-init support have no route to `.1` at all and lose inter-subnet and (for Secondary-VN-only VMs) egress connectivity (they can still reach OVN's own IPAM-assigned services on their local subnet).
- **DNS resolution**: a VM has exactly one `/etc/resolv.conf`, so the two cases from "VMs that also need network access outside this VirtualNetwork" (Section 1) are handled differently. A VM with a Primary UDN attachment elsewhere gets its DNS configuration entirely from that attachment's existing mechanism — nothing is injected for the Secondary VN subnet, to avoid conflicting with it. A VM with only Secondary VN subnets has DNS resolver configuration injected via the same cloud-init mechanism as the default route: the operator (or agent) reads the router pod's own `/etc/resolv.conf` — an ordinary pod using the default Kubernetes DNS policy, so it already contains the cluster's CoreDNS ClusterIP and standard search domains — and passes that content through unchanged into the VM's cloud-init network-config. This works because DNS queries the VM sends are forwarded through the router pod's cluster-network interface the same way any other non-inter-subnet, non-bare-metal traffic is (see the SNAT note below), and Kubernetes Service routing (OVN's destination-based ClusterIP DNAT) applies to that forwarded traffic the same way it would to traffic the router pod originated itself — this needs confirming for genuinely forwarded (not pod-process-originated) traffic (Open Questions).
- **Cluster-network egress SNAT is unconditional, not destination-dependent**: the router pod's cluster-network interface is an ordinary OVN logical switch port with default port security, which only allows egress traffic whose source IP matches the port's own registered address. Any traffic the router pod forwards out that interface — whether destined to a cluster-internal Service ClusterIP or to a genuinely external address — must first be SNATed to the router pod's own pod IP, or OVN drops it at the port. This SNAT rule should be a single, uniform "rewrite source to my own IP" applied to everything leaving via that interface, not something that tries to distinguish internal from external destinations first. Once SNATed, the packet is indistinguishable from any other traffic the router pod's own process might have sent, and the existing cluster networking stack already handles both cases correctly without any further help from the router pod: OVN's own Service load-balancing applies to cluster-internal destinations, and the node's pre-existing external-masquerade rule (which already excludes cluster-internal CIDRs, a prerequisite for basic cluster networking generally) applies to genuinely external ones. No additional exclusion logic is needed in the router pod's own rules.
- **The agent never discovers subnet state on its own.** The operator is the authoritative source of every subnet's existence, CIDR, and type — it is the controller that provisions the `Subnet` resource in the first place — so the agent's role is purely to apply whatever the operator has written to the ConfigMap: gateway IPs, per-subnet routes, and port-security patches (plus, in Phase 2, fabric-reachable routes). It does not query the fulfillment-service or watch `Subnet` resources itself.
- **Live attachment mechanism**: the operator patches the router pod's Multus network-attachment annotation to add or remove a Secondary UDN reference; `multus-dynamic-networks-controller` (an upstream Kubernetes Network Plumbing Working Group project, not an OpenShift-documented capability for non-VM pods — see Risks) observes the annotation change and injects/removes the interface in the running pod's network namespace. The agent is responsible for noticing the interface and completing its configuration — the controller only handles the netlink-level attachment, not IP/route/port-security setup.
- **Fallback trigger**: the operator detects that live attachment is unavailable either through a one-time capability probe (checking for the controller's deployment/CRDs when a VirtualNetwork is first provisioned as Secondary) or by timing out a per-change wait for the new interface to appear (Open Questions cover the trade-off between these).
- **NAT Gateway via EgressIP**: implemented entirely as an OVN-Kubernetes `EgressIP` resource selecting the router pod's namespace (and pod, for defense in depth), with no changes inside the router pod or its agent. This relies on VMs never having a cluster/pod-network presence (they attach only to their subnet's Secondary UDN), so the router pod is the only entity in its namespace an EgressIP selector could match. The ExternalIPPool address assigned to the NATGateway must be hostable on some node's own network — this is not a new requirement introduced by this design, but the same existing admin-provisioning pattern already used for every NetworkClass's ExternalIPPool (CIDRs chosen to suit that NetworkClass's actual routing mechanism), applied here to node-level EgressIP assignment instead of fabric BGP/DNAT (Open Questions).

#### Phase 2

- **Fabric-manager-conditional provisioning**: the operator resolves the VirtualNetwork's target NetworkClass and provisions the transit interface, EVPN attachment, and all static-route dispatch only if that NetworkClass has a fabric manager configured; otherwise the router pod stays a Phase-1-only, in-cluster gateway. This is a provisioning-time decision made once, at VirtualNetwork creation — it cannot change over the VirtualNetwork's lifetime without recreating it, since it determines whether the transit interface exists at all. The exact NetworkClass field/representation used to determine "no fabric manager" needs to be confirmed against the current NetworkClass API (Open Questions) — OSAC's existing Unified Networking model generally treats `fabricManager` as a required NetworkClass field, so this may correspond to a distinct NetworkClass configuration (e.g., a pure `udn-net`-style class) rather than an optional/empty value on the same field other NetworkClasses use.
- **Transit address persistence**: uses IPAMClaim with `ipam.lifecycle: Persistent`, so the router pod recovers the same transit address across restarts and reschedules.
- **Bare-metal transit interface**: a Primary CUDN with EVPN transport, MAC-VRF only (no IP-VRF), reaching a transit MAC-VRF shared with the fabric. This interface is created once at VirtualNetwork creation and is not affected by individual subnet changes.
- **Static route dispatch**: the operator pushes/withdraws fabric-side routes through the fabric manager's existing route-management API (the same mechanism already used for VPCs, routes, and NAT rules) as part of the same reconciliation loop that updates the router pod's ConfigMap, so the two stay consistent (Risks).

### Risks and Mitigations

| Risk | Phase | Impact | Mitigation |
|------|-------|--------|------------|
| Router pod is a SPOF, no HA in this phase | Both | All connectivity for the VN lost during pod failure | Kubernetes restarts quickly; HA explicitly deferred |
| Live attachment depends on unsupported upstream behavior (`multus-dynamic-networks-controller`) | 1 | Not covered by Red Hat's OpenShift support SLA; behavior not guaranteed across versions | Automatic fallback to recreation ensures subnet add/remove still works if this regresses or is absent; requires explicit product/support sign-off before shipping as default |
| The controller may be present (via OpenShift Virtualization) but scoped to KubeVirt pods only, never triggering for the router pod | 1 | Primary mechanism silently never engages; every subnet change takes the fallback (recreation) path even though the controller is "installed" | Verify the controller's actual scoping/RBAC before relying on it; the fallback still makes the feature correct, just without the intended no-restart benefit (Open Question 6) |
| Fallback recreation changes the router pod's MAC addresses | 1 | ARP reconvergence needed across every subnet in the VN | Persistent IPAMClaim keeps IPs stable; MAC change is a known, bounded cost, same as any pod restart |
| Cloud-init dependency for the VM's route to `.1` | Both | VMs without cloud-init have no route to `.1` (default route, or VN-CIDR-scoped route for dual-attached VMs) | Document requirement; no automated mitigation in this phase |
| OVN NB port-security patching has no declarative, tenant-safe implementation path today (confirmed during implementation, Open Question 10) — implemented anyway, as an accepted temporary tradeoff | 1 | The credential the `cudn_net` role already uses for all subnet/router-pod provisioning can read/write any tenant's OVN logical switches, not just the router pod's own; requires a new documented RBAC prerequisite for multi-cluster deployments, without which this step fails outright | Implemented via direct OVN NBDB manipulation (see Implementation Details, "Port security"), reusing the existing AAP credential rather than a new isolated component (an isolated design was considered and rejected — see Open Question 10). Query-and-match logic (not a blindly-trusted computed port name) converts a naming-scheme drift into a hard failure rather than a silent wrong patch. Track upstream for a declarative, tenant-scoped OVN-Kubernetes mechanism (e.g., a CRD field) to replace this |
| A fabric-manager-less NetworkClass's ExternalIPPool is provisioned with CIDRs unsuited to node-level EgressIP assignment | 1 | The EgressIP backing a NATGateway never reaches `Assigned`; the NATGateway never becomes Ready | Same admin-provisioning discipline already required for any NetworkClass's ExternalIPPool, just against a different mechanism; document the required node-level topology for this NetworkClass type (Open Questions) |
| Static routes can drift from the router pod's ConfigMap if the fabric-side push fails independently | 2 | VM subnet unreachable from the fabric, or vice versa, without an obvious symptom | Treat fabric route push/withdraw as part of the same reconciliation loop as the ConfigMap update, with retry/requeue on failure |
| Fabric-side route changes are not instantaneous | 2 | Brief propagation delay after a subnet change | Acceptable — subnet changes are already operator-paced, not sub-second |
| EVPN Type-2 (MAC) re-advertisement after router pod recreation | 2 | Transit interface's next-hop briefly unreachable at L2 | Same reconvergence cost as any pod recreation; consider gratuitous ARP on startup if unacceptable |

### Drawbacks

- Depends on an unsupported upstream mechanism (`multus-dynamic-networks-controller`) for its headline "no restart" property (Phase 1); the officially-supported fallback (recreation) does not fully eliminate disruption.
- No HA in either phase — a router pod failure is a real, if brief, outage for the whole VirtualNetwork.
- Introduces a second VN networking model to operate and support alongside OSAC's existing one, with its own failure modes (cloud-init dependency, IPAMClaim persistence) that must be understood independently.
- OVN NB port-security patching (required for the router pod to forward on behalf of other VMs' addresses) has no declarative, tenant-safe implementation path today and is implemented via direct OVN northbound database manipulation as an accepted, temporary tradeoff (see Risks, Open Question 10) — reusing the same broad credential the `cudn_net` role already has for all other subnet/router-pod provisioning, rather than a new isolated component (considered and rejected, see Open Question 10).

## Alternatives (Not Implemented)

### BGP daemon in the router pod (Phase 2 alternative)

Rather than operator-pushed static routes for bare-metal reachability, the router pod could run its own BGP process (e.g., FRR) over the transit interface, dynamically advertising and learning routes. Not adopted because it requires the fabric manager to accept a second, independent BGP neighbor relationship (a materially larger ask than an existing static-route API), introduces AS-numbering/route-target coordination, and adds a live routing protocol's packaging and convergence-time behavior — all to solve a discovery problem that doesn't exist here, since subnet add/remove is already a fully operator-orchestrated event.

### Always recreate the router pod on subnet change (Phase 1 alternative — no live-attachment attempt)

Simpler to implement and fully within Red Hat's supported OpenShift capabilities, but reintroduces a restart-driven disruption on every subnet change, which this enhancement's live-attachment/fallback design avoids whenever the upstream mechanism is functioning.

### Non-EVPN transit (Phase 2 alternative — macvlan/SR-IOV + external BGP/VRF-Lite)

An alternative way to reach the fabric that avoids depending on EVPN support for the transit CUDN, at the cost of requiring node-level VLAN/SR-IOV provisioning and a routing-protocol session from the router pod. Deferred as a possible future alternative if EVPN-based transit proves impractical.

## Open Questions

1. **(Phase 1) Capability probe vs. per-change timeout**: should the operator detect `multus-dynamic-networks-controller`'s availability once (at VirtualNetwork provisioning time) or attempt live attachment per subnet change and time out on failure? A one-time probe avoids a timeout delay once absence is known, but must handle the controller being installed or removed later.
2. **(Phase 1) Timeout value**: what is an acceptable wait for the agent to observe a live-attached interface before the operator falls back to recreation?
3. **(Phase 2) EVPN reconvergence time**: after a router pod recreation, how long does the fabric take to relearn the transit interface's new MAC via EVPN Type-2 advertisement, and is that gap acceptable without additional mitigation (e.g., gratuitous ARP on startup)?
4. **(Phase 1) Namespace-label scale**: how does the shared-label mechanism for dual-attached VMs' namespaces perform and stay consistent as such VMs are created and deleted across a growing, changing set of namespaces?
5. **(Phase 1) Support sign-off**: is depending on `multus-dynamic-networks-controller` as the primary (non-fallback) mechanism acceptable to Red Hat support/product, given it is not an officially documented OpenShift capability for non-VM pods?
6. **(Phase 1) Scoping of the controller when deployed via OpenShift Virtualization**: OSAC already requires OpenShift Virtualization for VMaaS, and OpenShift Virtualization's own VM interface hot-plug feature is built on `multus-dynamic-networks-controller` — so the controller is plausibly already running on every cluster relevant to this feature, without OSAC needing to deploy it separately. However, it may be scoped (via RBAC or its own configuration) to react only to KubeVirt-managed (virt-launcher) pods, not arbitrary pods like the router pod. This needs to be verified directly against the controller's actual deployment/RBAC configuration before assuming the primary mechanism will actually trigger for the router pod — presence in the cluster is not the same as applicability to a non-VM pod's annotation change.
7. **(Gates Phase 2) How "no fabric manager" is represented on a NetworkClass**: OSAC's existing Unified Networking model generally treats `fabricManager` as a required NetworkClass field, which is in tension with the premise that some deployments have none at all. This needs to be reconciled against the current NetworkClass API — whether "no fabric manager" is a distinct NetworkClass configuration (e.g., a `udn-net`-style class predating that model) or something else — before the operator can reliably decide whether to provision the transit interface for a given Secondary VirtualNetwork. This determination is exercised in Phase 1 too (to decide when *not* to provision Phase 2 machinery), but only has a functional consequence once Phase 2 exists.
8. **(Phase 1) What node-level network topology should be documented as the provisioning requirement for a fabric-manager-less NetworkClass's ExternalIPPool?** Not a feasibility question — ExternalIPPool CIDRs are already admin-provisioned per NetworkClass to suit whatever mechanism makes them reachable, and this is that same step applied to EgressIP instead of fabric BGP/DNAT. What remains is specifying, in provisioning documentation, the topology needed (e.g., a dedicated L2 segment or secondary NIC) for a fabric-manager-less NetworkClass's ExternalIPPool CIDRs to be valid for EgressIP assignment.
9. **(Phase 1) Does OVN's Service ClusterIP DNAT apply correctly to traffic forwarded through a pod's interface, not just traffic that pod's own process originates?** This determines whether mirroring the router pod's own `/etc/resolv.conf` onto Secondary-VN-only VMs (DNS resolution, Implementation Details) actually resolves cluster-internal DNS queries correctly. Expected to work, since OVN's Service load-balancing is destination-based and applies as packets traverse the logical topology regardless of source, but this should be verified directly rather than assumed before relying on it.
10. **(Phase 1) OVN NB port-security patching has no declarative, tenant-scoped implementation path today — implemented anyway, as an explicit, accepted, temporary tradeoff.** Investigated directly: OVN-Kubernetes's UDN/CUDN API (`Layer2Config`, `ClusterUserDefinedNetworkSpec`) exposes no per-pod or per-port field to relax port security — the only related knob, `ipam.mode: Disabled`, disables it cluster-wide for the whole logical switch (breaking DHCP-based addressing for every VM on the subnet, not just the router pod, which conflicts with this design's core addressing model). The only mechanism is `ovn-nbctl` against the `nbdb` container inside `ovnkube-node` pods (`openshift-ovn-kubernetes` namespace), which requires `pods/exec` access into a system namespace that Red Hat's own documentation gates at the cluster-admin level and grants no tenant scoping once held (any principal with this access can rewrite *any* tenant's logical switches, not just the router pod's own port).

    A narrowly isolated implementation was designed — a dedicated, minimal component holding this credential alone, triggered on-demand (a `batch/v1 Job`, not a standing pod, to minimize how long the credential is ever "hot"), invoked by osac-operator rather than AAP — and then rejected once investigation showed it wouldn't reliably deliver the isolation it was meant to: `osac-sa` (the ServiceAccount AAP job pods run as in the default single-cluster deployment) is already bound to `cluster-admin` (`osac-aap/config/base/osac-sa.yaml`), and the multi-cluster deployment's remote-cluster kubeconfig is entirely deployer-supplied with no OSAC-defined scope (`osac-operator/docs/vmaas-dedicated-cluster/README.md`) — so RBAC-level isolation built *around* AAP's own credential doesn't reliably contain anything AAP itself can already do in either deployment mode. Moving the logic into osac-operator instead of AAP was also considered and rejected: it doesn't help the multi-cluster case (both components would share the same deployer-supplied kubeconfig), and it's worse for the single-cluster case, since osac-operator's own ClusterRole doesn't have `pods/exec` today — adding it there would be a brand-new, permanent capability on the one component OSAC fully owns, expanding what a bug in any of its other resource-type reconcilers could reach, whereas AAP already has this power unconditionally.

    Given this is explicitly an interim measure expected to be replaced once a real mechanism exists, the decision was to implement it as simply as possible: directly in the `cudn_net` Ansible role, reusing the same credential that role already uses for all other subnet/router-pod provisioning, rather than building the isolated component. This is a deliberate, informed acceptance of the tenant-isolation risk described above, not a resolved problem. Implementation detail carried over from investigation: the OVN-Kubernetes logical switch port naming formula for a UDN/secondary attachment (`go-controller/pkg/util`) is an internal, unexported implementation detail with no compatibility guarantee — the implementation queries for the port by its stable `<namespace>_<podname>` name suffix and cross-checks the match via the pod's own Multus `network-status` annotation rather than trusting a computed name outright, so a naming-scheme drift causes a loud failure rather than a silent wrong patch.

    **Resolution requires something outside OSAC's own control**: either an upstream OVN-Kubernetes feature exposing a declarative, tenant-scoped way to relax port security per port/pod (e.g., a CRD field), or a different network design for inter-subnet forwarding within a Secondary VN that doesn't need it. Track as a follow-up (upstream feature request and/or a future design revision) to replace the current implementation, not as unfinished Phase 1 scope.

## Test Plan

### Unit Tests

**Phase 1:**
- VirtualNetwork type field validation and immutability.
- Router pod ConfigMap generation: correct gateway IPs, per-subnet routes.
- ConfigMap update on subnet add/remove.
- Fallback-trigger logic (simulated absence/timeout of `multus-dynamic-networks-controller`).

**Phase 2:**
- Router pod ConfigMap generation: fabric-reachable routes.

### Integration Tests

**Phase 1:**

1. Create a Secondary VirtualNetwork on a NetworkClass with no fabric manager configured → verify namespace and router pod created with the cluster-net interface only, no transit interface.
2. Create two Subnets → verify Secondary CUDNs created; verify live attachment succeeds when the controller is present (no router pod restart observed) and falls back to recreation when it is not.
3. Create VMs in different subnets → verify IP assignment and default-route configuration via cloud-init.
4. Verify inter-subnet connectivity (VM-A ↔ VM-B via the router pod) and internet egress (SNAT).
5. Remove a subnet → verify live detachment (or fallback recreation) and that remaining subnets are unaffected.
6. Create a VM requiring both a Primary UDN (elsewhere) and a Secondary VN subnet → verify it is placed in its own namespace, gets a route scoped to the VN's own CIDR via `.1` (not a default route) rather than the full default route a Secondary-VN-only VM gets, still reaches the Secondary VN subnet via label-based targeting, and its Primary attachment's own default route is unaffected.
7. Allocate an ExternalIP and create a NATGateway → verify an EgressIP resource is created selecting the router pod's namespace, reaches `Assigned`, and VM egress traffic is observed with the NATGateway's external IP as its source, with no change to the router pod's own configuration.
8. Delete the NATGateway → verify the EgressIP resource is removed and egress traffic reverts to the router pod's default (generic, shared) SNAT path.

**Phase 2:**

9. Create a Secondary VirtualNetwork on a NetworkClass with a fabric manager configured → verify namespace and router pod created with cluster-net and transit interfaces.
10. Configure a bare-metal subnet route → verify VM ↔ bare-metal connectivity in both directions.
11. Verify Phase 1 behavior (inter-subnet, egress, NAT Gateway if configured) is unaffected on a VirtualNetwork that also has Phase 2 bare-metal connectivity configured.

### Edge Cases

- Router pod failure → verify restart and connectivity resumption, including ARP reconvergence. (Both phases)
- `multus-dynamic-networks-controller` becomes unavailable mid-lifecycle (e.g., uninstalled) → verify subsequent subnet changes correctly fall back to recreation. (Phase 1)
- NATGateway's ExternalIP is not routable on any node's network → verify the EgressIP remains unassigned and the NATGateway surfaces a clear, actionable status rather than silently appearing Ready. (Phase 1)
- VirtualNetwork deletion → verify cleanup order (router pod, namespace; plus, where applicable, transit CUDN and fabric-side static routes for Phase 2, and the EgressIP resource for a Phase 1 NATGateway). (Both phases)

## Graduation Criteria

- **Dev Preview (Phase 1)**: single Secondary VirtualNetwork with two subnets, inter-subnet connectivity and egress validated; fallback-recreation path validated even if live attachment is not yet reliable.
- **Tech Preview (Phase 1)**: live-attachment path validated across supported OpenShift versions, NAT Gateway via EgressIP validated, subnet add/remove lifecycle automated, test coverage in place.
- **GA (Phase 1)**: HA story resolved or explicitly and durably documented as a limitation; Red Hat support/product sign-off obtained for the `multus-dynamic-networks-controller` dependency (or the design has moved to a fully-supported mechanism by then).
- **Dev Preview (Phase 2)**: bare-metal connectivity validated end-to-end on a single fabric-manager-backed Secondary VirtualNetwork, built on a GA (or later-stage) Phase 1.
- **Tech Preview / GA (Phase 2)**: static-route dispatch validated across fabric manager upgrades/restarts, EVPN reconvergence behavior (Open Question 3) characterized and documented.

## Upgrade / Downgrade Strategy

The VirtualNetwork type field is additive and defaults to OSAC's existing model — no existing VirtualNetwork changes behavior on upgrade, in either phase. Downgrading a cluster below the version that introduced Phase 1 would leave existing Secondary VirtualNetworks unmanaged; a documented manual migration (recreate as the Primary type) would be needed, since automated conversion between types is out of scope. Downgrading below the version that introduced Phase 2 (while keeping Phase 1) would leave existing bare-metal-connected VirtualNetworks without their transit interface/static routes; this should degrade to Phase 1 behavior (in-cluster connectivity preserved, bare-metal reachability lost) rather than breaking the VirtualNetwork entirely — to be validated as part of Phase 2's own test plan.

## Version Skew Strategy

The osac-operator must handle `multus-dynamic-networks-controller` being at a different version than expected, or absent entirely, purely through the capability-probe/fallback mechanism (Open Questions) — no hard version dependency is assumed. No kubelet or CRI/CNI-level version coordination beyond what the controller itself requires is introduced by this design. Phase 2 introduces no additional version-skew surface beyond the fabric manager's own existing route-management API compatibility.

## Support Procedures

**Phase 1:**
- **Router pod not Ready**: check for OVN port-security patch failures or IPAMClaim conflicts in operator logs.
- **Subnet stuck attaching**: check whether the fallback path (recreation) triggered as expected, or whether `multus-dynamic-networks-controller` is installed but not functioning — this should be distinguishable via router pod events.
- **NATGateway stuck**: check the backing EgressIP resource's status; `Assigned` failures indicate an ExternalIPPool/node-topology mismatch (Risks).

**Phase 2:**
- **Bare-metal unreachable**: verify the static route exists on both the fabric manager's side and the router pod's ConfigMap/routing table; verify the EVPN transit interface's Type-2 advertisement is current (no stale MAC from a prior pod instance).
- **VirtualNetwork deletion stuck**: verify fabric-side static routes were withdrawn before namespace/pod teardown; a phased finalizer (routes → router pod → namespace) is expected, mirroring the ordering already used elsewhere in OSAC's networking controllers.

## Infrastructure Needed

**Phase 1:** no new infrastructure beyond existing AAP/dispatcher patterns. `multus-dynamic-networks-controller` itself is not something OSAC needs to separately deploy in the common case: since OSAC already requires OpenShift Virtualization for VMaaS, and OpenShift Virtualization's own VM interface hot-plug feature depends on this same controller, it is plausibly already present on every cluster relevant to this feature. What remains to be confirmed (Open Question 6) is whether that instance of the controller is scoped to react only to KubeVirt-managed pods or will also act on the router pod's own Multus annotation changes — if it is scoped to KubeVirt pods only, the primary mechanism would need its own, separately-deployed instance of the controller (or an equivalent), which *would* then be new infrastructure to provision.

**Phase 2:** the fabric manager's existing route-management API, already required infrastructure for any fabric-manager-backed NetworkClass, with no additional infrastructure beyond it.
