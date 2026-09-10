# RFC: Emergency kill switch

**Status:** Draft
**Owner:** De-ASI-INTERFACE
**Filed:** 2026-09-10
**Policy anchors:** [POLICY-008 §3](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/08_INCIDENT_RESPONSE_POLICY.md), [POLICY-003 §5](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/03_TRADING_RISK_MANAGEMENT_POLICY.md), [POLICY-010](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/10_AI_AGENT_GOVERNANCE_POLICY.md)

## Context

POLICY-003 §5 commits to a hardware kill switch responsive within 3 seconds; POLICY-008 §3 requires containment "immediately." Today the fork ships neither the switch nor a measured SLO. This RFC proposes a design that respects Cl-eveland's existing product boundaries and dependency arrows.

## Non-goals

- Not a global platform kill (that is `systemctl stop`). This is a scoped kill: per-agent, per-deployment, or platform-wide when explicitly requested.
- Not a UI feature in this RFC. The UI belongs in a follow-up once the contract is proven.

## Placement (against the architecture)

The kill switch must live at the Agent Gateway (host process, `DynamicUser`), because per `docs/en/reference/architecture.md` §1 the Gateway owns "trusted routing" for the public agent data plane. Anything downstream is too late — the sandboxed Worker cannot fail-closed a decision that has already been signed off-chain.

## Contract

Three surfaces:

1. A `killswitch` table in Postgres, owned by `packages/db` (per the dependency arrows, `packages/db` is a root package with no internal deps upward — correct level). Keys: `scope` (`platform` | `project:<slug>` | `deployment:<key>` | `agent:<id>`), `enabled`, `reason`, `set_at`, `set_by`, `expires_at`.
2. A Gateway middleware that consults an in-memory copy (refreshed every 500ms) and, when a matching row is enabled, returns `503 kill_switch_active` with the reason. Fail-closed: if the refresh cannot reach the DB for more than 5 seconds, the middleware treats the last known state as authoritative. Never fail-open.
3. A `ctl` subcommand (`packages/ctl`) — `ctl killswitch set|clear|status` — with mandatory `--scope`, `--reason`, and (for `set`) `--expires`. Emits an audit event in the `platform` telemetry domain.

## SLO

Time-to-block = time from `ctl killswitch set` returning success to first blocked request at the Gateway.

Budget: ≤ 2 seconds P99, ≤ 3 seconds hard cap. Measured by a nightly smoke test (`security-killswitch-smoke.yml` — separate PR):

1. `ctl killswitch set --scope deployment:<test-key> --reason smoke --expires 60s`
2. Loop hitting the deployment; record time-to-first-503.
3. Fail if greater than 3s or if 200s continue after 3s.

## Interaction with existing contracts

- SessionBinding (`routing.md` §3): a kill switch does not tear down active bindings; it blocks new turns on affected scopes. Active turns complete or fail per their own timeout. Rationale: cutting an in-flight signed transaction mid-flight can be worse than letting it settle for reconciliation.
- Weighted routing (`routing.md` §2): a per-deployment kill effectively drops that target's weight to 0 without changing the route pointer. The route pointer is preserved so a clear+re-enable does not race with a promotion.
- Scheduling (`scheduling.md`): a kill affecting a scoped deployment causes its next `dispatching` transitions to fail with `kill_switch_active`; `missedTicks` continues to increment; on clear, coalesced execution proceeds per the existing planner semantics. This must be tested; missed-tick coalescing during a kill window is the most likely correctness bug.
- Observability (`observability.md`): every set/clear emits a `platform` domain event (not `agent`, because the actor is the operator, not the agent). Payload: scope, reason, actor, before/after state.

## Signing boundary

The kill switch is a Gateway-side block. It does not, cannot, and must not attempt to revoke keys or reach into the sandboxed Worker to unwind a decision. Key revocation is a separate operator action (POLICY-008 Phase 2 Containment) and belongs in the wallet layer, not here.

## Open questions

- Refresh cadence 500ms — too aggressive for the DB, too slow to hit a 2s P99? Alternative: LISTEN/NOTIFY on Postgres for push semantics.
- Should `platform` scope require two-person confirmation on `ctl` (per POLICY-010 §5 for A4)? Argument for: platform kill is the biggest possible blast radius. Argument against: it defeats the "immediate" mandate. Proposed compromise: single-person set with a mandatory `--i-understand-blast-radius` flag; two-person required for `--expires > 1h`.
- Interaction with `install-smoke` and `systemd-smoke` — the smoke test needs its own ephemeral DB or a well-isolated scope so it does not risk touching real state.

## Definition of done

- Design approved via this RFC.
- Implementation PR (feature branch, CM-3 per POLICY-011).
- `security-killswitch-smoke.yml` runs green three nights in a row.
- `GOVERNANCE.md` and `De-ASI-INTERFACE/policies/inventory/agents.yaml` updated with the kill-switch identifier per agent.
- Compliance calendar item O3 closed.
