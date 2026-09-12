# Epic 4: Fabric Manager Detection & EVPN Transit Interface

- **T-Shirt Size:** M
- **Phase:** 2
- **PRD Requirements:** FR-16
- **Design Reference:** design.md §Proposal (Phase 2, item 7), §Workflow Description ("VirtualNetwork creation," step 4), §Implementation Details/Notes/Constraints ("Fabric-manager-conditional provisioning," "Transit address persistence," "Bare-metal transit interface")
- **Dependencies:** Epic 1

## Summary

Adds the fabric-manager-detection logic and EVPN transit interface that Epic 5's route dispatch builds on. Delivered as its own epic because it establishes a real, independently verifiable capability (the router pod gains genuine EVPN MAC-VRF membership in the fabric) before route dispatch depends on it.

## Acceptance Criteria

- [ ] At VirtualNetwork creation, the operator resolves the target NetworkClass and determines whether it has a fabric manager configured.
- [ ] When one is configured, the router pod is provisioned with an additional transit interface — a VirtualNetwork-specific Primary CUDN with EVPN transport — reaching that VirtualNetwork's dedicated transit MAC-VRF.
- [ ] The transit interface's address is reserved via IPAMClaim with `ipam.lifecycle: Persistent`.
- [ ] When no fabric manager is configured, none of this is provisioned (verified as a negative case here, building on Epic 1 Story 1.05's initial check).
- [ ] OSAC provisions and reconciles one isolated transit network per fabric-backed VirtualNetwork, including its fabric-side IPAM resources, through the fabric-manager API; creating a tenant VirtualNetwork does not require a provider or tenant to create a subnet, VNI, or port assignment manually in the fabric UI. A configured transit prefix may be reused across isolated Netris VPC/VRF domains.

## Stories

### Story 4.01: [DEV] NetworkClass fabric-manager detection

**As a** provider,
**I want to** have the operator reliably determine whether a NetworkClass has a fabric manager configured,
**So that** bare-metal machinery is provisioned only where it can actually work.

**Acceptance Criteria:**
- [ ] The operator resolves a VirtualNetwork's target NetworkClass and determines fabric-manager presence/absence unambiguously.
- [ ] The determination is made once, at VirtualNetwork creation, and does not change over the VirtualNetwork's lifetime.

**Implementation Guidance:** This story resolves Open Question 7 (design.md) — confirm against the current NetworkClass API how "no fabric manager" is actually represented (OSAC's Unified Networking model generally treats `fabricManager` as required, so this may correspond to a distinct NetworkClass configuration, e.g. a `udn-net`-style class, rather than an empty/optional value on the same field). Do not guess at the API shape — verify against the actual NetworkClass CRD/controller before implementing. This determination should reuse or replace the initial check from Epic 1 Story 1.05 (which only needed to skip Phase 2 provisioning) with the confirmed, authoritative mechanism.

**Testing Approach:** Unit tests for the detection logic against both a fabric-manager-configured and fabric-manager-less NetworkClass fixture. No new integration test needed beyond what Story 1.05 and Stories 4.02–4.03 already cover.

**Dependencies:** Story 1.05 (from Epic 1)

---

### Story 4.02: [DEV] OSAC-managed VirtualNetwork-scoped EVPN transit lifecycle

**As a** provider,
**I want to** have OSAC allocate and manage an isolated EVPN transit network for each fabric-backed Secondary VirtualNetwork,
**So that** a tenant can create a VirtualNetwork without performing manual fabric-manager configuration first.

This story closes the provider-allocation gap in the original Epic 4 design. OSAC operates the fabric on the tenant's behalf, but the transit network is VirtualNetwork-scoped: each eligible VirtualNetwork receives an independent MAC-VRF/VNI, provider-selected transit prefix, fabric IPAM allocation, and EVPN CUDN. No transit resource is shared between VirtualNetworks.

**Acceptance Criteria:**
- [ ] Creating a fabric-manager-backed Secondary VirtualNetwork causes OSAC to allocate or reconcile exactly one transit network and matching fabric-side IPAM subnet for that VirtualNetwork through the fabric-manager API.
- [ ] The transit prefix comes from provider configuration, not from the tenant. The preferred value is a statically allocated IPv4 link-local `/24` (for example, `169.254.240.0/24`) with explicit exclusions; if compatibility validation rejects link-local addressing, OSAC uses a provider-configured private prefix. In either mode, OSAC performs overlap checks within the applicable Netris VPC/VRF, against the VirtualNetwork CIDR when they share that VPC/VRF, against platform-reserved link-local addresses, and against fabric-reserved ranges.
- [ ] OSAC obtains or reconciles the fabric resources required for that VirtualNetwork's transit network, including an isolated Netris VPC/VRF or equivalent, a unique MAC-VRF/VNI, and the required site, gateway, node, or fabric-port associations, without requiring manual fabric UI operations.
- [ ] OSAC waits for the VirtualNetwork-specific transit network and IPAM allocation to become usable before publishing the selected transit prefix, VNI, or endpoint information to that VirtualNetwork's dependent CUDN/IPAM resources.
- [ ] Before selecting link-local mode, OSAC verifies CUDN admission/readiness, IPAMClaim allocation, router-pod interface configuration, ARP with the fabric peer, and static routing using link-local next hops over the EVPN L2 segment. If any compatibility check fails, reconciliation selects the configured private-prefix fallback instead of leaving the VN partially provisioned.
- [ ] Two eligible VirtualNetworks never share a transit CUDN, Netris VPC/VRF, MAC-VRF, VNI, or fabric IPAM subnet. They may use the same selected transit prefix only when their VPC/VRF and CUDN isolation makes that reuse unambiguous and safe.
- [ ] Reconciliation is idempotent and safe to retry for the same VirtualNetwork: repeated sequential reconciliation does not allocate a second transit network, CIDR, or VNI.
- [ ] Reconciliation is safe under genuine concurrency: two overlapping reconcile/job invocations for the same VirtualNetwork, running at the same time (not one after the other), still result in exactly one transit network, CIDR, and VNI, with the losing invocation observing real contention rather than proceeding unserialized. This is a distinct requirement from retry-safety above and must be demonstrated with an actual concurrent-execution test, not inferred from the presence of a lock — a structurally-present lock is not evidence it serializes anything (see Implementation Guidance and design.md Risks).
- [ ] Deleting a VirtualNetwork withdraws its fabric routes and removes its transit CUDN/IPAM resources and OSAC-owned fabric IPAM subnet/transit network in dependency order, without affecting any other VirtualNetwork.
- [ ] Fabric API failures, allocation conflicts, and incomplete provisioning are retried through reconciliation and exposed in status/events without leaking orphaned resources.
- [ ] A transit resource deletion that the fabric API repeatedly rejects or fails (not a transient error) surfaces a distinct, dedicated status condition for stuck teardown — separate from the general reconcile-error/event stream — so a stalled, cost-bearing fabric resource cannot be mistaken for ordinary retry transience or missed entirely (design.md, "Stuck transit teardown").

**Implementation Guidance:** Use the existing VirtualNetwork provisioning lifecycle. Prefer deriving deterministic fabric resource names from the VirtualNetwork identity (UUID) over persisting a separate identifier mapping, if Netris's naming constraints (length, character set, uniqueness scope) allow it (design.md, Open Question 11); if they don't, persist the identifiers explicitly and document where (VirtualNetwork status, an annotation, or a separate resource — not left implicit). Do not treat `NETRIS_EVPN_TRANSIT_VNET_NAME` or similar environment variables as a shared source of truth; replace them with provider configuration describing the allocation source. The implementation must support adoption of an already-existing matching VN-specific transit network during upgrade, while refusing to adopt an unrelated shared resource and avoiding deletion of resources that OSAC does not own. A provider/fabric-scoped lock may serialize fabric API operations if required by the backend, but it must not turn the transit resource into a shared network — and whatever locking mechanism is used must be validated under real concurrent execution as part of this story's own test suite (see the concurrency acceptance criterion above), not assumed correct because the code reviews as a mutex. This codebase has a documented precedent for exactly this failure mode: a per-VirtualNetwork Lease-based lock elsewhere in `osac-aap` (`osac.service.vnet_lock`) was a complete no-op for multiple prior stories — its `include_role` calls were missing `tasks_from`, so the lock's own tasks never ran at all — and this went undetected until a genuine concurrent test was run, because nothing before that had ever exercised real concurrency against it.

**Testing Approach:** Integration tests against a fabric-manager test double or environment that exercises the fabric API: link-local compatibility validation with a provider-configured `/24`, private-prefix fallback when either backend rejects link-local addressing, allocation for two VirtualNetworks, verification of distinct VPC/VRF scopes, VNets, VNIs, MAC-VRFs, IPAM subnets, and CUDNs, including a case where both isolated VPC/VRF scopes use the same selected transit prefix; sequential-retry idempotency for repeated reconciliation; a separate, genuinely concurrent reconciliation test (two overlapping invocations, not two sequential ones) asserting single allocation and observable contention on the losing side; adoption of an existing matching VN-specific resource; deletion of one VirtualNetwork without affecting the other; injected failures during allocation/readiness/deletion; and an injected persistent (non-transient) deletion failure verifying the dedicated stuck-teardown status condition appears and later clears. The end-to-end test must succeed without a manual fabric UI setup step.

**Dependencies:** Story 4.01

---

### Story 4.03: [DEV] EVPN transit interface provisioning

**As a** provider,
**I want to** have the router pod join the fabric's transit MAC-VRF automatically when a fabric manager is present,
**So that** bare-metal connectivity (Epic 5) has a working transport to build on.

**Acceptance Criteria:**
- [ ] When Story 4.01 determines a fabric manager is configured, the operator provisions a VirtualNetwork-specific Primary CUDN with EVPN transport on the router pod, reaching that VirtualNetwork's dedicated transit MAC-VRF.
- [ ] The transit interface's address is reserved via IPAMClaim with `ipam.lifecycle: Persistent`, recovering the same address across restarts/reschedules.
- [ ] The transit interface is created once, at VirtualNetwork creation, and is unaffected by individual subnet add/remove.

**Implementation Guidance:** Per design.md, the transit CUDN uses EVPN transport with MAC-VRF only (no IP-VRF) — do not configure an `ipVRF` field. This requires the fabric manager to already support participating as a BGP EVPN peer for cluster-originated MAC-VRFs (an assumption stated in the PRD, not something this story needs to newly validate, but worth confirming is still true in the target environment before relying on it).

**Testing Approach:** Integration test: create a Secondary VirtualNetwork on a fabric-manager-configured NetworkClass, verify the transit interface exists with a persistent IPAMClaim-backed address; restart the router pod, verify the same address is recovered.

**Dependencies:** Story 4.01, Story 4.02

---

### Story 4.04: [QE] E2E validation of conditional transit interface provisioning

**As a** QE engineer,
**I want to** validate that the transit interface is provisioned exactly when it should be,
**So that** fabric-manager-less deployments never accidentally gain bare-metal machinery.

**Acceptance Criteria:**
- [ ] E2E scenario: create a Secondary VirtualNetwork on a fabric-manager-configured NetworkClass, verify the transit interface and its persistent address.
- [ ] E2E scenario: create a Secondary VirtualNetwork on a fabric-manager-less NetworkClass, verify no transit interface is ever created, including after subnet add/remove activity.

**Testing Approach:** Full e2e suite covering both NetworkClass configurations.

**Dependencies:** Story 4.01, Story 4.02, Story 4.03

## Test Case References

Verified by: TC-FR16-01, TC-FR16-02
