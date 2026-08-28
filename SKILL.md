---
name: audit-code
description: Audit code for concrete correctness, security, performance, UX, and maintainability risks within an agreed scope. Use for code audits, PR feedback, or adversarial review. Produce evidence-backed findings, focused verification, and a bounded independent review when permitted.
---

# Audit Code

Find concrete failures in the requested change and its affected paths. Use security, performance,
UX, DX, and edge cases as review lenses, not a requirement to launch five agents or repeat the
entire audit. Prefer a few demonstrated findings over a long list of hypothetical improvements.

## Scope and authority

Before reviewing, establish a compact mission contract:

- Demonstrated failure or requested outcome, acceptance criteria, and affected paths.
- Explicit non-goals and whether the task permits edits or is review/plan only.
- Expected footprint, runtime context, and evidence that requires external access.

Classify each concern:

1. **Mission blocker:** an acceptance criterion is unmet.
2. **Patch regression:** this change introduces a concrete failure.
3. **Mandatory safety:** a concrete security, authorization, privacy, or data-loss risk.
4. **Follow-up:** worthwhile but outside the current mission.
5. **Non-finding:** unsupported, duplicate, stale, or pre-existing without an in-scope consequence.

Only the first three can block the patch. A finding does not itself authorize edits, external
writes, publishing, workflow cancellation, or skill/memory changes. Respect the user's approval
boundary; plan-only work stops at the plan. Explain any necessary scope expansion before acting.

For a hotfix, reassess near five files, 150 non-generated changed lines, twice the expected
footprint, or an unplanned schema, queue, scheduler, protocol, or recovery mechanism. These are
tripwires, not universal size limits. Look for a smaller design before expanding the mission.
Do not demand new restrictions or infrastructure merely to satisfy a speculative scenario.

## Read only the applicable guidance

Use [audit-framework.md](references/audit-framework.md) for the severity rubric, finding schema,
coverage matrix, and report template. Its technical lists are a catalog: select checks for
boundaries present in the mission, rather than loading or applying every domain to every audit.
Keep the workflow here authoritative; reference checklists do not add extra review rounds.

| When the mission touches | Read or search |
| --- | --- |
| Access, provider callbacks, OAuth, data integrity, metrics, shared state | Relevant invariant, role, and edge sections in [audit-framework.md](references/audit-framework.md); search the affected boundary, such as `OAuth`, `no-data`, or `mutation` |
| Queues, claims, retries, locks | [Queue overlay](references/domain-overlays.md#queues-idempotency-and-locks) |
| CI, deployment, artifacts, test harnesses | [CI overlay](references/domain-overlays.md#cicd-test-infrastructure-and-artifact-promotion) |
| SSH bootstrap and worker trust | [SSH overlay](references/domain-overlays.md#ssh-bootstrap-and-remote-worker-trust) |
| macOS packaging, signing, distribution | [Release overlay](references/domain-overlays.md#macos-release-closure) and the framework's macOS module |
| Desktop preview, export, persisted editor state | [Editor overlay](references/domain-overlays.md#swiftuiappkit-preview-export-and-editor-freshness) |
| Imports, parsing, financial reconciliation | [Import overlay](references/domain-overlays.md#parser-import-and-personal-finance-reconciliation) |
| Forms, dashboards, list/detail views | [UI overlay](references/domain-overlays.md#ui-state-persistence-and-detail-loading) |
| Bun and SQLite | The framework's Bun + SQLite module |
| An external reviewer found a miss | [Gap reducer](agents/audit-gap-reducer.md), when a structured miss analysis would help |

An unmatched stack is not a blocker: derive its relevant invariants from the code and state
uncertain runtime assumptions. Companion skills are optional; do not add installation promotions
to every report or require unrelated tools to complete the audit.

## Decision and test proof

When a changed branch depends on a derived value such as `allowed`, `success`, `ready`, or `found`:

1. **Trace its producers.** Enumerate the materially different ways that value can arise:
   explicit choice, default, inherited value, fallback, synthetic promotion, bypass, or cache.
   Trace the consumer too. The same value need not carry the same authority or prove the same work.
2. **Verify the selected scope.** Establish the actor, resource, account/tenant, policy scope,
   precedence, and freshness that actually apply. A setting existing somewhere is not proof that
   it controls this decision. Reuse the canonical resolver; do not invent a parallel policy path.
3. **Try a controlled counterexample.** Keep the final value the same and change its source or
   scope. Ask whether the downstream behavior should remain the same under the actual contract.
   For example, a default allow is not automatically consent, and a successful skipped job is not
   proof that validation ran. Distinct sources need not be treated differently unless the contract
   requires it; this is a proof obligation, not a mandate to add flags or restrictions.
4. **Prove the test path.** Inspect the fixture's effective configuration, identity, scope, and
   prerequisites through the real resolver or an observable boundary. A test name or setup comment
   is not evidence. Mocks must not replace the decision being tested. Include a useful positive
   case so an always-deny, always-prompt, or fallback-only implementation cannot pass unnoticed.

Use the smallest counterexample that can falsify the claim, not a speculative Cartesian test
matrix. A before/after failure must occur for the claimed reason, not a missing fixture dependency.
If a fixture changes scope or reachability, re-establish the evidence; do not carry forward the
old test's proof claim. Distinguish source inspection, focused checks, and actual runtime proof.

## Audit workflow

1. **Map the causal paths.** Read the code and product contract. Identify producers, resolvers,
   consumers, mutations, and affected sibling entry points. Build a compact matrix of critical
   invariants, paths, evidence, and gaps using the framework. Include relevant read/preview paths,
   not only mutating endpoints. A prior missed invariant belongs in this matrix.
2. **Review from each applicable lens.** Check security, performance, UX, DX, and edge cases against
   that matrix. Trace failures through the real execution path, including wrappers, defaults,
   retries, and provider boundaries. Examine shared lifecycle ownership before accepting a local
   compensation helper; do not mock the canonical path away to justify the helper.
3. **Reconcile findings.** Require a trigger, code/runtime evidence, impact, confidence, scope
   disposition, and smallest useful fix. Resolve disagreement through evidence, not reviewer votes.
   An implementation being stricter than a spec is not automatically correct: reconcile the
   product/security requirement and document any intentional difference.
4. **Verify and get independent review.** Run the smallest relevant check when cheap and permitted.
   Use one independent reviewer when delegation is available and allowed; add specialists only for
   a concrete risk or explicit request. Provide the mission, non-goals, raw artifacts, and current
   diff without your desired answer or previous verdict. Ask it to challenge a consequential
   assumption. If delegation is unavailable, perform and disclose a main-thread review instead.
5. **Fix only authorized, admitted findings.** For report-only work, report them without editing.
   After a fix, rerun the affected check and one targeted review of the changed causal path. Broaden
   only when the fix changes the mission or exposes another concrete affected path. Do not restart
   every specialist or rerun unrelated suites after each correction.
6. **Close with evidence.** Do a final main-thread pass against acceptance criteria and current
   feedback. Stop when no admitted in-scope finding remains, or state the exact blocker. Use the
   framework's single report template. Separate verified results, unresolved findings, follow-ups,
   and environment requirements that were not checked. Do not claim completion from consensus,
   a submitted command, an audit alone, or a queued/skipped check.

## Active PRs and external feedback

- Resolve the live PR head before collecting checks and reviews; pass that identity explicitly to
  dependent readers. Collect independent evidence in parallel only after its shared inputs exist.
  A cached summary is not a current-head lookup. Recheck the head before a completion or merge claim.
- Read all pages of conversation comments, submitted review bodies, and inline threads, including
  unresolved/outdated state and late feedback. Review summaries are not substitutes for the original
  human comments. Verify each finding against the current code and record its disposition.
- Match CI evidence to the head, actual run/attempt, and relevant jobs. Inspect failed-job logs before
  assigning cause. A successful wrapper or skipped draft workflow does not prove the code was tested.
  Separate superseded runs, queued jobs, infrastructure failures, and code failures. Inspect an older
  blocking run before any authorized cancellation; do not cancel work just to make a status green.
- Reply to or resolve only findings actually addressed, and only when authorized. Do not dismiss an
  unresolved legitimate review to enable merging. If the head changes, revisit the affected evidence.
- When an external finding exposes a miss, identify the missed invariant, evidence, assumption, and
  minimal counterexample. Check whether existing guidance was absent, ambiguous, or simply not
  followed. Propose the smallest reusable update; do not append the incident as another universal
  rule or edit skills/memory without authorization.

## Verification and reporting discipline

Require concrete evidence for findings and distinguish confirmed defects from uncertain risks.
For admitted High/Critical findings, include a focused regression check or explain the verification
blocker. Do not request broad testing merely because it is available. Ensure code, tests, docs, and
claims agree with the actual contract, including the unchanged positive path.

Keep reports useful to the operator: findings first, no repeated verdicts or empty boilerplate.
Retain a small findings-to-fixes ledger during a multi-step task so later work can resume without
inventing which checks passed or which approvals were granted. Audit harmful UX as a risk, not a
recommended tactic; do not turn a review into operational abuse instructions.
