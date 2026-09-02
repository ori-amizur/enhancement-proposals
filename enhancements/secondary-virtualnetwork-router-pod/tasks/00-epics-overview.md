# Epic Breakdown — Secondary VirtualNetwork: Router Pod Model with Bare-Metal Connectivity

No Jira Feature exists yet for this work (tracking-link: TBD in the PRD/design). These artifacts are plain files, not synced to Jira — file them manually if/when a Feature key is created.

## Requirement IDs

Requirement IDs (FR-N) are assigned sequentially from the PRD's Acceptance Criteria (Section 3) and In-Scope items (Section 2.1) not otherwise covered by an Acceptance Criterion, in PRD order. See `coverage-matrix.md` for the full requirement text.

## Epics

| # | Epic | T-Shirt Size | Phase | Requirements | Dependencies |
|---|------|--------------|-------|---------------|--------------|
| 1 | [Secondary VirtualNetwork Foundation](epic-1-foundation.md) | L | 1 | FR-1–FR-6, FR-14, FR-15, FR-21, FR-22 | None |
| 2 | [Live Subnet Attach/Detach](epic-2-live-attach.md) | M | 1 | FR-9–FR-13 | Epic 1 |
| 3 | [NAT Gateway via EgressIP](epic-3-nat-gateway.md) | S | 1 | FR-7, FR-8 | Epic 1 |
| 4 | [Fabric Manager Detection & EVPN Transit Interface](epic-4-transit-interface.md) | M | 2 | FR-16 | Epic 1 |
| 5 | [Bidirectional Bare-Metal Route Dispatch](epic-5-bare-metal-routing.md) | L | 2 | FR-17–FR-20 | Epic 4 |

No epic sizes to XXL — no splits required beyond the two-epic split already applied within each phase (foundation vs. live-attach optimization; transit-interface provisioning vs. route dispatch).

**Feature-level sizing check:** skipped — no Jira Feature exists to read a Size/Story-Points field from. For reference against the sizing rubric, the five epics sum to roughly 2×L + 2×M + 1×S, which is consistent with a Feature-level size of L–XL if this were tracked as one Feature, or two M–L Features if tracked as one per phase.

## Dependency Order

```text
Epic 1 (Foundation)
  ├── Epic 2 (Live Attach/Detach)       — optimizes Epic 1's subnet lifecycle, no other epic depends on it
  ├── Epic 3 (NAT Gateway)              — independent of Epic 2, can run in parallel with it
  └── Epic 4 (Transit Interface)        — independent of Epics 2/3, can run in parallel with them
        └── Epic 5 (Bare-Metal Routing) — requires Epic 4's transit interface to exist
```

Recommended order: **Epic 1 first** (nothing else can start without the router pod existing). Once Epic 1 is done, **Epics 2, 3, and 4 can proceed in parallel** — they touch different parts of the router pod (agent/attach logic, NAT/EgressIP, transit interface) and don't share implementation surface. **Epic 5 requires Epic 4** to be complete (it dispatches routes over the transit interface Epic 4 provisions).

Phase 1 (Epics 1–3) is independently shippable and delivers complete value with no fabric manager involved. Phase 2 (Epics 4–5) is additive and requires Phase 1 (specifically Epic 1) to already be in place; it does not require Epics 2 or 3.

## Story Implementation Notes

- **Epic 1**: stories 1.01–1.05 are sequential (each builds directly on the previous one's resources); 1.06 (QE) and 1.07 (DOCS) follow once 1.01–1.05 are merged.
- **Epic 2**: 2.01 (agent) can start in parallel with 2.02 (live-attach patch logic) since they're different code paths that converge in 2.03 (fallback trigger, which needs both). 2.04/2.05 follow.
- **Epic 3**: fully sequential, small epic.
- **Epic 4**: 4.01 (NetworkClass detection) must land before 4.02 (conditional transit interface provisioning); 4.03 (QE) follows.
- **Epic 5**: 5.01 and 5.02 can proceed in parallel (they're the two directions of the same route-dispatch mechanism); 5.03 (reconciliation consistency) depends on both; 5.04/5.05 follow.
