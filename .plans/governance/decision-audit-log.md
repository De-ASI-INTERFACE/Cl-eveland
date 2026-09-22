# RFC: Decision audit log

**Status:** Draft
**Owner:** De-ASI-INTERFACE
**Filed:** 2026-09-10
**Policy anchors:** [POLICY-010 §8](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/10_AI_AGENT_GOVERNANCE_POLICY.md), [POLICY-008 §3](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/08_INCIDENT_RESPONSE_POLICY.md), [POLICY-009 §5](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/09_RESERVE_CUSTODY_POLICY.md)
**External anchors:** [Federal Reserve SR 11-7](https://www.federalreserve.gov/supervisionreg/srletters/sr1107a1.pdf), [NIST AI RMF 1.0](https://www.nist.gov/itl/ai-risk-management-framework)

## Context

POLICY-010 §8 requires an append-only, hash-chained audit record for every in-scope agent decision, sufficient for post-incident reconstruction. Today the platform emits `agent` domain telemetry via OTLP, which is streaming and lossy by design. This RFC proposes a decision-audit-log layer that sits alongside — not inside — OTLP.

## Non-goals

- Not a replacement for OTLP `agent` telemetry. That stream is designed for cost-effective observability; the audit log is designed for evidentiary reconstruction. Different SLAs, different retention, different query patterns.
- Not a blockchain. Hash-chained append-only in Postgres with periodic anchoring is sufficient and does not require on-chain overhead per decision.

## Placement (against the architecture)

The record must be produced at the point of decision by the injected Eve hook's private providers (owner of the `agent` telemetry domain per `docs/en/reference/observability.md`) and flushed through a dedicated collector pipeline that writes to a `decision_audit` table owned by `packages/session-collector`. This respects the arrow `packages/session-collector -> packages/core + packages/db` from `architecture.md` §2.

## Record schema (proposed)

Every record contains:

- Monotonic id, ISO-8601 UTC timestamp.
- Agent identifier (from POLICY-010 inventory), deployment key, session id, turn id.
- Model provider, family, and pinned version identifier. Pinning MUST be stable (SR 11-7 §V); model endpoints without a stable version pin are ineligible for authority tier A3+.
- SHA-256 of the frozen system prompt (matches `agents.yaml`).
- SHA-256 of the tool set snapshot (matches `agents.yaml` including build-visible environment).
- SHA-256 of the input; PII lives outside the log.
- Ordered list of tool calls, each with SHA-256 of args and response and per-call timestamp.
- RPC dependency list (Solana / EVM / exchange), canonical endpoint id (not URL), method, response slot, timestamp.
- Decision: kind (`propose` | `broadcast` | `no_action`) and SHA-256 of payload. Full payload lives in an object store.
- Human approval reference if applicable.
- Resulting transaction signature if any.
- Authority tier (A0-A4).
- `prior_record_sha256` — hash-chain link to the previous record for the same deployment.
- `self_sha256` — deterministic hash of this record excluding self_sha256.

Full payloads reference blobs stored in an S3-compatible bucket with object-lock in compliance mode. That is the tamper-evidence guarantee; Postgres is the index and the chain.

## Chain semantics

- On insert, compute `prior_record_sha256` from the last row for `(deployment_key)`. Serialization on `(deployment_key)` is enforced by an advisory lock or a per-deployment queue at the collector.
- Compute `self_sha256` deterministically. Reject inserts where the client-supplied hash does not match server recomputation.
- Every 15 minutes, a scheduled job (per `scheduling.md` §1 cron discovery) computes a Merkle root of the last window across all deployments and writes it to a `chain_anchor` table with a wall-clock timestamp. On-chain anchoring is a later enhancement.

## Retention

- Postgres index rows: 2 years.
- Object-lock blobs: 7 years (aligned with SAR retention in POLICY-007 §6, since audit records will be the primary evidence for any suspicious activity determination).
- Anchors: indefinite.

## Interaction with existing contracts

- Telemetry domains (`observability.md`): the audit log is not a fifth domain; it is a sibling data plane consumed by the same Collector but written to a different destination and never fanned out to user-configured backends. The Collector already has the trust boundary needed to enforce this.
- Agent environment (`agent-environment.md` §1): the hash of the agent's build-visible variables set is included in the record (as part of `tool_set_sha256`), because a variable change is a behavior change and must be reconstructable.
- Session and turn boundaries: the record is emitted on `turn.completed` and `turn.failed` (per `scheduling.md` §3 dispatch settlement). One record per decision, not per tool call. Tool calls are nested.

## Query pattern

The primary query is "given an on-chain transaction signature, produce the full decision record and its chain proof." That is a single indexed lookup on `resulting_tx_signature` followed by a walk of `prior_record_sha256` back to the nearest anchor. Optimized for reconstruction, not for analytics — analytics stays in OTLP.

## Open questions

- Object-lock backend choice (S3 with Object Lock in compliance mode is the reference; MinIO with equivalent semantics is the self-hosted option). Which is acceptable under POLICY-009 §6 audit-trail language?
- Serialization: advisory lock vs. per-deployment queue at the collector. Advisory lock is simpler but degrades under load; per-deployment queue matches the SessionBinding model already in `routing.md` §3.
- PII handling: input and args are hashed by default; a policy switch may allow storing plaintext for specific tool names (e.g., `solana.sendTransaction`) where the plaintext is on-chain anyway. This needs explicit review under POLICY-004.

## Definition of done

- Design approved via this RFC.
- Implementation PR (feature branch, CM-3).
- Verifier CLI (`ctl audit verify --deployment <key>`) that walks the chain and reports break points.
- Recovery drill: reconstruct a chosen decision from tx signature alone, in <= 10 minutes, without operator memory.
- Compliance calendar item O4 closed.
