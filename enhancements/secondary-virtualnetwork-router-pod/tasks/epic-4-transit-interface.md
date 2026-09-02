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
- [ ] When one is configured, the router pod is provisioned with an additional transit interface — a Primary CUDN with EVPN transport — reaching the fabric's shared transit MAC-VRF.
- [ ] The transit interface's address is reserved via IPAMClaim with `ipam.lifecycle: Persistent`.
- [ ] When no fabric manager is configured, none of this is provisioned (verified as a negative case here, building on Epic 1 Story 1.05's initial check).

## Stories

### Story 4.01: [DEV] NetworkClass fabric-manager detection

**As a** provider,
**I want to** have the operator reliably determine whether a NetworkClass has a fabric manager configured,
**So that** bare-metal machinery is provisioned only where it can actually work.

**Acceptance Criteria:**
- [ ] The operator resolves a VirtualNetwork's target NetworkClass and determines fabric-manager presence/absence unambiguously.
- [ ] The determination is made once, at VirtualNetwork creation, and does not change over the VirtualNetwork's lifetime.

**Implementation Guidance:** This story resolves Open Question 7 (design.md) — confirm against the current NetworkClass API how "no fabric manager" is actually represented (OSAC's Unified Networking model generally treats `fabricManager` as required, so this may correspond to a distinct NetworkClass configuration, e.g. a `udn-net`-style class, rather than an empty/optional value on the same field). Do not guess at the API shape — verify against the actual NetworkClass CRD/controller before implementing. This determination should reuse or replace the initial check from Epic 1 Story 1.05 (which only needed to skip Phase 2 provisioning) with the confirmed, authoritative mechanism.

**Testing Approach:** Unit tests for the detection logic against both a fabric-manager-configured and fabric-manager-less NetworkClass fixture. No new integration test needed beyond what Story 1.05 and Story 4.02 already cover.

**Dependencies:** Story 1.05 (from Epic 1)

---

### Story 4.02: [DEV] EVPN transit interface provisioning

**As a** provider,
**I want to** have the router pod join the fabric's transit MAC-VRF automatically when a fabric manager is present,
**So that** bare-metal connectivity (Epic 5) has a working transport to build on.

**Acceptance Criteria:**
- [ ] When Story 4.01 determines a fabric manager is configured, the operator provisions a Primary CUDN with EVPN transport on the router pod, reaching the fabric's shared transit MAC-VRF.
- [ ] The transit interface's address is reserved via IPAMClaim with `ipam.lifecycle: Persistent`, recovering the same address across restarts/reschedules.
- [ ] The transit interface is created once, at VirtualNetwork creation, and is unaffected by individual subnet add/remove.

**Implementation Guidance:** Per design.md, the transit CUDN uses EVPN transport with MAC-VRF only (no IP-VRF) — do not configure an `ipVRF` field. This requires the fabric manager to already support participating as a BGP EVPN peer for cluster-originated MAC-VRFs (an assumption stated in the PRD, not something this story needs to newly validate, but worth confirming is still true in the target environment before relying on it).

**Testing Approach:** Integration test: create a Secondary VirtualNetwork on a fabric-manager-configured NetworkClass, verify the transit interface exists with a persistent IPAMClaim-backed address; restart the router pod, verify the same address is recovered.

**Dependencies:** Story 4.01

---

### Story 4.03: [QE] E2E validation of conditional transit interface provisioning

**As a** QE engineer,
**I want to** validate that the transit interface is provisioned exactly when it should be,
**So that** fabric-manager-less deployments never accidentally gain bare-metal machinery.

**Acceptance Criteria:**
- [ ] E2E scenario: create a Secondary VirtualNetwork on a fabric-manager-configured NetworkClass, verify the transit interface and its persistent address.
- [ ] E2E scenario: create a Secondary VirtualNetwork on a fabric-manager-less NetworkClass, verify no transit interface is ever created, including after subnet add/remove activity.

**Testing Approach:** Full e2e suite covering both NetworkClass configurations.

**Dependencies:** Story 4.01, Story 4.02

## Test Case References

Verified by: TC-FR16-01, TC-FR16-02
