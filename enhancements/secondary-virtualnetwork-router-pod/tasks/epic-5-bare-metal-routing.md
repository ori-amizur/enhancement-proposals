# Epic 5: Bidirectional Bare-Metal Route Dispatch

- **T-Shirt Size:** L
- **Phase:** 2
- **PRD Requirements:** FR-17, FR-18, FR-19, FR-20
- **Design Reference:** design.md §Proposal (Phase 2, item 8), §Workflow Description ("Bare-metal connectivity setup"), §Implementation Details/Notes/Constraints ("Static route dispatch"), Risk 6.4/6.5
- **Dependencies:** Epic 4

## Summary

Delivers end-to-end VM ↔ bare-metal connectivity by dispatching static routes in both directions: from the router pod's VN subnets to the fabric manager, and from bare-metal subnets into the router pod's own ConfigMap. This is the epic that makes Epic 4's transit interface actually useful.

## Acceptance Criteria

- [ ] Adding a VM subnet results in the operator pushing a static route for that subnet's CIDR into the fabric manager, with the router pod's transit address as next-hop; removing it withdraws the route.
- [ ] When a bare-metal subnet becomes relevant to the VirtualNetwork, the operator writes the corresponding route into the router pod's ConfigMap, applied live by the agent.
- [ ] A VM on any subnet can reach a bare-metal server on the fabric, and vice versa, once both routes are in place.
- [ ] Deploying this epic does not change behavior for any Phase-1-only (fabric-manager-less) Secondary VirtualNetwork.

## Stories

### Story 5.01: [DEV] Static route dispatch to the fabric manager

**As a** provider,
**I want to** have VM subnet CIDRs published to the fabric automatically,
**So that** bare-metal servers can route back to VMs.

**Acceptance Criteria:**
- [ ] Adding a VM subnet to a fabric-manager-backed Secondary VirtualNetwork pushes a static route into the fabric manager's existing route-management API: the subnet's CIDR via the router pod's transit-interface address.
- [ ] Removing the subnet withdraws that route.
- [ ] This uses the same dispatcher mechanism already used to manage the fabric manager's VPCs, routes, and NAT rules elsewhere in OSAC — not a new integration surface.

**Implementation Guidance:** Extend the same reconciliation loop that already updates the router pod's ConfigMap (Epic 1/2) to also push/withdraw this route as part of the same subnet add/remove flow, so the two stay consistent (see Story 5.03).

**Testing Approach:** Integration test: add a subnet, verify the route appears in the fabric manager; remove it, verify withdrawal. Unit tests for route construction and the push/withdraw API calls.

**Dependencies:** Story 4.02 (from Epic 4)

---

### Story 5.02: [DEV] Bare-metal-subnet route delivery to the router pod

**As a** provider,
**I want to** have bare-metal subnet routes delivered into the router pod automatically,
**So that** VMs can reach bare-metal servers without manual configuration.

**Acceptance Criteria:**
- [ ] When a bare-metal subnet becomes relevant to the VirtualNetwork, the operator writes a route into the router pod's ConfigMap: the bare-metal subnet's CIDR via the fabric peer's transit address, over the transit interface.
- [ ] The agent applies this route live, without a router pod restart.
- [ ] The operator determines this route's content from its own authoritative Subnet CR state, never from a discovery mechanism.

**Implementation Guidance:** Extend the agent's ConfigMap schema (Epic 2, Story 2.01) to include fabric-reachable routes alongside the existing per-subnet routes — this is an additive change to an existing reconciliation path, not a new agent capability.

**Testing Approach:** Integration test: configure a bare-metal subnet route, verify it's applied to the router pod's routing table live. Unit tests for ConfigMap schema extension and route application.

**Dependencies:** Story 2.01 (from Epic 2), Story 4.02 (from Epic 4)

---

### Story 5.03: [DEV] Reconciliation consistency between fabric-side and router-pod-side routes

**As a** provider,
**I want to** have the fabric-side route push and the router pod's ConfigMap update stay consistent with each other,
**So that** a partial failure doesn't leave one side stale.

**Acceptance Criteria:**
- [ ] The fabric-side route push/withdraw (Story 5.01) and the router pod ConfigMap update (Story 5.02) are treated as part of the same reconciliation loop, with retry/requeue on failure.
- [ ] A transient failure on either side does not silently leave the two out of sync — the reconciler retries until both are consistent, or surfaces a clear error/status if it cannot.

**Implementation Guidance:** Addresses Risk 6.4 (design.md) directly. Follow the existing requeue pattern already used elsewhere in osac-operator's networking controllers (e.g., the `VirtualMachineReference` precondition check pattern referenced in the Unified Networking design) rather than inventing a new consistency mechanism.

**Testing Approach:** Integration tests simulating a failure on one side (e.g., fabric manager API returns an error) and verifying the reconciler retries and eventually converges, or surfaces status clearly if it cannot.

**Dependencies:** Story 5.01, Story 5.02

---

### Story 5.04: [QE] E2E validation of bidirectional bare-metal connectivity

**As a** QE engineer,
**I want to** validate VM-to-bare-metal connectivity end-to-end in both directions,
**So that** regressions are caught before release, without affecting Phase 1 behavior.

**Acceptance Criteria:**
- [ ] E2E scenario: a VM reaches a bare-metal server on the fabric, and the bare-metal server reaches the VM, once both routes are configured.
- [ ] E2E scenario: on a VirtualNetwork with Phase 2 configured, verify Phase 1 behavior (inter-subnet routing, egress, NAT Gateway if configured) is unaffected.
- [ ] E2E scenario: deploying Phase 2 (this epic) into an environment with existing Phase-1-only Secondary VirtualNetworks does not change their behavior.

**Testing Approach:** Full e2e suite covering the scenarios above against a real fabric-manager-backed test environment.

**Dependencies:** Story 5.01, Story 5.02, Story 5.03

---

### Story 5.05: [DOCS] Document bare-metal connectivity setup and troubleshooting

**As an** SRE or provider,
**I want to** understand how bare-metal connectivity is configured and how to troubleshoot it,
**So that** I can operate and support this capability.

**Documentation Scope:** Readers need to understand: the two-route-direction model (VN→fabric and fabric→router-pod); how to verify both routes exist when troubleshooting unreachable bare-metal servers; the EVPN Type-2 MAC re-advertisement consideration after a router pod restart (Open Question 3); the deletion ordering expected for a VirtualNetwork with bare-metal connectivity configured (routes withdrawn before pod/namespace teardown).

**Documentation Inputs:**
- **Story 5.01:** the fabric-side static route and its content.
- **Story 5.02:** the router-pod-side route and its content.
- **Story 5.03:** what a consistency failure looks like and how it's surfaced.

**Dependencies:** Story 5.01, Story 5.02, Story 5.03

## Test Case References

Verified by: TC-FR17-01, TC-FR18-01, TC-FR19-01, TC-FR20-01
