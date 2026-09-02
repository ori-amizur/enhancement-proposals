# Epic 3: NAT Gateway via EgressIP

- **T-Shirt Size:** S
- **Phase:** 1
- **PRD Requirements:** FR-7, FR-8
- **Design Reference:** design.md §Proposal (Phase 1, item 6), §Workflow Description ("NAT Gateway configuration (no fabric manager)"), §Implementation Details/Notes/Constraints ("NAT Gateway via EgressIP")
- **Dependencies:** Epic 1

## Summary

Gives tenants on fabric-manager-less Secondary VirtualNetworks a dedicated, stable external IP for egress traffic, implemented entirely as an OVN-Kubernetes EgressIP resource — no change to the router pod itself.

## Acceptance Criteria

- [ ] Creating a NATGateway for a fabric-manager-less Secondary VirtualNetwork results in an EgressIP resource selecting the router pod's namespace, using the NATGateway's ExternalIP.
- [ ] VM egress traffic is SNATed to the router pod's own IP (unchanged, existing behavior) and then SNATed again to the NATGateway's ExternalIP as it leaves the node.
- [ ] Without a NATGateway configured, egress behavior is exactly the pre-existing default path.
- [ ] Deleting the NATGateway removes the EgressIP and reverts to default egress.

## Stories

### Story 3.01: [DEV] NATGateway-to-EgressIP controller

**As a** tenant on a fabric-manager-less deployment,
**I want to** configure a NAT Gateway for my Secondary VirtualNetwork,
**So that** my egress traffic uses a known, stable external IP.

**Acceptance Criteria:**
- [ ] The existing NATGateway resource (`virtual_network`/`external_ip` fields, unchanged) is honored for Secondary VirtualNetworks on fabric-manager-less NetworkClasses by creating an OVN-Kubernetes `EgressIP` resource.
- [ ] The `EgressIP` selects the router pod's namespace via `namespaceSelector`, and the router pod's own label via `podSelector` for defense in depth.
- [ ] The `EgressIP`'s address is the NATGateway's ExternalIP.
- [ ] Only one NATGateway per VirtualNetwork is supported, consistent with the existing NATGateway resource's semantics elsewhere in OSAC.
- [ ] Deleting the NATGateway removes the `EgressIP`.
- [ ] The router pod and its agent require no code changes for this feature.

**Implementation Guidance:** This is purely an operator-side controller addition — no router pod or agent involvement (design.md is explicit that the existing default-egress path is untouched). Branch this logic on the same fabric-manager-presence check from Epic 1 Story 1.05 (NAT Gateway for fabric-manager-present VirtualNetworks is out of scope here — delegated to the fabric manager's own mechanism, unaffected by this story).

**Testing Approach:** Integration tests: create a NATGateway, verify the `EgressIP` resource and its selectors/address; verify VM egress traffic is observed with the NATGateway's external IP as source once `EgressIP` reports `Assigned`; delete the NATGateway, verify `EgressIP` removal and reversion to default egress. Unit tests for the EgressIP resource construction and the fabric-manager-presence branch.

**Dependencies:** Story 1.02, Story 1.04, Story 1.05 (from Epic 1)

---

### Story 3.02: [QE] E2E validation of NAT Gateway egress

**As a** QE engineer,
**I want to** validate NAT Gateway behavior end-to-end,
**So that** regressions in egress-IP handling are caught before release.

**Acceptance Criteria:**
- [ ] E2E scenario: allocate an ExternalIP, create a NATGateway, verify VM egress traffic uses that IP as its source.
- [ ] E2E scenario: delete the NATGateway, verify egress reverts to the default (generic, shared) SNAT path.
- [ ] Negative scenario: the NATGateway's ExternalIP is not routable on any node's network — verify the `EgressIP` remains unassigned and the NATGateway surfaces a clear, actionable status rather than silently appearing Ready.

**Testing Approach:** Full e2e suite exercising the scenarios above, including the negative case with a deliberately misconfigured (non-routable) ExternalIPPool CIDR.

**Dependencies:** Story 3.01

---

### Story 3.03: [DOCS] Document ExternalIPPool provisioning for fabric-manager-less NetworkClasses

**As a** provider,
**I want to** understand how to provision an ExternalIPPool that works with EgressIP-based NAT Gateway,
**So that** I set up fabric-manager-less NetworkClasses correctly.

**Documentation Scope:** Readers (providers/admins) need to understand that ExternalIPPool CIDRs for a fabric-manager-less NetworkClass must be valid for node-level EgressIP assignment (e.g., a dedicated L2 segment or secondary NIC on the nodes), the same way CIDRs for a fabric-manager-backed NetworkClass are chosen to suit that fabric's routing mechanism — this is the same existing provisioning discipline applied to a new NetworkClass type, not a new capability.

**Documentation Inputs:**
- **Story 3.01:** the EgressIP-based mechanism and its address-hostability requirement (Open Question 8 / Risk in the design doc).

**Dependencies:** Story 3.01

## Test Case References

Verified by: TC-FR7-01, TC-FR8-01
