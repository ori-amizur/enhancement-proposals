# Epic 5: Bidirectional Bare-Metal Route Dispatch

- **T-Shirt Size:** L
- **Phase:** 2
- **PRD Requirements:** P-1 prerequisite, FR-17, FR-18, FR-19, FR-20
- **Design Reference:** design.md §Proposal (Phase 2, item 8), §Workflow Description ("Bare-metal connectivity setup"), §Implementation Details/Notes/Constraints ("Backend-neutral transit contract", "Static route dispatch"), Risk 6.4/6.5
- **Dependencies:** Epic 4 and Story 5.00 (Secondary Subnet backend ownership)

## Summary

Delivers end-to-end VM ↔ bare-metal connectivity by dispatching static routes in both directions: from Kubernetes-owned VM subnets to the selected fabric manager, and from fabric-owned bare-metal subnets into the router pod's own ConfigMap. Every Secondary Subnet has exactly one immutable owner. A Kubernetes-owned Subnet therefore receives an explicit fabric-side route, while a fabric-owned Subnet is represented natively by the fabric and contributes a route toward the fabric from the router pod; the same Subnet is never provisioned by both backends. This is the epic that makes Epic 4's backend-specific transit interface actually useful. The current execution target is the Netris/EVPN transit contract; the route model is intentionally independent of whether a future backend uses EVPN or a VLAN/LocalNet path.

## Acceptance Criteria

- [ ] Adding a Kubernetes-owned VM Subnet to a supported fabric-manager-backed Secondary VirtualNetwork results in the operator pushing a static route for that subnet's CIDR into the fabric manager, with the router pod's transit address as next-hop; removing it withdraws the route. Fabric-owned Subnets are not sent through this explicit fabric-route path because the fabric already owns their representation.
- [ ] When a fabric-owned bare-metal Subnet becomes relevant to the VirtualNetwork, the operator writes the corresponding route into the router pod's ConfigMap, applied live by the agent.
- [ ] A VM on any subnet can reach a bare-metal server on the fabric, and vice versa, once both routes are in place.
- [ ] Deploying this epic does not change behavior for any Phase-1-only (fabric-manager-less) Secondary VirtualNetwork.

## Stories

### Story 5.00: [DEV] Secondary Subnet backend ownership and dispatch selection

**As a** provider,
**I want to** assign each Secondary Subnet exactly one immutable backend owner,
**So that** later fabric route dispatch is unambiguous and the same Subnet is
never provisioned by both the fabric and Kubernetes backends.

**Acceptance Criteria:**
- [ ] `Subnet.spec.implementationStrategy` is optional and immutable, and is
  accepted only when the parent VirtualNetwork uses `Secondary` networking.
- [ ] An explicit value resolves to exactly one registered backend manager. An
  omitted value uses the deployment default: the current default selects the
  fabric target when available and otherwise the Kubernetes target.
- [ ] A Secondary Subnet is provisioned and deprovisioned through one target
  only, even when the parent NetworkClass resolves both fabric and Kubernetes
  managers.
- [ ] The selected owner role is persisted so deletion remains deterministic
  if the parent VirtualNetwork is already gone.
- [ ] Primary Subnets retain their existing dispatcher behavior, and an
  implementation strategy is rejected for them.

**Implementation Scope:** Extend the public/private Subnet API and operator
CRD, fulfillment-service validation/mapping, and CLI creation flow; select one
dispatcher target in the operator; persist the selected role for deletion; and
expose the field in the normal Subnet views.

**Testing Approach:** API scope and immutability tests; CLI request-building
tests; operator tests proving explicit/default selection, one-target
provision/deprovision, and Primary-network rejection.

**Dependencies:** Epic 1 dispatcher and NetworkClass foundations. This is a
cross-cutting prerequisite for Stories 5.01–5.03, not a transit-interface
implementation and not an additional Epic 4 story.

---

### Story 5.01: [DEV] Static route dispatch to the fabric manager

**As a** provider,
**I want to** have VM subnet CIDRs published to the fabric automatically,
**So that** bare-metal servers can route back to VMs on Kubernetes-owned Subnets.

**Acceptance Criteria:**
- [ ] Adding a Kubernetes-owned VM Subnet to a supported fabric-manager-backed Secondary VirtualNetwork pushes a static route into the selected fabric manager's existing route-management API: the subnet's CIDR via the router pod's transit-interface address. The current supported backend is Netris/EVPN.
- [ ] Removing the subnet withdraws that route.
- [ ] Adding or removing a fabric-owned Secondary Subnet does not create or withdraw this explicit fabric-side route; its native fabric representation remains managed by the selected fabric backend.
- [ ] This uses the same dispatcher mechanism already used to manage the fabric manager's VPCs, routes, and NAT rules elsewhere in OSAC — not a new integration surface.

**Implementation Guidance:** Extend the Subnet reconciliation lifecycle with an independent route-only fabric target. The route intent must remain separate from Subnet ownership: the Kubernetes-owned Subnet is provisioned by Kubernetes, while its CIDR is additionally published through the selected fabric manager. Persist route job history and the selected manager so deletion remains deterministic (see Story 5.03 for cross-direction consistency).

**Testing Approach:** Integration test: add a subnet, verify the route appears in the fabric manager; remove it, verify withdrawal. Unit tests for route construction and the push/withdraw API calls.

**Dependencies:** Story 5.00, Story 4.03 (from Epic 4)

---

### Story 5.02: [DEV] Bare-metal-subnet route delivery to the router pod

**As a** provider,
**I want to** have fabric-owned bare-metal subnet routes delivered into the router pod automatically,
**So that** VMs can reach bare-metal servers without manual configuration.

**Acceptance Criteria:**
- [ ] When a fabric-owned bare-metal Subnet becomes relevant to the VirtualNetwork, the operator writes a route into the router pod's ConfigMap: the bare-metal subnet's CIDR via the authoritative fabric peer transit address(es), over the transit interface.
- [ ] The agent applies this route live, without a router pod restart.
- [ ] The operator determines the destination CIDR and ownership from authoritative Subnet CR state, and obtains only the backend-specific transit peer data from the persisted transit contract; it never discovers bare-metal subnets from the fabric.

**Implementation Guidance:** Reuse the existing `RouteSpec`/`routes` ConfigMap contract and live route reconciler. Add a backend-neutral aggregate route projection that is invoked by both Kubernetes and fabric Subnet lifecycles, protected by the per-VirtualNetwork mutation Lease. The current Netris implementation publishes the peer transit address(es) in the VN-scoped transit contract; a future backend supplies the same route intent through its own contract without changing the agent.

**Testing Approach:** Integration test: configure a bare-metal subnet route, verify it's applied to the router pod's routing table live. Unit tests for ConfigMap schema extension and route application.

**Dependencies:** Story 5.00, Story 2.01 (from Epic 2), Story 4.03 (from Epic 4)

---

### Story 5.03: [DEV] Reconciliation consistency between fabric-side and router-pod-side routes

**As a** provider,
**I want to** have the fabric-side route push and the router pod's ConfigMap update stay consistent with each other,
**So that** a partial failure doesn't leave one side stale.

**Acceptance Criteria:**
- [ ] The fabric-side route push/withdraw (Story 5.01) and the router pod ConfigMap update (Story 5.02) are treated as part of the same reconciliation loop, with retry/requeue on failure.
- [ ] A transient failure on either side does not silently leave the two out of sync — the reconciler retries until both are consistent, or surfaces a clear error/status if it cannot.
- [ ] `Subnet.status.conditions[type=RoutesReady]` reports whether the router-pod route projection and, when applicable, the explicit fabric-side route have converged.

**Implementation Guidance:** Addresses Risk 6.4 (design.md) directly. Reuse the existing owner and `FabricRouteJobs` AAP job histories, the per-VirtualNetwork Lease, and the provisioning lifecycle's retry/backoff/requeue behavior rather than inventing a second job or locking mechanism. The route condition must distinguish pending, failed, and succeeded states; Phase-1-only or unsupported backends do not receive this condition. This is convergent reconciliation, not a distributed transaction: an intermediate partial state is acceptable only while it is visible and being retried.

**Testing Approach:** Integration tests simulating a failure on one side (e.g., fabric manager API returns an error) and verifying the reconciler retries and eventually converges, or surfaces status clearly if it cannot.

**Dependencies:** Story 5.01, Story 5.02

---

### Story 5.04: [QE] E2E validation of bidirectional bare-metal connectivity

**As a** QE engineer,
**I want to** validate VM-to-bare-metal connectivity end-to-end in both directions,
**So that** regressions are caught before release, without affecting Phase 1 behavior.

**Acceptance Criteria:**
- [ ] E2E scenario provisions the complete path: a Netris-backed Secondary VirtualNetwork, a Kubernetes-owned VM Subnet, a VM, a fabric-owned bare-metal Subnet, and a BareMetalInstance.
- [ ] E2E scenario: the VM reaches the BareMetalInstance on the fabric, and the BareMetalInstance reaches the VM, once both routes are configured.
- [ ] The VM's `ip route get <bare-metal-IP>` selects the secondary interface and secondary source IP; the test fails if the primary interface is selected.
- [ ] The BareMetalInstance's route to the VM's secondary IP uses the fabric route toward the router transit path; the test must not pass through the primary network.
- [ ] Deleting the BareMetalInstance and Subnet withdraws the router-pod and fabric-side routes before the VirtualNetwork is removed.
- [ ] E2E scenario: on a VirtualNetwork with Phase 2 configured, verify Phase 1 behavior (inter-subnet routing, egress, NAT Gateway if configured) is unaffected.
- [ ] E2E scenario: deploying Phase 2 (this epic) into an environment with existing Phase-1-only Secondary VirtualNetworks does not change their behavior.

**Testing Approach:** Full e2e suite against a real fabric-manager-backed test environment. Create the VM and BareMetalInstance through their normal OSAC APIs, wait for both Subnets to report `RoutesReady=True`, verify the router-pod route and fabric-side route independently, run bidirectional connectivity checks, and assert route selection with `ip route get` on both endpoints. Use the BMI's fabric-subnet IP as the destination so a primary-network path cannot produce a false positive.

**Dependencies:** Story 5.01, Story 5.02, Story 5.03

---

### Story 5.05: [DOCS] Document bare-metal connectivity setup and troubleshooting

**As an** SRE or provider,
**I want to** understand how bare-metal connectivity is configured and how to troubleshoot it,
**So that** I can operate and support this capability.

**Documentation Scope:** Readers need to understand: the two-route-direction model (VN→fabric and fabric→router-pod); how to verify both routes exist when troubleshooting unreachable bare-metal servers; the current Netris EVPN Type-2 MAC re-advertisement consideration after a router pod restart (Open Question 3); that a future VLAN/LocalNet backend may have different L2 reconvergence behavior; and the deletion ordering expected for a VirtualNetwork with bare-metal connectivity configured (routes withdrawn before pod/namespace teardown).

**Documentation Inputs:**
- **Story 5.01:** the fabric-side static route and its content.
- **Story 5.02:** the router-pod-side route and its content.
- **Story 5.03:** what a consistency failure looks like and how it's surfaced.

**Dependencies:** Story 5.01, Story 5.02, Story 5.03

## Test Case References

Verified by: TC-FR17-01, TC-FR17-02, TC-FR18-01, TC-FR18-02, TC-FR19-01, TC-FR20-01
