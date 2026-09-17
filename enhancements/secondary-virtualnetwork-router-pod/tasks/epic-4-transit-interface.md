# Epic 4: Fabric Manager Transit Contract & Netris EVPN Interface

- **T-Shirt Size:** M
- **Phase:** 2
- **PRD Requirements:** FR-16
- **Design Reference:** design.md §Proposal (Phase 2, items 7–8), §Workflow Description ("VirtualNetwork creation," step 4), §Implementation Details/Notes/Constraints ("Fabric-manager-conditional provisioning," "Backend-neutral transit contract," "Transit address persistence," "Bare-metal transit interface")
- **Dependencies:** Epic 1

## Summary

Adds the fabric-manager and transit-capability detection logic and the Netris EVPN transit interface that Epic 5's route dispatch builds on. Delivered as its own epic because it establishes a real, independently verifiable capability (the router pod gains genuine Netris EVPN MAC-VRF membership in the fabric) before route dispatch depends on it.

The design uses a backend-specific transit contract rather than assuming that
every fabric manager means EVPN. The current epic implements only the Netris
contract. A future backend such as `agentless_net` may publish a VN-specific
tagged VLAN/LocalNet attachment connected to the VN's routing namespace on a
net-node, but that implementation is explicitly outside this epic.

## Acceptance Criteria

- [ ] At VirtualNetwork creation, the operator resolves the target NetworkClass and identifies the selected fabric manager and its supported transit capability; fabric-manager presence alone does not imply EVPN.
- [ ] When the selected backend is the supported Netris/EVPN backend, the router pod is provisioned with an additional transit interface — a VirtualNetwork-specific Primary CUDN with EVPN transport — reaching that VirtualNetwork's dedicated transit MAC-VRF.
- [ ] The transit interface's address is reserved via IPAMClaim with `ipam.lifecycle: Persistent`.
- [ ] When no fabric manager is configured, or when the selected backend does not provide a supported transit contract, none of this is provisioned (verified as a negative case here, building on Epic 1 Story 1.05's initial check).
- [ ] OSAC provisions and reconciles one isolated, backend-appropriate transit network per fabric-backed VirtualNetwork, including its fabric-side resources, through the selected fabric-manager API; creating a tenant VirtualNetwork does not require a provider or tenant to create a subnet, VNI, VLAN, or port assignment manually in the fabric UI. For the current Netris implementation, a configured transit prefix may be reused across isolated Netris VPC/VRF domains, while address allocation within each prefix uses disjoint Netris and OSAC/CUDN pools.

## Stories

### Story 4.01: [DEV] NetworkClass fabric-manager and transit-capability detection

**As a** provider,
**I want to** have the operator reliably determine the selected fabric manager and whether it provides a supported transit contract,
**So that** bare-metal machinery is provisioned only where the selected backend can actually support it.

**Acceptance Criteria:**
- [ ] The operator resolves a VirtualNetwork's target NetworkClass and determines the selected fabric-manager name and transit capability unambiguously.
- [ ] For the current epic, the operator recognizes Netris/EVPN as supported and does not infer EVPN merely from fabric-manager presence.
- [ ] A configured but unsupported fabric manager produces an explicit unsupported-transit result and does not provision an EVPN transit interface.
- [ ] The determination is made once, at VirtualNetwork creation, and does not change over the VirtualNetwork's lifetime.

**Implementation Guidance:** This story resolves Open Question 7 (design.md) — confirm against the current NetworkClass API how "no fabric manager" is represented and how manager capabilities are exposed. Do not guess at the API shape — verify against the actual NetworkClass CRD/controller before implementing. Reuse or replace the initial check from Epic 1 Story 1.05 with an immutable creation-time decision that records both the selected backend and the supported transit mode. The current implementation only accepts Netris/EVPN; the contract must leave room for a future `agentless_net` VLAN/LocalNet mode without treating it as EVPN.

**Testing Approach:** Unit tests for the detection logic against fabric-manager-less, Netris/EVPN, and configured-but-unsupported backend NetworkClass fixtures. No new integration test needed beyond what Story 1.05 and Stories 4.02–4.03 already cover.

**Dependencies:** Story 1.05 (from Epic 1)

---

### Story 4.02: [DEV] OSAC-managed VirtualNetwork-scoped Netris EVPN transit lifecycle

**As a** provider,
**I want to** have OSAC allocate and manage an isolated Netris EVPN transit network for each Netris-backed Secondary VirtualNetwork,
**So that** a tenant can create a VirtualNetwork without performing manual fabric-manager configuration first.

This story closes the provider-allocation gap in the original Epic 4 design. OSAC operates Netris on the tenant's behalf, but the transit network is VirtualNetwork-scoped: each eligible VirtualNetwork receives an independent MAC-VRF/VNI, provider-selected transit prefix, fabric IPAM allocation, and EVPN CUDN. No transit resource is shared between VirtualNetworks. The output is a VN-scoped transit contract consumed by the Kubernetes-side manager; a future fabric backend may publish a different contract and is not implemented here.

**Acceptance Criteria:**
- [ ] Creating a Netris/EVPN-backed Secondary VirtualNetwork causes OSAC to allocate or reconcile exactly one transit network and matching fabric-side IPAM subnet for that VirtualNetwork through the Netris API.
- [ ] The transit prefix comes from provider configuration, not from the tenant. The preferred value is a statically allocated IPv4 link-local `/24` (for example, `169.254.240.0/24`) with explicit exclusions; if compatibility validation rejects link-local addressing, OSAC uses a provider-configured private prefix. In either mode, the default allocation policy assigns a Netris `/26`, an OSAC/CUDN `/26`, and a reserved `/25`; Netris is restricted to its pool and CUDN is restricted to the OSAC pool. OSAC performs overlap and containment checks within the applicable Netris VPC/VRF, against the VirtualNetwork CIDR when they share that VPC/VRF, against platform-reserved link-local addresses, and against fabric-reserved ranges.
- [ ] OSAC obtains or reconciles the Netris resources required for that VirtualNetwork's transit network, including an isolated Netris VPC/VRF, a unique MAC-VRF/VNI, and the required site, gateway, node, or fabric-port associations, without requiring manual fabric UI operations.
- [ ] OSAC publishes a VN-scoped transit contract identifying `evpn` as the transport and providing the values required by the Kubernetes-side manager, including the transit CIDR, disjoint pools, MAC-VRF VNI, VTEP, CUDN name, and persistent IPAMClaim name.
- [ ] OSAC waits for the VirtualNetwork-specific transit network and IPAM allocation to become usable before publishing the selected transit prefix, VNI, or endpoint information to that VirtualNetwork's dependent CUDN/IPAM resources.
- [ ] Before selecting link-local mode, OSAC verifies CUDN admission/readiness, IPAMClaim allocation, router-pod interface configuration, ARP with the fabric peer, and static routing using link-local next hops over the EVPN L2 segment. If any compatibility check fails, reconciliation selects the configured private-prefix fallback instead of leaving the VN partially provisioned.
- [ ] Two eligible VirtualNetworks never share a transit CUDN, Netris VPC/VRF, MAC-VRF, VNI, or fabric IPAM subnet. They may use the same selected transit prefix only when their VPC/VRF and CUDN isolation makes that reuse unambiguous and safe.
- [ ] Reconciliation is idempotent and safe to retry for the same VirtualNetwork: repeated sequential reconciliation does not allocate a second transit network, CIDR, or VNI.
- [ ] Reconciliation is safe under genuine concurrency: two overlapping reconcile/job invocations for the same VirtualNetwork, running at the same time (not one after the other), still result in exactly one transit network, CIDR, and VNI, with the losing invocation observing real contention rather than proceeding unserialized. This is a distinct requirement from retry-safety above and must be demonstrated with an actual concurrent-execution test, not inferred from the presence of a lock — a structurally-present lock is not evidence it serializes anything (see Implementation Guidance and design.md Risks).
- [ ] Deleting a VirtualNetwork withdraws its fabric routes and removes its transit CUDN/IPAM resources and OSAC-owned fabric IPAM subnet/transit network in dependency order, without affecting any other VirtualNetwork.
- [ ] Fabric API failures, allocation conflicts, and incomplete provisioning are retried through reconciliation and exposed in status/events without leaking orphaned resources.
- [ ] The router's transit IPAMClaim can never allocate an address from the Netris-owned or otherwise reserved ranges; a current-host snapshot is not accepted as the sole protection against later Netris allocations.
- [ ] A transit resource deletion that the fabric API repeatedly rejects or fails (not a transient error) surfaces a distinct, dedicated status condition for stuck teardown — separate from the general reconcile-error/event stream — so a stalled, cost-bearing fabric resource cannot be mistaken for ordinary retry transience or missed entirely (design.md, "Stuck transit teardown").

**Implementation Guidance:** Use the existing VirtualNetwork provisioning lifecycle. Prefer deriving deterministic fabric resource names from the VirtualNetwork identity (UUID) over persisting a separate identifier mapping, if Netris's naming constraints (length, character set, uniqueness scope) allow it (design.md, Open Question 11); if they don't, persist the identifiers explicitly and document where (VirtualNetwork status, an annotation, or a separate resource — not left implicit). Do not treat `NETRIS_EVPN_TRANSIT_VNET_NAME` or similar environment variables as a shared source of truth; replace them with provider configuration describing the allocation source. The provider configuration must define disjoint Netris and OSAC/CUDN pools within the selected `/24`, defaulting to `/26` each with the remaining `/25` reserved. Netris must enforce its pool; CUDN must reserve the Netris and unused ranges so its IPAMClaim can allocate only from the OSAC pool. The implementation must support adoption of an already-existing matching VN-specific transit network during upgrade, while refusing to adopt an unrelated shared resource and avoiding deletion of resources that OSAC does not own. A provider/fabric-scoped lock may serialize fabric API operations if required by the backend, but it must not turn the transit resource into a shared network — and whatever locking mechanism is used must be validated under real concurrent execution as part of this story's own test suite (see the concurrency acceptance criterion above), not assumed correct because the code reviews as a mutex. This codebase has a documented precedent for exactly this failure mode: a per-VirtualNetwork Lease-based lock elsewhere in `osac-aap` (`osac.service.vnet_lock`) was a complete no-op for multiple prior stories — its `include_role` calls were missing `tasks_from`, so the lock's own tasks never ran at all — and this went undetected until a genuine concurrent test was run, because nothing before that had ever exercised real concurrency against it.

**Testing Approach:** Integration tests against a fabric-manager test double or environment that exercises the fabric API: link-local compatibility validation with a provider-configured `/24`, private-prefix fallback when either backend rejects link-local addressing, enforcement of disjoint default `/26` Netris and OSAC/CUDN pools with the remaining `/25` reserved, allocation for two VirtualNetworks, verification of distinct VPC/VRF scopes, VNets, VNIs, MAC-VRFs, IPAM subnets, and CUDNs, including a case where both isolated VPC/VRF scopes use the same selected transit prefix; sequential-retry idempotency for repeated reconciliation; a separate, genuinely concurrent reconciliation test (two overlapping invocations, not two sequential ones) asserting single allocation and observable contention on the losing side; adoption of an existing matching VN-specific resource; deletion of one VirtualNetwork without affecting the other; injected failures during allocation/readiness/deletion; and an injected persistent (non-transient) deletion failure verifying the dedicated stuck-teardown status condition appears and later clears. The end-to-end test must succeed without a manual fabric UI setup step.

**Dependencies:** Story 4.01

---

### Story 4.03: [DEV] Netris EVPN transit interface provisioning

**As a** provider,
**I want to** have the router pod join Netris's transit MAC-VRF automatically when the Netris/EVPN backend is selected,
**So that** bare-metal connectivity (Epic 5) has a working transport to build on.

**Acceptance Criteria:**
- [ ] When Story 4.01 determines that the Netris/EVPN backend is selected, the operator provisions a VirtualNetwork-specific Primary CUDN with EVPN transport on the router pod, reaching that VirtualNetwork's dedicated transit MAC-VRF.
- [ ] The transit interface's address is reserved via IPAMClaim with `ipam.lifecycle: Persistent`, recovering the same address across restarts/reschedules.
- [ ] The transit interface is created once, at VirtualNetwork creation, and is unaffected by individual subnet add/remove.

**Implementation Guidance:** Per design.md, the Netris transit CUDN uses EVPN transport with MAC-VRF only (no IP-VRF) — do not configure an `ipVRF` field. This requires Netris to support participating as a BGP EVPN peer for cluster-originated MAC-VRFs (an assumption stated in the PRD, not something this story needs to newly validate, but worth confirming is still true in the target environment before relying on it). Do not branch on arbitrary fabric-manager presence; the transport must come from the supported transit contract.

**Testing Approach:** Integration test: create a Secondary VirtualNetwork on a NetworkClass selecting Netris/EVPN, verify the transit interface exists with a persistent IPAMClaim-backed address; restart the router pod, verify the same address is recovered. Include a negative test for a configured-but-unsupported fabric backend.

**Dependencies:** Story 4.01, Story 4.02

---

### Story 4.04: [QE] E2E validation of conditional Netris EVPN transit provisioning

**As a** QE engineer,
**I want to** validate that the Netris EVPN transit interface is provisioned exactly when it should be,
**So that** fabric-manager-less and unsupported-backend deployments never accidentally gain EVPN machinery.

**Acceptance Criteria:**
- [ ] E2E scenario: create a Secondary VirtualNetwork on a NetworkClass selecting Netris/EVPN, verify the transit interface and its persistent address.
- [ ] E2E scenario: create a Secondary VirtualNetwork on a fabric-manager-less NetworkClass, verify no transit interface is ever created, including after subnet add/remove activity.
- [ ] E2E scenario: create a Secondary VirtualNetwork on a NetworkClass selecting a configured but unsupported fabric backend, verify no EVPN transit interface is created and the unsupported capability is visible in status/events.

**Testing Approach:** Full e2e suite covering Netris/EVPN, fabric-manager-less, and configured-but-unsupported backend NetworkClass configurations. A future `agentless_net` VLAN/LocalNet implementation requires additional test cases and is not part of this epic's execution scope.

**Dependencies:** Story 4.01, Story 4.02, Story 4.03

## Test Case References

Verified by: TC-FR16-01, TC-FR16-02
