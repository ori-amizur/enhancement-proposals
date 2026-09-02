# Epic 2: Live Subnet Attach/Detach

- **T-Shirt Size:** M
- **Phase:** 1
- **PRD Requirements:** FR-9, FR-10, FR-11, FR-12, FR-13
- **Design Reference:** design.md §Proposal (Phase 1, items 4–5), §Workflow Description ("Subnet creation (live-attach path)," "Subnet removal"), §Implementation Details/Notes/Constraints ("Live attachment mechanism," "Fallback trigger"), §9 Primary Mechanism
- **Dependencies:** Epic 1

## Summary

Replaces Epic 1's recreation-only subnet lifecycle with a live-attach-first mechanism: the router pod agent watches a ConfigMap and configures interfaces without a restart, and subnets attach live via `multus-dynamic-networks-controller` when available, falling back to Epic 1's recreation path automatically otherwise. Delivers the "no restart" property this feature is named for, without ever making subnet changes less reliable than Epic 1's baseline.

## Acceptance Criteria

- [ ] A router pod agent watches a ConfigMap and applies gateway-IP, route, and port-security changes to existing interfaces live, without a pod restart.
- [ ] The agent never independently queries or discovers subnet existence — all state it applies originates from the operator via the ConfigMap.
- [ ] When `multus-dynamic-networks-controller` is available and reacts to the router pod, adding a subnet results in live interface injection, detected and configured by the agent, with no restart.
- [ ] When it is unavailable, or the new interface doesn't appear within a defined timeout, the operator falls back to Epic 1's recreation path automatically.
- [ ] Subnet removal follows the same two-tier approach.

## Stories

### Story 2.01: [DEV] Router pod agent — live ConfigMap-driven reconciliation

**As a** provider,
**I want to** have the router pod apply routing and gateway state changes without restarting,
**So that** route/config-level changes never disrupt the VirtualNetwork.

**Acceptance Criteria:**
- [ ] The router pod runs an agent process that watches a ConfigMap written by the operator.
- [ ] For interfaces that already exist on the pod, the agent applies gateway-IP, route, and port-security changes live, without a pod restart.
- [ ] The agent applies exactly what the operator has written — it never queries the fulfillment-service or watches `Subnet` resources directly.

**Implementation Guidance:** Implement as a lightweight watch-and-reconcile loop inside the router pod's container image, replacing a static `sleep infinity`-style placeholder — the agent watches the ConfigMap and applies whatever it says (gateway IPs, routes, port security) without querying any other resource for state.

**Testing Approach:** Unit tests for ConfigMap-diff-to-action translation (given old/new ConfigMap content, verify the correct `ip route`/`ip addr`/port-security calls are made). Integration test: update the ConfigMap for an existing subnet's route, verify it's applied without the pod restarting (check pod start-time/UID unchanged).

**Dependencies:** Story 1.02, Story 1.04 (from Epic 1)

---

### Story 2.02: [DEV] Live subnet attachment via Multus annotation patch

**As a** tenant,
**I want to** have a new subnet attach to the router pod without a restart,
**So that** my VirtualNetwork grows without a connectivity interruption.

**Acceptance Criteria:**
- [ ] When a subnet is added, the operator patches the router pod's Multus network-attachment annotation to add the new Secondary UDN, instead of immediately recreating the pod.
- [ ] When `multus-dynamic-networks-controller` injects the corresponding interface into the running pod, the agent (Story 2.01) detects it and configures it: link up, IPAMClaim-reserved gateway IP, routes, port security.
- [ ] The Subnet is marked Ready with no router pod restart having occurred.

**Implementation Guidance:** Per design.md §9, this depends on an upstream, not-officially-supported-for-non-VM-pods mechanism (`multus-dynamic-networks-controller`) — implement detection of the newly-injected interface via periodic link inspection or netlink events in the agent (Story 2.01), not by assuming a specific notification callback exists. Coordinate with whoever resolves Open Question 6 (controller scoping) before relying on this path in production.

**Testing Approach:** Integration test, gated on `multus-dynamic-networks-controller` being present in the test cluster: add a subnet, verify the interface appears in the running pod and is fully configured without a restart.

**Dependencies:** Story 2.01

---

### Story 2.03: [DEV] Fallback trigger and two-tier subnet removal

**As a** provider,
**I want to** have subnet changes fall back to Epic 1's recreation path automatically when live attachment isn't available,
**So that** the feature never fails outright, only degrades to a known-safe behavior.

**Acceptance Criteria:**
- [ ] If `multus-dynamic-networks-controller` is not installed/available, or the agent does not observe the new interface within a configured timeout, the operator recreates the router pod via Epic 1's existing recreation logic (Story 1.05).
- [ ] Subnet removal follows the same two-tier approach: live detachment where available, recreation fallback otherwise.
- [ ] The chosen detection strategy (one-time capability probe vs. per-change timeout — Open Question 3) is implemented consistently and documented.

**Implementation Guidance:** Reuse Story 1.05's recreation function directly as the fallback branch — do not duplicate that logic. Resolve Open Question 3 (probe vs. timeout) as an explicit implementation decision in this story; if resolved as a one-time probe, check for the controller's deployment/CRDs at VirtualNetwork provisioning time and cache the result. Resolve Open Question 4 (timeout value) with an initial, documented default that can be tuned later.

**Testing Approach:** Integration tests: simulate `multus-dynamic-networks-controller` absence (or a timeout) and verify the fallback path executes correctly for both subnet add and remove; verify pod status/events distinguish "live-attached" from "recreated" (per the SRE user story in design.md).

**Dependencies:** Story 2.02, Story 1.05 (from Epic 1)

---

### Story 2.04: [QE] E2E validation of live attach/detach and fallback

**As a** QE engineer,
**I want to** validate both the live-attach and fallback paths end-to-end,
**So that** regressions in either path are caught before release.

**Acceptance Criteria:**
- [ ] E2E scenario: with `multus-dynamic-networks-controller` present, add and remove a subnet; verify no router pod restart in either direction.
- [ ] E2E scenario: with the controller absent (or forced to time out), add and remove a subnet; verify correct fallback-to-recreation behavior and that existing subnets are unaffected.
- [ ] E2E scenario: router pod failure/restart (independent of subnet changes) — verify normal Kubernetes restart and reconvergence.

**Testing Approach:** Full e2e suite covering both code paths in a real (or kind-based) cluster, toggling `multus-dynamic-networks-controller` availability between test runs.

**Dependencies:** Story 2.01, Story 2.02, Story 2.03

---

### Story 2.05: [DOCS] Document live-attach behavior and its support status

**As an** SRE or provider,
**I want to** understand when subnet changes are live vs. disruptive, and the support status of the underlying mechanism,
**So that** I can set expectations and troubleshoot correctly.

**Documentation Scope:** Readers need to understand: the two-tier behavior (live-attach vs. recreation fallback) and how to tell which occurred from pod events; that `multus-dynamic-networks-controller` is not an officially supported OpenShift capability for non-VM pods, and what that means operationally; expected MAC-based ARP reconvergence behavior when the fallback path triggers.

**Documentation Inputs:**
- **Story 2.02:** the live-attach mechanism and its dependency on `multus-dynamic-networks-controller`.
- **Story 2.03:** the fallback trigger and how to distinguish live-attach from recreation via pod events/status.

**Dependencies:** Story 2.02, Story 2.03

## Test Case References

Verified by: TC-FR9-01, TC-FR10-01, TC-FR11-01, TC-FR12-01, TC-FR13-01
