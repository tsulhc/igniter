# PR #350 — review of author replies (2026-09-23)

Reference: https://github.com/pokt-network/igniter/pull/350
Reviewed head: `ca45a24ed24dfb374cb65dabb6454c5361236406`
Scope: technical review of `tsulhc`'s three inline replies and final PR comment, **not** an additional GitHub PR review or evidence of a completed CI run.

## 1. Temporal replay compatibility

Author reply: https://github.com/pokt-network/igniter/pull/350#discussion_r4086440185

Verified: `ExecutePendingTransactions.ts` on PR head is byte-identical to upstream `staging` (`216cd08a473aff0ea1046713016500e606e27f95`). The dispatcher still uses `t.id`. `listPending()` now returns `{ id: number }[]`, so both previously recorded full-row activity results and newly projected results have the property the unchanged workflow consumes. The response correctly addresses the reviewer's replay concern. A Temporal replay regression test would add assurance; the new DAL unit test does not exercise history replay.

## 2. Pending filter regression

Author reply: https://github.com/pokt-network/igniter/pull/350#discussion_r4086442899

Verified: the new test captures the real Drizzle SQL condition passed to `.where()`, renders it with `PgDialect` and checks for a `status` predicate and `pending` parameter. This addresses the prior mock's failure to test `.where()`. Optional hardening: assert the full expected parameter array when the query is intentionally limited to a single parameter.

## 3. Projection location and verifier consistency

Author reply: https://github.com/pokt-network/igniter/pull/350#discussion_r4086453989

Verified: the dispatcher now projects `id` in the DAL. The verifier's `listPendingWithHash()` instead selects whole rows at the DAL layer and subsequently maps them in its activity to `{ id, executionHeight }`. The verifier's activity-result projection avoids carrying the full row into that particular Temporal result, but performs an unnecessary sensitive-row read. Keeping the verifier cleanup separate preserves PR scope. Track it as follow-up.

## 4. Final summary and credential exposure

Author comment: https://github.com/pokt-network/igniter/pull/350#issuecomment-5801633063

The final comment accurately summarizes the dispatcher fix and acknowledges the separate `getTransaction` concern. Prefer “sensitive transaction params” to “decrypted transaction params”: in the inspected application code, `params` is a plain `text` column and the DAL inserts the JSON string directly. Do not assume encryption/decryption at this column boundary without evidence of an additional infrastructure mechanism.

The child `ExecuteTransaction` workflow only uses `status` and `hash` from `getTransaction`, but its activity currently returns the full row. There are three syntactic call sites in the child; any single normal execution path calls `getTransaction` at most twice. Narrowing this activity result deserves a separate fix and a replay-compatibility assessment before changing the child workflow's activity contract.

## Security follow-up for maintainers

- Narrow `getTransaction`'s **workflow-facing** activity return to `status`/`hash` (and retain a separate DAL full-row accessor for signing and verifier operations requiring additional fields). Ensure the change is compatible with already-recorded Temporal histories.
- Project `id` and `executionHeight` at the DAL boundary in `listPendingWithHash`.
- Audit storage and query-log redaction for sensitive transaction parameters, including `params`; handle any findings through the project's appropriate security remediation process.
- Consider whether legacy Temporal histories that contain full rows need access restrictions, retention review, or other remediation.

## Review state at time of inspection

The three original review threads are marked outdated but remain unresolved. Jorge's review remains `CHANGES_REQUESTED`. After confirming CI and any replay coverage, request re-review; neither these comments nor this document constitutes approval.

This document is stored in a separate documentation branch in `tsulhc/igniter`. It does not modify the upstream PR, change its review state, or introduce changes to its feature branch.
