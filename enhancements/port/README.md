---
title: port-network-interface-abstraction
authors:
  - oamizur@redhat.com
creation-date: 2026-06-24
last-updated: 2026-06-24
tracking-link:
  - TBD
see-also:
  - "/enhancements/networking"
  - "/enhancements/unified-networking"
replaces:
  - N/A
superseded-by:
  - N/A
---

# Port: Network Interface Abstraction

## Summary

This enhancement introduces a **Port** resource to OSAC as a lightweight
abstraction for a network interface. A Port represents a single connection point
between a resource and a Subnet, carrying a private IP address. Ports are
auto-created when a ComputeInstance (or BareMetalInstance) is provisioned---one
per network attachment---and auto-deleted when the parent resource is destroyed.
The primary motivation is to disambiguate ExternalIP attachment for multi-NIC
resources: instead of targeting an entire ComputeInstance, an
ExternalIPAttachment targets a specific Port, identifying exactly which network
interface receives the external IP.

## Motivation

Today, `ExternalIPAttachment` targets a ComputeInstance by ID. This works when a
VM has a single network interface, but ComputeInstance already supports multiple
network attachments (`spec.network_attachments[]`), each connecting the VM to a
different Subnet. When a tenant wants to attach an ExternalIP to a multi-NIC VM,
the current API cannot express *which* NIC should receive the traffic.

Cloud providers solve this the same way: AWS Elastic IPs attach to ENIs (Elastic
Network Interfaces), not to EC2 instances. OpenStack Floating IPs attach to
Ports, not to VMs. OSAC needs an equivalent abstraction.

The unified-networking enhancement (OSAC-1029) addresses multi-NIC with a
`primary` flag on network attachments, which determines the default gateway,
SNAT source, and DNAT target. The `primary` flag is the right tool for routing
and NAT identity, but it limits ExternalIP attachment to a single NIC. This
proposal complements `primary` with auto-created Port resources, enabling
independent ExternalIP attachment to any NIC. See
[Alternative 2](#alternative-2-primary-network-attachment-flag-without-port)
for a detailed comparison.

### User Stories

- As a tenant, I want to attach an ExternalIP to a specific network interface on
  my multi-NIC VM, so that external traffic is routed to the correct Subnet and
  NIC rather than being ambiguously assigned to the VM as a whole.

- As a tenant, I want to see which Ports (network interfaces) my ComputeInstance
  has and what private IP each was assigned, so that I can choose the right Port
  when creating an ExternalIPAttachment.

- As a tenant, I want Ports to be automatically managed when I create or delete a
  ComputeInstance, so that I don't need to manually create or clean up network
  interface resources.

- As a platform operator, I want Port lifecycle to be bound to the parent
  resource with Kubernetes ownerReferences, so that garbage collection is
  automatic and orphaned Ports cannot accumulate.

### Goals

- Introduce Port as a first-class OSAC resource (proto, database, CRD) that
  represents a network interface on a Subnet
- Auto-create Ports when a ComputeInstance is created (one per
  `network_attachment`) and auto-delete them with the parent
- Add `port` as a target option in ExternalIPAttachment, allowing ExternalIPs to
  be attached to a specific network interface
- Surface Port ID and assigned IP address as output-only fields on
  ComputeInstance's `network_attachments[]` entries
- Maintain backward compatibility: existing `compute_instance` target in
  ExternalIPAttachment continues to work for single-NIC VMs during migration

### Non-Goals

- Ports for NATGateway: a NATGateway has a single external-facing connection and
  does not need a Port for disambiguation. NATGateway remains a direct target in
  ExternalIPAttachment.
- Ports for Cluster: A Cluster's API and ingress endpoints are not network
  interfaces on a subnet---they are opaque service endpoints whose
  implementation varies (LoadBalancer, VIP, Route, etc.). The Port abstraction
  (a NIC on a subnet) does not map naturally to these endpoints. Cluster
  continues to use the existing `target_endpoint` enum (API/INGRESS) for
  disambiguation, which matches the fixed, well-known nature of these
  endpoints. If Cluster internals evolve to expose concrete network interfaces,
  Ports can be extended to Clusters at that time.
- User-created or detachable Ports: Ports are system-managed and bound to their
  parent resource's lifecycle. Independent Port creation, deletion, or
  reassignment is out of scope.
- SecurityGroups on Ports: SecurityGroups remain on
  `ComputeInstance.spec.network_attachments[]`, not on the Port. A Port
  represents a network connection point; firewall rules are a property of the
  workload, not the interface.
- IPAM management: Port records the IP address assigned to the interface. The
  mechanism for IP allocation (DHCP, static, OVN IPAM) is determined by the
  NetworkClass implementation, not by the Port resource.

## Proposal

A Port is a lean resource with a bound lifecycle:

- **Created automatically** when a ComputeInstance is provisioned---one Port per
  entry in `spec.network_attachments[]`
- **Deleted automatically** when the parent ComputeInstance is deleted, via
  Kubernetes ownerReferences
- **Lives in the same namespace** as the ComputeInstance CR
- **References a Subnet** in the same namespace (plain name, no namespace
  qualification needed)

The Port carries minimal fields:

| Field | Description |
|-------|-------------|
| `spec.subnetRef` | Subnet CR name (same namespace) |
| `status.ipAddress` | Private IP assigned from the Subnet's CIDR |
| `status.ownerType` | Resource kind that owns this Port (e.g., `ComputeInstance`) |
| `status.ownerRef` | Name of the owning resource |
| `status.state` | PENDING, READY, FAILED, DELETING |

ExternalIPAttachment gains `port` as an additional option in its `oneof target`:

```
oneof target {
  string compute_instance = 2;  // existing, to be deprecated
  string cluster = 3;           // existing, unchanged
  string baremetal_instance = 4; // existing, to be deprecated in favor of port
  string port = 6;              // NEW
}
```

For ComputeInstance and BareMetalInstance, `port` replaces the direct resource
reference. For Cluster, the existing `cluster` + `target_endpoint` mechanism
remains unchanged.

### Workflow Description

#### ComputeInstance Creation with Ports

**Tenant** creates a ComputeInstance with multiple network attachments.

1. The tenant uses the OSAC CLI to create a ComputeInstance with two NICs:

       $ ./osac create computeinstance \
              --name my-vm \
              --template ocp_virt_vm \
              --network-attachment subnet=frontend-subnet,security-groups=sg-web \
              --network-attachment subnet=backend-subnet,security-groups=sg-db

2. The Fulfillment Service validates the request, creates the ComputeInstance
   record, and **auto-creates two Port records** in the same transaction:
   - Port A: `subnetRef=frontend-subnet`, `ownerType=ComputeInstance`,
     `ownerRef=my-vm`
   - Port B: `subnetRef=backend-subnet`, `ownerType=ComputeInstance`,
     `ownerRef=my-vm`

3. The Fulfillment Service populates output-only fields on each network
   attachment in the response:

   ```json
   {
     "spec": {
       "network_attachments": [
         {
           "subnet": "frontend-subnet",
           "security_groups": ["sg-web"],
           "port": "port-abc123",
           "ip_address": "10.0.1.15"
         },
         {
           "subnet": "backend-subnet",
           "security_groups": ["sg-db"],
           "port": "port-def456",
           "ip_address": "10.0.2.7"
         }
       ]
     }
   }
   ```

4. The reconciler creates Port CRs on the hub cluster. The osac-operator creates
   Port CRs with `ownerReferences` pointing to the ComputeInstance CR.

5. The Port controller reconciles each Port, assigns an IP address (via the
   NetworkClass implementation), and reports status back.

#### ExternalIP Attachment to a Specific Port

**Tenant** attaches an ExternalIP to the frontend NIC of their multi-NIC VM.

1. The tenant lists ports for their VM:

       $ ./osac port list --owner my-vm
       NAME           SUBNET            IP ADDRESS    STATE
       port-abc123    frontend-subnet   10.0.1.15     Ready
       port-def456    backend-subnet    10.0.2.7      Ready

2. The tenant creates an ExternalIPAttachment targeting the frontend port:

       $ ./osac create externalipattachment \
              --external-ip my-eip \
              --port port-abc123 \
              --name my-attachment

3. The Fulfillment Service validates:
   - The ExternalIP exists and is in ALLOCATED state
   - The Port exists and is in READY state
   - No other ExternalIPAttachment is bound to this Port
   - No other ExternalIPAttachment is bound to this ExternalIP

4. The ExternalIPAttachment is created. The osac-operator's
   ExternalIPAttachment controller resolves the Port to find:
   - Owner type: ComputeInstance
   - Owner name: my-vm
   - Subnet: frontend-subnet

5. The controller dispatches provisioning based on owner type. For
   ComputeInstance, it triggers the AAP playbook which creates a MetalLB
   LoadBalancer Service in the Subnet's namespace, routing the ExternalIP to the
   VM.

#### ComputeInstance Deletion

1. The tenant deletes the ComputeInstance.
2. The Fulfillment Service detaches any ExternalIPAttachments bound to the
   instance's Ports (setting `ExternalIP.status.attached = false`).
3. The Fulfillment Service deletes the Port records.
4. On the hub cluster, ownerReferences cause Kubernetes to garbage-collect the
   Port CRs when the ComputeInstance CR is deleted.

### API Extensions

#### Port CRD (New)

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: Port
metadata:
  name: port-abc123
  namespace: osac-system          # same namespace as ComputeInstance
  ownerReferences:
  - apiVersion: osac.openshift.io/v1alpha1
    kind: ComputeInstance
    name: my-vm
    uid: <compute-instance-uid>
spec:
  subnetRef: frontend-subnet      # Subnet CR name in same namespace
status:
  state: Ready                    # Pending | Ready | Failed | Deleting
  ipAddress: 10.0.1.15
  ownerType: ComputeInstance
  ownerRef: my-vm
```

#### Modified: ComputeInstance NetworkAttachment

Output-only fields are added to `NetworkAttachment`:

```protobuf
message NetworkAttachment {
  string subnet = 1;
  repeated string security_groups = 2;
  string port = 3;        // OUTPUT_ONLY - Port ID, set by system
  string ip_address = 4;  // OUTPUT_ONLY - Private IP from Port status
}
```

#### Modified: ExternalIPAttachment target oneof

```protobuf
message ExternalIPAttachmentSpec {
  string external_ip = 1;

  oneof target {
    string compute_instance = 2;   // DEPRECATED - use port for multi-NIC
    string cluster = 3;            // unchanged
    string baremetal_instance = 4; // DEPRECATED - use port for multi-NIC
    string port = 6;               // NEW - Port ID
  }

  ExternalIPAttachmentEndpoint target_endpoint = 5; // required when target=cluster
}
```

#### ExternalIPAttachment CR (when targeting a Port)

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: ExternalIPAttachment
metadata:
  name: my-attachment
spec:
  externalIP: my-eip
  target:
    kind: Port
    name: port-abc123
status:
  state: Active
  externalIPAddress: 203.0.113.45
```

### Implementation Details/Notes/Constraints

#### Fulfillment Service

**Proto:**
- New `port_type.proto` defining Port message with spec (subnetRef) and status
  (ipAddress, ownerType, ownerRef, state)
- New `port_service.proto` with Get, List, and Signal RPCs (no public
  Create/Delete---Ports are managed internally)
- Modified `compute_instance_type.proto`: add output-only `port` and `ip_address`
  fields to `NetworkAttachment` message
- Modified `external_ip_attachment_type.proto`: add `port` (field 6) to the
  `oneof target`

**Database:**
- New migration creating `ports` and `archived_ports` tables following existing
  patterns (id, name, timestamps, finalizers, creator, tenant, labels,
  annotations, data jsonb, version)
- New unique partial index on ExternalIPAttachment:
  `one_per_port` on `data->'spec'->>'port'` WHERE not deleted

**Servers:**
- New Port server (read-only + Signal for feedback)
- Modified ComputeInstance server: auto-create Ports on Create (one per
  network_attachment, in same transaction), auto-delete on Delete (detach
  ExternalIPs first), populate output-only fields on Get/List
- Modified ExternalIPAttachment server: validate `port` target (exists, READY
  state, not already consumed), resolve Port's owner for any owner-specific
  validation

**Reconciler:**
- New Port reconciler following existing patterns (create Port CR on hub,
  handle feedback)

**CLI:**
- New `port list --owner <id>` and `port describe <id>` commands
- Modified ExternalIPAttachment commands to accept `--port` flag

#### OSAC Operator

**CRD:**
- New Port CRD in `api/v1alpha1/` with spec (subnetRef) and status (state,
  ipAddress, ownerType, ownerRef)

**Controllers:**
- New Port controller + feedback controller
- Modified ComputeInstance controller: create Port CRs with ownerReferences
  when creating ComputeInstance CRs
- Modified ExternalIPAttachment controller: support `port` as target type,
  resolve Port to determine owner and dispatch provisioning accordingly

#### OSAC AAP (Ansible)

**Playbooks:**
- New `playbook_osac_create_port.yml` / `playbook_osac_delete_port.yml` (may be
  no-ops initially if Port provisioning requires no infrastructure action beyond
  what Subnet creation provides)
- Modified attach/detach ExternalIP playbooks: accept Port reference, resolve
  Port CR to find owner type and details

**Roles:**
- Modified `metallb_l2` attach/detach tasks: when payload contains a Port
  reference, look up the Port CR to find the owner type and details, then proceed
  with existing provisioning logic (e.g., MetalLB LoadBalancer Service for
  ComputeInstance owners)

#### Implementation Order

| Phase | Repo | Work | Breaking? |
|-------|------|------|-----------|
| 1 | fulfillment-service | Port proto + DB migration | No |
| 2 | osac-operator | Port CRD type definition | No |
| 3 | fulfillment-service | ComputeInstance auto-creates Ports, output-only fields | No |
| 4 | osac-operator | Port controller + feedback controller | No |
| 5 | fulfillment-service | Add `port` to ExternalIPAttachment `oneof target` | No |
| 6 | osac-operator | ExternalIPAttachment controller supports port target | No |
| 7 | osac-aap | Attach/detach playbooks support port resolution | No |
| 8 | All repos | Deprecate `compute_instance` target, migrate to `port` | Yes |

Phases 1--7 are fully additive. Phase 8 is the only breaking change and can be
deferred.

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Migration complexity | Existing ExternalIPAttachments reference `compute_instance`, not `port` | Keep `compute_instance` target working during migration; provide tooling to migrate existing attachments to `port` targets |
| Port creation failure | ComputeInstance creation fails if Ports cannot be created | Create Ports in the same database transaction as ComputeInstance; rollback atomically on failure |
| Orphaned Ports | Ports could outlive their parent if ownerReferences are misconfigured | Use Kubernetes ownerReferences for CRs; database-level cascade or explicit cleanup in Fulfillment Service |
| Additional API complexity | New resource type adds cognitive load for tenants | Ports are auto-managed; tenants only interact with them when listing (to choose a Port for ExternalIP attachment) |

### Drawbacks

Introducing Port adds a new resource type to OSAC's data model. For single-NIC
VMs (the common case today), Port is an extra layer of indirection that provides
no immediate benefit---the tenant must now specify a Port ID instead of a
ComputeInstance ID when attaching an ExternalIP.

However, this cost is offset by:
- Multi-NIC support requires *some* disambiguation mechanism; Port provides a
  clean, well-understood one
- Port is auto-managed, so the operational burden is minimal
- The pattern is established across cloud providers (AWS ENI, OpenStack Port)

## Alternatives (Not Implemented)

### Alternative 1: Polymorphic Reference (target_type + target_id)

Replace the `oneof target` with generic `target_type` and `target_id` string
fields. Each resource type (ComputeInstance, NATGateway, Cluster) is referenced
by type and ID.

**Why not selected:** Does not solve the multi-NIC disambiguation problem. A
polymorphic reference to a ComputeInstance still does not specify which network
interface should receive the ExternalIP. Additionally, type safety is weaker than
a proto `oneof`, and validation logic must switch on string values.

### Alternative 2: Primary Network Attachment Flag (without Port)

The unified-networking enhancement (OSAC-1029) proposes a `primary` boolean
field on network attachments. Exactly one attachment is marked `primary: true`,
and all ExternalIP traffic (DNAT) is routed to the primary subnet's IP address.

The `primary` flag serves three purposes in unified-networking:

1. **Default gateway** --- the primary subnet provides the default route;
   secondary subnets receive only connected routes (no gateway)
2. **DNAT target for ExternalIP** --- ExternalIP traffic is routed to the
   primary subnet's IP
3. **SNAT source for NATGateway** --- outbound NAT uses the primary subnet's
   IP as the source

**Why not sufficient on its own for ExternalIP disambiguation:** The `primary`
flag is the right mechanism for default gateway and SNAT source selection---a
resource needs exactly one default route and one SNAT identity. However, for
ExternalIP attachment it limits each multi-NIC resource to a single target. A VM
with NICs on both a frontend and backend subnet can only receive external
traffic on the primary NIC. If a tenant needs ExternalIPs on *different* NICs
of the same VM (e.g., a public-facing IP on the frontend NIC and a management
IP on the backend NIC), the `primary` field cannot express this.

**Port and `primary` are complementary, not mutually exclusive.** The `primary`
flag remains necessary for default gateway and SNAT source selection. Port adds
per-interface granularity for ExternalIP attachment: each network attachment
gets its own Port with a stable ID, and ExternalIPAttachment can target any
Port independently. This aligns with cloud provider models where multiple
Elastic IPs can be attached to different ENIs on the same instance.

### Alternative 3: No Port --- Direct References per Resource Type

Keep the existing model: each resource type directly references ExternalIP in
its own way. ComputeInstance uses `compute_instance` target,
NATGateway has `spec.externalIP`, Cluster uses `cluster` + `target_endpoint`.

For multi-NIC disambiguation, add a `network_attachment_index` field to
ExternalIPAttachment when the target is a ComputeInstance.

**Why not selected:** Index-based references are fragile---if the
`network_attachments` array were ever reordered (even though it's currently
immutable), references would break silently. A Port with a stable ID is more
robust. Additionally, the index approach does not generalize to BareMetalInstance
or future resource types that may have multiple network interfaces.

### Alternative 3: Port as a Universal Target for All Resource Types

Make Port the *only* target in ExternalIPAttachment, including for NATGateway and
Cluster. Every resource that needs an ExternalIP would create Ports.

**Why not selected:** NATGateway has a single external-facing connection; a Port
adds no disambiguation value. Cloud providers (AWS, OpenStack) similarly do not
use Ports/ENIs for NAT gateways in their user-facing APIs.

For Cluster, an alternative approach would have the Cluster controller
auto-create Ports for its API and ingress endpoints (e.g., `my-cluster-api`,
`my-cluster-ingress`), replacing the `target_endpoint` enum with a uniform
Port-based model. However, a Cluster's endpoints are not network interfaces on
a subnet---they are opaque service endpoints whose backing infrastructure
varies (LoadBalancer Service, VIP, Route, etc.) and may live inside the managed
cluster rather than on the hub. The question of "who owns the Port" has no
clean answer: the Cluster resource is a high-level abstraction, and the actual
network interface is an internal implementation detail. The `target_endpoint`
enum is the right level of abstraction for a small fixed set of opaque
endpoints, while Port is the right abstraction for concrete network interfaces
on subnets.

## Open Questions

1. **BareMetalInstance:** Does BareMetalInstance have the same multi-NIC
   requirement as ComputeInstance? If so, it should also auto-create Ports. The
   BareMetalInstance API should be reviewed to confirm.

2. **IP allocation timing:** Should the Port's IP address be assigned at Port
   creation time (eagerly) or deferred until the VM is actually running? Eager
   allocation allows tenants to see the IP before the VM boots; deferred
   allocation avoids holding IPs for VMs that fail to start.

3. **Port visibility in CLI:** Should `port list` be a standalone command, or
   should Port information only be visible via `computeinstance describe`? A
   standalone command is more flexible (e.g., can filter by subnet), but may
   expose an abstraction tenants don't need to think about.

4. **Future: Cluster Ports.** Extending Ports to Cluster to replace
   `target_endpoint` was considered, but a Cluster's API and ingress endpoints
   are opaque service endpoints, not concrete network interfaces on subnets.
   The Port abstraction (a NIC on a subnet) does not map naturally---the
   backing infrastructure varies (LoadBalancer, VIP, Route) and may live inside
   the managed cluster, making "who owns the Port" unclear. If Cluster
   networking evolves to expose concrete, addressable network interfaces, Ports
   can be extended at that time. Until then, `target_endpoint` remains the
   right abstraction for Cluster's fixed set of well-known endpoints.

## Test Plan

*Section to be completed when targeted at a release.*

- **Unit:** Port proto validation, ComputeInstance server Port auto-creation
  logic, ExternalIPAttachment validation for port target
- **Integration:** Full lifecycle: create ComputeInstance with 2 NICs, verify 2
  Ports created, attach ExternalIP to specific Port, verify provisioning targets
  correct NIC, delete ComputeInstance, verify Ports cleaned up
- **E2E:** Multi-NIC ExternalIP attachment workflow end-to-end across
  fulfillment-service, osac-operator, and osac-aap
- **Backward compatibility:** Existing `compute_instance` target continues to
  work during migration period

## Graduation Criteria

*Section to be completed when targeted at a release.*

- **Dev Preview:** Port CRD and proto defined; auto-creation from
  ComputeInstance; ExternalIPAttachment supports `port` target
- **Tech Preview:** Full lifecycle tested; CLI commands available; migration
  tooling from `compute_instance` target to `port` target
- **GA:** `compute_instance` target deprecated; all ExternalIPAttachments use
  `port` for ComputeInstance/BareMetalInstance targets

## Upgrade / Downgrade Strategy

- **Upgrade:** Existing ExternalIPAttachments targeting `compute_instance`
  continue to work. New Port records are auto-created for existing
  ComputeInstances via a one-time migration job. Tenants can gradually switch to
  `port` targets.
- **Downgrade:** If downgrading before migration is complete, the
  `compute_instance` target remains functional. Port CRs and database records can
  be cleaned up if the feature is rolled back.

## Version Skew Strategy

- Fulfillment Service must handle both `compute_instance` and `port` targets in
  ExternalIPAttachment during the migration period.
- osac-operator must handle ExternalIPAttachment CRs with either target type.
- osac-aap playbooks must handle both old-style (ComputeInstance reference) and
  new-style (Port reference) payloads.

## Support Procedures

- **Port stuck in Pending:** Check if the Subnet referenced by the Port exists
  and is in READY state. Verify the Port reconciler is running and the hub
  cluster is reachable.
- **ExternalIPAttachment fails with port target:** Verify the Port exists, is in
  READY state, and is not already consumed by another ExternalIPAttachment.
  Check that the Port's owner (ComputeInstance) is in RUNNING state.
- **Orphaned Ports after ComputeInstance deletion:** Check ownerReferences on
  Port CRs. If missing, manually delete the Port CRs and database records.

## Infrastructure Needed

No additional infrastructure is required. Implementation uses existing OSAC
components (fulfillment-service, osac-operator, osac-aap) and existing Kubernetes
primitives (CRDs, ownerReferences).
