# Fork Governance

This repository is a **soft fork** of [`evelandhq/eveland`](https://github.com/evelandhq/eveland). As of 2026-09-10 (tag `upstream-sync-2026-09-10`), the fork has been designated **self-maintained** for the purposes of the De-ASI-INTERFACE agent runtime.

## What "self-maintained" means here

- `main` in this fork is authoritative for the De-ASI-INTERFACE deployment. It is not a mirror of upstream.
- Upstream syncs are opt-in and land as reviewed pull requests, not as fast-forwards. The upstream remote is retained (`upstream = evelandhq/eveland`) as a source, not as a truth.
- Divergence is measured against the tag `upstream-sync-2026-09-10`. Any subsequent upstream sync creates a new tag of the form `upstream-sync-YYYY-MM-DD` for the same purpose.
- The maintenance obligation this implies (CVE response, framework upgrades, systemd/sandbox toolchain drift) is acknowledged and is tracked in the compliance calendar in [De-ASI-INTERFACE/policies/COMPLIANCE_CALENDAR.md](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/COMPLIANCE_CALENDAR.md).

## Change control

Repository controls on this fork are aligned with [POLICY-011 Change Management and Segregation of Duties](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/11_CHANGE_MANAGEMENT_POLICY.md):

- `main` is branch-protected: linear history, no force-push, no deletion, CODEOWNERS review required, signed commits required, conversation resolution required.
- Administrator bypass is permitted for the sole principal, is logged by GitHub, and any use of bypass is recorded in the sole-operator exception log referenced in POLICY-011 §4.
- Merging an upstream sync PR is a CM-3 or CM-4 change depending on scope; the reviewer must consult POLICY-011 §3 before approving.

## Trust boundaries relevant to agents

The Cl-eveland platform hosts AI agents whose governance is defined in [POLICY-010 AI Agent Governance](https://github.com/De-ASI-INTERFACE/De-ASI-INTERFACE/blob/main/policies/10_AI_AGENT_GOVERNANCE_POLICY.md). Two invariants that this repository must uphold and that reviewers must protect on every merge:

1. **No agent runtime holds signing authority over any Tier 1–3 reserve account.** Enforced at the wallet layer, not at the code layer; but any change here that would broaden agent signing scope must be reviewed against POLICY-009 §5.
2. **Kill-switch responsiveness and decision-audit-log continuity are release gates.** A release that regresses either does not proceed to production. The gate implementations are tracked as open compliance-calendar items O3 and O4 and will land in follow-up PRs.

## Contact

Governance questions: raise a GitHub issue with the label `governance` and tag the repository owner.
