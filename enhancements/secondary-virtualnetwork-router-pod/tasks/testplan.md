# Testplan — Secondary VirtualNetwork: Router Pod Model with Bare-Metal Connectivity

## Overview

- **Feature:** TBD (no Jira Feature exists yet)
- **Total test cases:** 26
- **Requirements covered:** 22 of 22

## Test Cases

### FR-1: VirtualNetwork type field, defaults to Primary

#### TC-FR1-01: Creating a VirtualNetwork without the type field defaults to Primary

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.01 | AC-1 | critical | automated |

##### Preconditions
- Tenant has permission to create VirtualNetworks.

##### Steps
1. Create a VirtualNetwork without setting the networking-type field.
2. Create a second VirtualNetwork with the networking-type field explicitly set to `Secondary`.
3. Read back both VirtualNetworks' status/spec.

##### Expected Results
- The first VirtualNetwork's networking-type field reads `Primary`.
- The second VirtualNetwork's networking-type field reads `Secondary`, and its namespace/router pod are provisioned (per Story 1.02).

---

### FR-2: type field immutable after creation

#### TC-FR2-01: Attempting to change networking-type after creation is rejected

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.01 | AC-2 | high | automated |

##### Preconditions
- A VirtualNetwork exists with networking-type `Primary`.

##### Steps
1. Submit an update to the VirtualNetwork changing networking-type to `Secondary`.

##### Expected Results
- The update is rejected with a validation error identifying the networking-type field as immutable.
- The VirtualNetwork's networking-type remains `Primary`.

---

### FR-3: Secondary VN provisions one namespace + one router pod + one Secondary UDN per subnet

#### TC-FR3-01: Creating a Secondary VirtualNetwork provisions exactly one namespace and router pod

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.02 | AC-1 | critical | automated |

##### Preconditions
- Tenant has permission to create VirtualNetworks.

##### Steps
1. Create a VirtualNetwork with networking-type `Secondary`.
2. Query the cluster for namespaces and pods labeled for this VirtualNetwork.

##### Expected Results
- Exactly one namespace exists for the VirtualNetwork.
- Exactly one router pod exists in that namespace, with a single cluster-network interface and zero subnet interfaces.

#### TC-FR3-02: Creating a Subnet provisions a Secondary CUDN and a router pod interface

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.02, 1.03 | AC-2, AC-3 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists (TC-FR3-01).

##### Steps
1. Create a Subnet referencing the VirtualNetwork with CIDR `10.0.1.0/24`.
2. Query the cluster for the Secondary CUDN and the router pod's interfaces.

##### Expected Results
- A ClusterUserDefinedNetwork with `role: Secondary` exists targeting the router pod's namespace via the shared label.
- The router pod has a new interface with address `10.0.1.1`.

---

### FR-4: VMs across subnets communicate via router pod `.1` gateway

#### TC-FR4-01: Two VMs in different subnets of the same Secondary VirtualNetwork can reach each other

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.04 | AC-1 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with two Subnets (`10.0.1.0/24`, `10.0.2.0/24`).
- A VM (`VM-A`) is running on the first subnet, a VM (`VM-B`) on the second.

##### Steps
1. From `VM-A`, send an ICMP echo request to `VM-B`'s IP address.

##### Expected Results
- `VM-A` receives an ICMP echo reply from `VM-B`'s IP address within 2 seconds.

---

### FR-5: VMs reach the internet via router pod SNAT egress

#### TC-FR5-01: A VM reaches an external address via the router pod's SNAT path

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.04 | AC-2 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with one Subnet and one running VM.
- An externally-reachable test endpoint is available.

##### Steps
1. From the VM, send an HTTP GET request to the test endpoint.
2. Capture the source IP address observed by the test endpoint.

##### Expected Results
- The HTTP request returns a 200 status code.
- The source IP address observed by the test endpoint is the router pod's cluster-network IP, not the VM's own subnet IP.

---

### FR-6: No-fabric-manager deployments get router pod without transit interface/static-route dispatch

#### TC-FR6-01: A Secondary VirtualNetwork on a fabric-manager-less NetworkClass never gains a transit interface

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.05 | AC-3 | high | automated |

##### Preconditions
- A NetworkClass exists with no fabric manager configured.

##### Steps
1. Create a Secondary VirtualNetwork targeting that NetworkClass.
2. Add and remove two Subnets.
3. Query the router pod's interfaces after each step.

##### Expected Results
- At no point does the router pod have a transit interface.
- No static route API calls are made to any fabric manager (verified via absence of corresponding dispatcher/AAP job records).

---

### FR-7: NATGateway (no fabric manager) implemented via EgressIP SNAT to ExternalIP

#### TC-FR7-01: Configuring a NATGateway results in VM egress using the NATGateway's external IP

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 3.01 | AC-1, AC-2, AC-3 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists on a fabric-manager-less NetworkClass, with a running VM.
- An ExternalIP is allocated from an ExternalIPPool provisioned with node-assignable CIDRs.

##### Steps
1. Create a NATGateway referencing the VirtualNetwork and the ExternalIP.
2. Wait for the backing `EgressIP` resource to report `Assigned`.
3. From the VM, send an HTTP GET request to a test endpoint and capture the observed source IP.

##### Expected Results
- The `EgressIP` resource's status shows `Assigned` with the ExternalIP's address.
- The test endpoint observes the NATGateway's ExternalIP as the source IP, not the router pod's cluster-network IP.

---

### FR-8: Without NATGateway, default egress unaffected

#### TC-FR8-01: Deleting a NATGateway reverts egress to the default SNAT path

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 3.01 | AC-4 | high | automated |

##### Preconditions
- TC-FR7-01 has been executed and the NATGateway is Ready.

##### Steps
1. Delete the NATGateway.
2. From the VM, send an HTTP GET request to the test endpoint and capture the observed source IP.

##### Expected Results
- The `EgressIP` resource no longer exists.
- The test endpoint observes the router pod's cluster-network IP as the source, matching TC-FR5-01's default behavior.

---

### FR-9: Live subnet attachment via `multus-dynamic-networks-controller`, no restart

#### TC-FR9-01: Adding a subnet with the controller present attaches live with no pod restart

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 2.02 | AC-1, AC-2, AC-3 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with one Subnet.
- `multus-dynamic-networks-controller` is installed and confirmed reacting to the router pod (Open Question 6 resolved favorably in this test environment).
- The router pod's current UID/start-time is recorded.

##### Steps
1. Create a second Subnet referencing the VirtualNetwork.
2. Poll the router pod's interfaces until the new one appears.
3. Re-read the router pod's UID/start-time.

##### Expected Results
- The new interface appears within the configured timeout with the correct IPAMClaim-reserved gateway address.
- The router pod's UID/start-time is unchanged from before Step 1 (no restart occurred).
- The Subnet's status reports Ready.

---

### FR-10: Fallback to router pod recreation when live attachment unavailable/times out

#### TC-FR10-01: Adding a subnet with the controller absent falls back to recreation

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 2.03 | AC-1 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with one Subnet.
- `multus-dynamic-networks-controller` is not installed in the test cluster.
- The router pod's current UID/start-time is recorded.

##### Steps
1. Create a second Subnet referencing the VirtualNetwork.
2. Wait past the configured live-attach timeout.
3. Re-read the router pod's UID/start-time and interfaces.

##### Expected Results
- The router pod's UID/start-time differs from before Step 1 (recreation occurred).
- The recreated pod has interfaces for both the original and new subnet, with the original subnet's gateway IP unchanged from before recreation.
- The Subnet's status reports Ready.

---

### FR-11: Subnet removal follows the same two-tier approach

#### TC-FR11-01: Removing a subnet detaches live when the controller is present

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 2.03 | AC-2 | high | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with two Subnets, live-attached per TC-FR9-01.
- The router pod's current UID/start-time is recorded.

##### Steps
1. Delete the second Subnet.
2. Poll the router pod's interfaces until the corresponding one disappears.
3. Re-read the router pod's UID/start-time.

##### Expected Results
- The interface for the removed subnet is gone from the router pod within the configured timeout.
- The router pod's UID/start-time is unchanged (no restart occurred).
- The remaining subnet's connectivity (per TC-FR4-01) is unaffected.

---

### FR-12: Route/gateway-IP/port-security changes on existing interfaces always applied live

#### TC-FR12-01: A route change to the router pod's ConfigMap is applied without a restart

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 2.01 | AC-2 | high | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with one Subnet.
- The router pod's current UID/start-time is recorded.

##### Steps
1. Update the router pod's ConfigMap to add a new route on the existing subnet interface.
2. Query the router pod's routing table.
3. Re-read the router pod's UID/start-time.

##### Expected Results
- The new route appears in the router pod's routing table (`ip route` output includes the added route).
- The router pod's UID/start-time is unchanged (no restart occurred).

---

### FR-13: Agent never independently discovers subnet existence

#### TC-FR13-01: The agent applies only ConfigMap-supplied state, with no direct Subnet-resource access

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 2.01 | AC-3 | medium | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with one Subnet.
- The agent's RBAC permissions are known/inspectable.

##### Steps
1. Inspect the agent's service account RBAC bindings.
2. Modify the router pod's ConfigMap directly (bypassing the operator) to remove the existing subnet's route.
3. Query the router pod's routing table.

##### Expected Results
- The agent's service account has no `get`/`list`/`watch` permission on the `Subnet` custom resource.
- The router pod's routing table reflects the manually-edited ConfigMap (route removed), confirming the agent applies ConfigMap content directly rather than re-deriving it from `Subnet` state.

---

### FR-14: Existing Primary-model VirtualNetworks unaffected

#### TC-FR14-01: A pre-existing Primary-type VirtualNetwork is unchanged after this feature is deployed

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.01, 1.02 | AC-4 | critical | automated |

##### Preconditions
- A VirtualNetwork with networking-type `Primary` exists, created before this feature's rollout, with a running VM.

##### Steps
1. Deploy this feature's osac-operator changes to the cluster.
2. Verify the existing VirtualNetwork's resources and connectivity.

##### Expected Results
- No namespace, router pod, or Secondary CUDN is created for the existing VirtualNetwork.
- The existing VM's connectivity (inter-subnet if applicable, egress) is unchanged from before the deployment.

---

### FR-15: Dual-attached VMs (Primary UDN elsewhere) can still attach to a Secondary VN subnet via shared-label targeting

#### TC-FR15-01: A VM with a Primary UDN elsewhere reaches its Secondary VN subnet from its own namespace

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.03 | AC-1, AC-2 | high | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with one Subnet.
- A VM attached via OSAC's Primary VirtualNetwork type (which uses a Primary UDN) exists in a separate namespace, `tenant-ns-A`.

##### Steps
1. Create a VM in `tenant-ns-A` with attachments to both the Primary UDN and the Secondary VN's subnet.
2. Verify the VM's namespace.
3. From the VM, send an ICMP echo request to another VM on the same Secondary VN subnet.

##### Expected Results
- The VM is created in `tenant-ns-A`, not the router pod's namespace.
- The ICMP echo reply is received within 2 seconds, confirming subnet reachability from the dual-attached VM's own namespace.

---

### FR-16: Fabric-manager-configured deployment → router pod gets transit interface + bare-metal mechanism

#### TC-FR16-01: A Secondary VirtualNetwork on a fabric-manager-configured NetworkClass gains a transit interface

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 4.01, 4.02 | AC-1, AC-2 | critical | automated |

##### Preconditions
- A NetworkClass exists with a fabric manager configured.

##### Steps
1. Create a Secondary VirtualNetwork targeting that NetworkClass.
2. Query the router pod's interfaces.

##### Expected Results
- The router pod has a transit interface (a Primary CUDN with EVPN transport) in addition to the cluster-network interface.
- The transit interface's address is reserved via an IPAMClaim with `ipam.lifecycle: Persistent`.

#### TC-FR16-02: The transit interface's address persists across a router pod restart

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 4.02 | AC-2 | high | automated |

##### Preconditions
- TC-FR16-01 has been executed; the transit interface's address is recorded.

##### Steps
1. Force a router pod restart (delete the pod).
2. Query the recreated router pod's transit interface address.

##### Expected Results
- The recreated router pod's transit interface has the same address as recorded before the restart.

---

### FR-17: Adding/removing VM subnet → static route pushed/withdrawn to fabric manager

#### TC-FR17-01: Adding a VM subnet pushes a static route to the fabric manager

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 5.01 | AC-1, AC-2 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists on a fabric-manager-configured NetworkClass with a transit interface (TC-FR16-01).

##### Steps
1. Create a Subnet (`10.0.1.0/24`) referencing the VirtualNetwork.
2. Query the fabric manager's route table for the VN's VRF.
3. Delete the Subnet.
4. Re-query the fabric manager's route table.

##### Expected Results
- After Step 2, the fabric manager's route table contains an entry for `10.0.1.0/24` with the router pod's transit address as next-hop.
- After Step 4, that route entry is no longer present.

---

### FR-18: Bare-metal subnet relevant → route written to router pod ConfigMap, applied live

#### TC-FR18-01: Configuring a bare-metal subnet route updates the router pod's routing table live

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 5.02 | AC-1, AC-2 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists on a fabric-manager-configured NetworkClass with a transit interface.
- A bare-metal subnet (`10.0.3.0/24`) exists on the fabric.
- The router pod's current UID/start-time is recorded.

##### Steps
1. Configure the bare-metal subnet as relevant to the VirtualNetwork (per the operator's provisioning flow).
2. Query the router pod's routing table.
3. Re-read the router pod's UID/start-time.

##### Expected Results
- The router pod's routing table contains a route for `10.0.3.0/24` via the fabric peer's transit address over the transit interface.
- The router pod's UID/start-time is unchanged (no restart occurred).

---

### FR-19: Bidirectional VM ↔ bare-metal connectivity once routes in place

#### TC-FR19-01: A VM and a bare-metal server reach each other once both routes exist

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 5.04 | AC-1 | critical | automated |

##### Preconditions
- TC-FR17-01 and TC-FR18-01 have both been executed successfully for the same VirtualNetwork/bare-metal subnet pair.
- A VM is running on the VN subnet; a bare-metal server is running on the bare-metal subnet.

##### Steps
1. From the VM, send an ICMP echo request to the bare-metal server's IP.
2. From the bare-metal server, send an ICMP echo request to the VM's IP.

##### Expected Results
- Both ICMP echo requests receive replies within 2 seconds.

---

### FR-20: Phase 2 deployment doesn't change Phase-1-only VN behavior

#### TC-FR20-01: A fabric-manager-less Secondary VirtualNetwork is unaffected after Phase 2 is deployed

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 5.04 | AC-2, AC-3 | high | automated |

##### Preconditions
- A Secondary VirtualNetwork exists on a fabric-manager-less NetworkClass (Phase 1 only), with working inter-subnet connectivity and egress established before Phase 2's osac-operator changes are deployed.

##### Steps
1. Deploy Phase 2 (Epics 4–5) to the cluster.
2. Re-verify inter-subnet connectivity (per TC-FR4-01) and egress (per TC-FR5-01) on the pre-existing VirtualNetwork.
3. Query the router pod's interfaces.

##### Expected Results
- Inter-subnet connectivity and egress behave identically to before the Phase 2 deployment.
- The router pod still has no transit interface.

---

### FR-21: DNS resolution — Secondary-VN-only VMs mirror the router pod's own resolv.conf; dual-attached VMs unaffected

#### TC-FR21-01: A Secondary-VN-only VM resolves both external and cluster-internal DNS names via the router pod's own resolver

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.04 | AC-5, AC-7 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with one Subnet and a running VM with no Primary UDN attachment elsewhere.

##### Steps
1. Read the VM's `/etc/resolv.conf`.
2. Read the router pod's own `/etc/resolv.conf`.
3. From the VM, run a DNS lookup for a public hostname (e.g., `example.com`).
4. From the VM, run a DNS lookup for a cluster-internal service name resolvable via CoreDNS (e.g., `kubernetes.default.svc.cluster.local`).

##### Expected Results
- The VM's `/etc/resolv.conf` content is identical to the router pod's own `/etc/resolv.conf` content read in Step 2.
- The public hostname lookup returns a valid IP address.
- The cluster-internal service name lookup returns the correct ClusterIP for that service, confirming OVN's Service DNAT applies correctly to traffic forwarded through the router pod (design.md Open Question 9).

#### TC-FR21-02: A dual-attached VM's DNS configuration is unaffected by the Secondary VN subnet

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.04 | AC-6 | high | automated |

##### Preconditions
- A VM exists with both a Primary UDN attachment (in its own namespace) and a Secondary VN subnet attachment (per TC-FR15-01).
- The VM's `/etc/resolv.conf` content prior to the Secondary VN attachment is recorded.

##### Steps
1. Attach the VM to the Secondary VN subnet (if not already done in TC-FR15-01).
2. Read the VM's `/etc/resolv.conf`.

##### Expected Results
- The VM's `/etc/resolv.conf` is unchanged from the value recorded before the Secondary VN attachment — it does not contain the router pod's resolver configuration.

---

### FR-22: Route scope to `.1` via cloud-init — full default route for Secondary-VN-only VMs; VN-CIDR-scoped route for dual-attached VMs

#### TC-FR22-01: A Secondary-VN-only VM gets a full default route to `.1` and reaches every subnet in the VN

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.04 | AC-4 | critical | automated |

##### Preconditions
- A Secondary VirtualNetwork exists with two Subnets, each with a running VM, none with a Primary UDN attachment elsewhere.

##### Steps
1. Read the routing table of the VM on the first subnet.
2. From that VM, send an ICMP echo request to the VM on the second subnet.
3. From that VM, send an ICMP echo request to an external address.

##### Expected Results
- The VM's routing table shows a default route (`0.0.0.0/0`) via `.1` on its local subnet interface.
- Both ICMP echo replies are received within 2 seconds, confirming inter-subnet and internet reachability via the single injected default route.

#### TC-FR22-02: A dual-attached VM gets a VN-CIDR-scoped route (not a default route) via `.1`, and its Primary attachment's default route is unchanged

| Story | AC | Priority | Automation |
|-------|-----|----------|------------|
| Story 1.04 | AC-4, AC-5 | critical | automated |

##### Preconditions
- A VM exists with both a Primary UDN attachment (in its own namespace) and a Secondary VN subnet attachment (per TC-FR15-01), on a Secondary VirtualNetwork with at least two Subnets.
- The VM's default route prior to the Secondary VN attachment is recorded.

##### Steps
1. Read the VM's routing table after the Secondary VN subnet attachment.
2. From the VM, send an ICMP echo request to a VM on the *other* Subnet of the same Secondary VirtualNetwork (one it is not directly attached to).

##### Expected Results
- The VM's default route is unchanged from the value recorded before the Secondary VN attachment — it still points at the Primary attachment's gateway, not `.1`.
- The routing table additionally shows a route for the Secondary VirtualNetwork's own CIDR via `.1` on the Secondary VN subnet interface.
- The ICMP echo reply from the other subnet's VM is received within 2 seconds, confirming the VN-CIDR-scoped route correctly reaches a subnet the VM is not directly attached to, via the router pod's inter-subnet forwarding.

## Gaps

All covered requirements have test cases and all story ACs referenced above are mapped to test cases. Two story ACs are not independently test-cased because they describe internal implementation constraints verified indirectly by other test cases in this plan, not separate observable behaviors:

- Story 1.05's fabric-manager-detection groundwork (superseded by Story 4.01's confirmed implementation) — verified via TC-FR6-01 and TC-FR16-01 rather than a dedicated case for the interim check.
- Story 2.01's "no independent discovery" AC beyond RBAC inspection (TC-FR13-01 covers the RBAC angle; the ConfigMap-only-source-of-truth property is otherwise exercised implicitly by every other live-reconciliation test case in this plan).

## Summary

| Metric | Count |
|--------|-------|
| Total test cases | 26 |
| Critical | 16 |
| High | 9 |
| Medium | 1 |
| Low | 0 |
| Automated | 26 |
| Manual | 0 |
| Requirements with test cases | 22 / 22 |
| Requirements without test cases | 0 |
