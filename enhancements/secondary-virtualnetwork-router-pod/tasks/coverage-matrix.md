# Coverage Matrix — Secondary VirtualNetwork: Router Pod Model with Bare-Metal Connectivity

Requirement IDs are assigned sequentially from the PRD's Acceptance Criteria (§3.1 Phase 1, §3.2 Phase 2) and In-Scope items not otherwise captured by an Acceptance Criterion (dual-attachment namespace targeting, FR-15; DNS resolution, FR-21; route scope to `.1`, FR-22 — identified after the initial decomposition and appended with the next available ID rather than renumbering, per the filename/ID stability convention).

## PRD Requirement → Epic/Story Mapping

| PRD Requirement | Epic | Story | Test Cases | Status |
|---|---|---|---|---|
| FR-1: VirtualNetwork type field, defaults to existing model | Epic 1 | Story 1.01 | TC-FR1-01 | Covered |
| FR-2: type field immutable after creation | Epic 1 | Story 1.01 | TC-FR2-01 | Covered |
| FR-3: Secondary VN provisions one namespace + one router pod + one Secondary UDN per subnet | Epic 1 | Story 1.02, 1.03 | TC-FR3-01, TC-FR3-02 | Covered |
| FR-4: VMs across subnets communicate via router pod `.1` gateway | Epic 1 | Story 1.04 | TC-FR4-01 | Covered |
| FR-5: VMs reach the internet via router pod SNAT egress | Epic 1 | Story 1.04 | TC-FR5-01 | Covered |
| FR-6: No-fabric-manager deployments get router pod without transit interface/static-route dispatch | Epic 1 | Story 1.05 | TC-FR6-01 | Covered |
| FR-7: NATGateway (no fabric manager) implemented via EgressIP SNAT to ExternalIP | Epic 3 | Story 3.01 | TC-FR7-01 | Covered |
| FR-8: Without NATGateway, default egress unaffected | Epic 3 | Story 3.01 | TC-FR8-01 | Covered |
| FR-9: Live subnet attachment via `multus-dynamic-networks-controller`, no restart | Epic 2 | Story 2.02 | TC-FR9-01 | Covered |
| FR-10: Fallback to router pod recreation when live attachment unavailable/times out | Epic 2 | Story 2.03 | TC-FR10-01 | Covered |
| FR-11: Subnet removal follows the same two-tier approach | Epic 2 | Story 2.03 | TC-FR11-01 | Covered |
| FR-12: Route/gateway-IP/port-security changes on existing interfaces always applied live | Epic 2 | Story 2.01 | TC-FR12-01 | Covered |
| FR-13: Agent never independently discovers subnet existence | Epic 2 | Story 2.01 | TC-FR13-01 | Covered |
| FR-14: Existing default-model VirtualNetworks unaffected | Epic 1 | Story 1.01, 1.02 | TC-FR14-01 | Covered |
| FR-15: Dual-attached VMs (Primary UDN elsewhere) can still attach to a Secondary VN subnet via shared-label targeting | Epic 1 | Story 1.03 | TC-FR15-01 | Covered |
| FR-16: Fabric-manager-configured deployment → router pod gets transit interface + bare-metal mechanism | Epic 4 | Story 4.01, 4.02 | TC-FR16-01, TC-FR16-02 | Covered |
| FR-17: Adding/removing VM subnet → static route pushed/withdrawn to fabric manager | Epic 5 | Story 5.01 | TC-FR17-01 | Covered |
| FR-18: Bare-metal subnet relevant → route written to router pod ConfigMap, applied live | Epic 5 | Story 5.02 | TC-FR18-01 | Covered |
| FR-19: Bidirectional VM ↔ bare-metal connectivity once routes in place | Epic 5 | Story 5.04 | TC-FR19-01 | Covered |
| FR-20: Phase 2 deployment doesn't change Phase-1-only VN behavior | Epic 5 | Story 5.04 | TC-FR20-01 | Covered |
| FR-21: DNS resolution — Secondary-VN-only VMs mirror the router pod's own `/etc/resolv.conf` via cloud-init; dual-attached VMs are unaffected (Primary attachment's DNS applies) | Epic 1 | Story 1.04 | TC-FR21-01, TC-FR21-02 | Covered |
| FR-22: Route scope to `.1` via cloud-init — full default route for Secondary-VN-only VMs; VN-CIDR-scoped route (not a default route) for dual-attached VMs, so the Primary attachment's default route is never overridden | Epic 1 | Story 1.04 | TC-FR22-01, TC-FR22-02 | Covered |

## Gaps

All PRD requirements are covered by the decomposition. Two items from the PRD's Risks/Open Questions are deliberately *not* independent requirements with their own test cases, since they're process/documentation items rather than behavioral requirements:

- Open Question 6 (Section 4.01/Story 1.05, Epic 4 Story 4.01): confirming the NetworkClass representation of "no fabric manager" is implementation groundwork for FR-6 and FR-16, not a separately testable behavior.
- Risk 6.6 (unsupported `multus-dynamic-networks-controller` dependency, Epic 2): a product/support sign-off item, not a behavioral requirement — tracked via Story 2.05 (DOCS) rather than a test case.

## Stories Without PRD Traceability

- Story 1.07, 2.05, 3.03, 5.05 (all `[DOCS]`): documentation stories don't map to a single behavioral PRD requirement — each documents the user-facing surface introduced by its epic's `[DEV]` stories. Justified per the decompose skill's guidance that `[DOCS]` stories are documentation-outcome-oriented, not behavioral.
- Story 5.03 (reconciliation consistency): addresses Risk 6.4 directly rather than a numbered PRD Acceptance Criterion. Justified as necessary implementation work to make FR-17/FR-18 reliable under partial failure, called out explicitly as a risk in the PRD.
