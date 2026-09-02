# Epic 1: Secondary VirtualNetwork Foundation

- **T-Shirt Size:** L
- **Phase:** 1
- **PRD Requirements:** FR-1, FR-2, FR-3, FR-4, FR-5, FR-6, FR-14, FR-15, FR-21, FR-22
- **Design Reference:** design.md §Proposal (Phase 1, items 1–3), §Workflow Description ("VirtualNetwork creation," "Subnet creation (fallback path)," "ComputeInstance (VM) creation"), §Implementation Details/Notes/Constraints (Phase 1)
- **Dependencies:** None

## Summary

Delivers a fully functional Secondary VirtualNetwork: the opt-in type flag, a single router pod per VN providing inter-subnet routing and internet egress, and subnet add/remove via router pod recreation (the safe baseline that Epic 2 later optimizes to avoid restarts). This epic alone is independently valuable — a tenant gets working multi-subnet connectivity with no dependency on any later epic.

## Acceptance Criteria

- [ ] A VirtualNetwork can be created with its networking type set to Secondary; omitting it defaults to Primary, and existing VirtualNetworks are unaffected.
- [ ] The type field is immutable after creation.
- [ ] A Secondary VirtualNetwork provisions exactly one namespace and one router pod, with one Secondary UDN attachment per subnet.
- [ ] A VM with a Primary UDN attachment elsewhere in addition to a Secondary VN subnet is placed in its own namespace, not the router pod's, and still reaches the Secondary VN subnet.
- [ ] VMs in different subnets of the same Secondary VirtualNetwork communicate with each other via the router pod's `.1` gateway.
- [ ] VMs reach the internet via the router pod's SNAT egress path.
- [ ] Adding or removing a subnet is reflected in the router pod (via recreation) without disrupting unrelated VirtualNetworks.
- [ ] A deployment with no fabric manager configured never provisions a transit interface or any fabric-side machinery for its Secondary VirtualNetworks.
- [ ] A VM with only Secondary VN subnets has DNS resolver configuration mirroring the router pod's own `/etc/resolv.conf`; a VM with a Primary UDN attachment elsewhere is unaffected (its Primary attachment's own DNS configuration applies).
- [ ] A VM with a Primary UDN attachment elsewhere reaches every subnet in the Secondary VN via a cloud-init-injected route scoped to the VN's own CIDR through `.1`, without its Primary attachment's default route being overridden.

## Stories

### Story 1.01: [DEV] VirtualNetwork networking-type field

**As a** tenant,
**I want to** set a networking type on my VirtualNetwork,
**So that** I can opt into the router-pod model instead of OSAC's Primary model.

**Acceptance Criteria:**
- [ ] `VirtualNetwork` gains a networking-type field (enum, e.g. `Primary` | `Secondary`), defaulting to `Primary` when omitted.
- [ ] The field is immutable — an update attempting to change it after creation is rejected with a clear error.
- [ ] Existing VirtualNetworks (created before this field existed) behave as `Primary`.

**Implementation Guidance:** Add the field to the VirtualNetwork proto/CRD spec (name TBD per design.md §API Extensions, e.g. `spec.networkingType`). Enforce immutability the same way other immutable VirtualNetwork fields are already enforced in the fulfillment-service/osac-operator validation webhook or admission logic. No behavior change for `Primary` — this story only adds the field and its validation, not the router-pod provisioning logic (Story 1.02).

**Testing Approach:** Unit tests for field defaulting, validation, and immutability rejection. No integration test needed yet — provisioning behavior is exercised starting in Story 1.02's tests.

**Dependencies:** None

---

### Story 1.02: [DEV] Router pod and namespace provisioning for Secondary VirtualNetworks

**As a** tenant with a Secondary VirtualNetwork,
**I want to** have a single namespace and router pod provisioned automatically,
**So that** my VN has a working gateway without manual setup.

**Acceptance Criteria:**
- [ ] Creating a VirtualNetwork with networking type `Secondary` provisions exactly one namespace and one router pod within it.
- [ ] The router pod starts with only a cluster-network interface (no subnet interfaces) until subnets are added.
- [ ] Deleting the VirtualNetwork cleans up the namespace and router pod.
- [ ] A `Primary`-type VirtualNetwork triggers none of this provisioning.

**Implementation Guidance:** Extend the osac-operator's VirtualNetwork reconciler to branch on the networking-type field (Story 1.01) and, for `Secondary`, create the namespace and router pod Deployment/Pod per design.md's foundation mechanism. Tag the namespace and router pod with the `osac.openshift.io/tenant` label per existing tenant-isolation convention.

**Testing Approach:** Integration test: create a Secondary VirtualNetwork, verify namespace + router pod exist with only the cluster-net interface; delete it, verify cleanup. Unit tests for the reconciler's branch logic.

**Dependencies:** Story 1.01

---

### Story 1.03: [DEV] Secondary CUDN per subnet with shared-label namespace targeting

**As a** tenant,
**I want to** have each subnet's Secondary UDN reach both the router pod's namespace and any namespace where a dual-attached VM lives,
**So that** VMs needing both this Secondary VN and another network can still use my subnet.

**Acceptance Criteria:**
- [ ] Creating a Subnet on a Secondary VirtualNetwork creates a `role: Secondary` CUDN targeting the router pod's namespace via a shared namespace label, not a fixed single namespace.
- [ ] The operator maintains that shared label on any additional namespace containing a VM that also attaches to this subnet but requires a Primary UDN elsewhere.
- [ ] The router pod gains an interface for the new subnet.

**Implementation Guidance:** Per design.md §Implementation Details ("Namespace-per-VN and shared-label targeting"), use a `namespaceSelector` matching a shared label rather than a fixed namespace reference on the Secondary CUDN. Address Open Question 7.2 (namespace-label scale) with a straightforward initial implementation (label applied/removed as dual-attached VMs are created/deleted); do not over-engineer for scale in this story — flag any scaling concern found during implementation back to the design.

**Testing Approach:** Integration test: create a VM requiring both a Primary UDN (in another namespace) and a Secondary VN subnet; verify it's placed in its own namespace and still reaches the subnet. Unit tests for label-selector construction and maintenance logic.

**Dependencies:** Story 1.02

---

### Story 1.04: [DEV] Router pod gateway, SNAT egress, persistent addressing, and cloud-init route/DNS injection

**As a** tenant,
**I want to** have VMs in different subnets reach each other and the internet through the router pod, with working DNS resolution,
**So that** my VirtualNetwork behaves like a real network without per-subnet infrastructure.

**Acceptance Criteria:**
- [ ] The router pod acts as the `.1` default gateway on each subnet's interface and forwards traffic between subnets.
- [ ] The router pod SNATs *all* traffic leaving via its cluster-network interface to its own pod IP — uniformly, regardless of destination — before forwarding it; this is not conditioned on whether the destination is cluster-internal or external.
- [ ] Each subnet's `.1` gateway IP is reserved via IPAMClaim with `ipam.lifecycle: Persistent`, so the router pod recovers the same address after a restart.
- [ ] A route to `.1` is injected via cloud-init on every VM attached to this Secondary VN, scoped based on dual-attachment: a full default route for a VM with only Secondary VN subnets, or a route scoped to just this VN's own CIDR for a VM that also has a Primary UDN attachment elsewhere (documented as a hard requirement — VMs without cloud-init have no route to `.1` at all).
- [ ] For a dual-attached VM, the VN-CIDR-scoped route reaches every other subnet in the VN via `.1`, and the VM's Primary attachment's own default route is left unmodified.
- [ ] For a VM with only Secondary VN subnets (no Primary UDN elsewhere), cloud-init also injects DNS resolver configuration identical to the router pod's own `/etc/resolv.conf`.
- [ ] For a VM with a Primary UDN attachment elsewhere, no DNS configuration is injected via the Secondary VN subnet — its Primary attachment's own mechanism is authoritative.
- [ ] A DNS query from a Secondary-VN-only VM to the injected resolver (the router pod's own DNS server) resolves cluster-internal service names correctly, forwarded through the router pod the same way any other cluster-internal-destined traffic is.
- [ ] OVN port security on the router pod's ports is patched to allow forwarding packets with source/destination IPs outside the router's own address.

**Implementation Guidance:** Enable `ip_forward` and standard L3 forwarding in the router pod's network namespace; configure a single, unconditional SNAT rule (rewrite source to the router pod's own pod IP) for all traffic leaving via the cluster-network interface — do not attempt to distinguish cluster-internal from external destinations in this rule; that distinction is already handled correctly downstream by OVN's own Service load-balancing and the node's pre-existing external-masquerade rule, with no additional logic needed here (see design.md's "Cluster-network egress SNAT is unconditional" note). Reserve gateway IPs via IPAMClaim per design.md's persistence pattern. Patch OVN port security via the same NB-database mechanism already used for equivalent purposes elsewhere in OSAC's networking controllers. For the route injected via cloud-init: determine scope from whether the VM also has a Primary UDN attachment (per Story 1.03) — inject a full default route to `.1` if not, or a route for just this VN's own CIDR via `.1` if so, never both/neither. For DNS: read the router pod's own `/etc/resolv.conf` at VM-creation time and pass its contents through unchanged into the VM's cloud-init network-config, alongside the route injection, only for the no-Primary-UDN-elsewhere case — do not select or construct a different resolver.

**Testing Approach:** Integration tests: two VMs in different subnets can ping/reach each other; a VM reaches an external address; a Secondary-VN-only VM resolves a cluster-internal service name via DNS (confirms design.md Open Question 9 — verify this explicitly, don't assume it); a dual-attached VM's `/etc/resolv.conf` is unaffected by this mechanism; a dual-attached VM's injected route is scoped to the VN's CIDR (not a default route) and its Primary attachment's default route is unchanged, while it still reaches another subnet in the VN; router pod restart preserves the same gateway IPs (verify via IPAMClaim status). Unit tests for IPAMClaim request construction, the unconditional-SNAT rule generation, route-scope selection logic, and cloud-init resolv.conf payload construction.

**Dependencies:** Story 1.02, Story 1.03

---

### Story 1.05: [DEV] Recreation-based subnet add/remove and fabric-manager-absence check

**As a** provider,
**I want to** have subnet changes reflected in the router pod, and bare-metal machinery skipped entirely when no fabric manager exists,
**So that** the VirtualNetwork works correctly end-to-end even before live-attach optimization (Epic 2) or fabric integration (Epic 4/5) exist.

**Acceptance Criteria:**
- [ ] Adding a subnet to a Secondary VirtualNetwork results in the router pod being recreated with the new Secondary UDN attachment, recovering existing subnets' addresses via persistent IPAMClaim.
- [ ] Removing a subnet results in the router pod being recreated without the removed attachment, with remaining subnets unaffected beyond the recreation itself.
- [ ] At VirtualNetwork creation, the operator resolves the target NetworkClass and records whether it has a fabric manager configured; when it does not, no transit interface or fabric-side machinery (Epics 4/5) is ever provisioned for this VirtualNetwork.

**Implementation Guidance:** This story implements the "always recreate" baseline described in design.md's Alternatives section, which Epic 2 later replaces with a live-attach-first, recreate-as-fallback approach — write the recreation logic so Epic 2 can slot the live-attach path in front of it without rework (e.g., as a function/method the fallback path can call directly). For the NetworkClass check, see design.md's Open Question 7 (Phase 2) — implement using whatever NetworkClass field/representation is confirmed to indicate "no fabric manager"; if not yet resolved, flag this explicitly rather than guessing at the API shape.

**Testing Approach:** Integration tests: add a subnet, verify router pod recreation and address persistence; remove a subnet, verify the same; create a Secondary VN on a fabric-manager-less NetworkClass, verify no transit interface appears. Unit tests for the recreation reconciliation logic.

**Dependencies:** Story 1.02, Story 1.04

---

### Story 1.06: [QE] E2E validation of Secondary VirtualNetwork core connectivity and lifecycle

**As a** QE engineer,
**I want to** validate Secondary VirtualNetwork behavior end-to-end,
**So that** regressions in core connectivity are caught before release.

**Acceptance Criteria:**
- [ ] E2E scenario: create a Secondary VirtualNetwork with two subnets and VMs in each; verify inter-subnet connectivity and internet egress.
- [ ] E2E scenario: add a third subnet, verify connectivity for all three; remove one, verify the remaining two are unaffected.
- [ ] E2E scenario: a VM requiring a Primary UDN elsewhere plus a Secondary VN subnet is reachable on both.
- [ ] Negative scenario: a VM without cloud-init has no default route and cannot reach other subnets or the internet.

**Testing Approach:** Full e2e test suite exercising the workflows above against a real (or kind-based) cluster, following osac-test-infra's existing pytest/gRPC/K8s client patterns.

**Dependencies:** Story 1.01, Story 1.02, Story 1.03, Story 1.04, Story 1.05

---

### Story 1.07: [DOCS] Document the Secondary VirtualNetwork type

**As a** tenant or provider,
**I want to** understand what the Secondary VirtualNetwork type is and when to use it,
**So that** I can decide whether it fits my workload.

**Documentation Scope:** Readers need to understand: what a Secondary VirtualNetwork is (single router pod, VN-scoped gateway) and how it differs from the Primary model; how to opt in (the type field); the cloud-init requirement for VM default routes; and that bare-metal connectivity (Phase 2) is a separate, additive capability not required for basic use.

**Documentation Inputs:**
- **Story 1.01:** the networking-type field — name, allowed values, default, immutability.
- **Story 1.04:** the cloud-init requirement for VM default routes, and the consequence of omitting it.
- **Story 1.05:** subnet add/remove behavior (a brief interruption is possible in this phase, refined further once Epic 2 lands).

**Dependencies:** Story 1.01, Story 1.02, Story 1.03, Story 1.04, Story 1.05

## Test Case References

Verified by: TC-FR1-01, TC-FR2-01, TC-FR3-01, TC-FR3-02, TC-FR4-01, TC-FR5-01, TC-FR6-01, TC-FR14-01, TC-FR15-01, TC-FR21-01, TC-FR21-02, TC-FR22-01, TC-FR22-02
