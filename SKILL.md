---
name: audit-code
description: Run a two-pass, multidisciplinary code audit led by a tie-breaker lead, combining security, performance, UX, DX, and edge-case analysis into one prioritized report with concrete fixes. When sub-agents are available and permitted, iterate fresh-eyes audit passes until no meaningful findings remain. Use when the user asks to audit code, perform a deep review, stress-test a codebase, or produce a risk-ranked remediation plan across backend, frontend, APIs, infra scripts, and product flows. Keep the audit stack-agnostic first, then add runtime-specific checks as an overlay.
---

# Audit Code

## Overview

Run an expert-panel audit with strict sequencing and one unified output document.
Produce findings first, sorted by severity, with file references, exploit/perf/flow impact, and actionable fixes.
For local desktop utilities, treat privileged OS integration, local privacy leakage, same-user abuse, and release-toolchain trust as first-class risk areas rather than defaulting to server-style findings.
For privacy-sensitive desktop flows, treat the global pasteboard as a shared side channel, not a harmless transport layer; audit clipboard managers, same-user listeners, and restoration races whenever the app touches it.
For CI/CD, release, and infrastructure automation, treat external records and side effects (deployments, statuses, tags, releases, caches, environments, cloud objects) as durable state; audit creation, cancellation, cleanup, and provenance matching end-to-end.
When the product edits OS-owned or user-owned config artifacts such as launchd plists, crontabs, `.env` files, or other flat files, explicitly audit:
- identifier-to-path derivation and traversal resistance
- hidden-artifact creation via dot-prefixed or otherwise scanner-skipped names
- read-only capability parity between UI and lower layers
- split-brain state between persisted config and runtime override systems
- malformed-config partial-failure behavior during scans
- CLI probe failure handling for runtime-state reads
- stale-index versus fingerprint-based conflict safety for line-oriented edits

Load `references/audit-framework.md` before starting the analysis.
When a third-party reviewer, bot, human reviewer, or later fresh-eyes pass finds issues after this skill already ran, treat that as audit feedback to improve shift-left coverage. Reconcile the feedback, extract the missed invariant, and update the relevant audit checklist or prompt guidance when the lesson is reusable.

## Required Inputs

Collect or infer the following:
- Audit scope: paths, modules, PR diff, or whole repository.
- Product context: PRD/spec/user stories, trust boundaries, and critical business flows.
- Runtime context: deployment model, queue/cron/background jobs, traffic profile, data sensitivity, and abuse assumptions.
- Constraints: timeline, acceptable risk, and preferred remediation style.

If product context is missing, state assumptions explicitly and continue.

## Team Roles

Use exactly these roles:
- Security expert
- Performance expert
- UX expert
- DX expert
- Edge case master
- Tie-breaker team lead

The tie-breaker lead resolves conflicts, prioritizes issues, and produces the final single report.

## Workflow

Follow this sequence every time:

1. Build Context
Read code + product flows. Identify assets, entry points, high-risk operations, privileged actions, external dependencies, and "failure hurts" journeys.
For products that replace native OS behavior, explicitly map the prerequisites to intercept the native action versus the prerequisites to complete the replacement flow.

2. Build Invariant Coverage Matrix
Before specialist pass 1, map critical invariants to every mutating path (HTTP routes, webhooks, async jobs, scripts):
- Data-integrity invariants: linked records, transaction boundaries, and conflict handling must preserve consistency.
- Access lifecycle invariants: permission changes (disable/revoke/role change) must take effect across active credentials and privileged actions.
- Entitlement invariants: plan/tier/feature gates must be enforced on every trigger path (API/UI/webhook/job), and queued work must re-check entitlement at execution time.
- Input/protocol invariants: validation, canonicalization, parser behavior, and payload size/media-type policy must be consistent across equivalent paths.
- Sentinel semantics invariants: special values (for example `0`, empty, `NULL`) must have one canonical meaning across UI/API/webhook/worker paths.
- State-transition invariants: lifecycle transitions (active/archived/deleted/expired) must be explicit, legal, and consistently enforced.
- Cross-trigger policy invariants: business rules (for example downgrade timing, reset authority, pause/resume criteria) must remain consistent across user actions, provider callbacks, and background workers.
- Mutation outcome invariants: state-changing handlers must only signal success (UX/audit/events) after durable write success; persistence failures must be surfaced.
- Write-freshness invariants: callback/verification paths must avoid stale full-record rewrites; use conditional field-scoped updates for concurrent edit safety.
- Side-effect ownership invariants: if a semantic link mutates shared record fields (for example tax flags, reconciliation markers, or categorization), persist whether the link owns that mutation and refresh any stored "previous value" snapshot when relinking to a recreated backing record.
- Deferred-attachment invariants: if the product allows creating semantic events before the final external identifier or bank transaction exists, equivalent attach/remap paths must exist across API and UI and must preserve create-path validation and policy checks.
- Idempotency/order invariants: retries, duplicates, and out-of-order events must not corrupt state or duplicate side effects.
- Claim lifecycle invariants: claim/lease-based workers must persist attempts/status and release claim markers on every success and failure path.
- Time-window invariants: timezone and boundary behavior (expiry, rollovers, DST) must be deterministic.
- Resource-boundedness invariants: loops, fan-out, queues, and in-memory maps must have caps/backpressure/cleanup.
- Shared-identity exclusivity invariants: if one external/shared identifier (for example a bank transaction id, provider event id, or import fingerprint) must not back multiple semantic link types, enforce that exclusivity at the datastore layer, not only with application-side prechecks.
- External dependency invariants: timeouts, partial failures, fallback behavior, stale-cache behavior, and explicit provider policy parameters must be intentional.
- Observability invariants: high-risk state changes and failures must emit actionable, traceable signals with required schema fields (actor/target and before/after context where applicable).
- Editable-surface invariants: fields exposed as editable in UI/API must be durably persisted or explicitly documented and enforced as immutable.
- Deployment/automation invariants: deployment docs and release scripts must align with CI artifact strategy, branch policy, and path-specific ingress controls.
- Automation provenance invariants: destructive cleanup of external records must identify ownership with stable provenance evidence and boundary-safe identifiers, not broad branch/SHA/environment/name filters.
- Cancellation/stale-run invariants: external side effects created before skipped, failed, or canceled automation must be prevented up front or cleaned by an independent later path.
- Fast-path/fallback parity invariants: optimized paths and fallback paths must produce behaviorally equivalent artifacts/state with the same compiler/runtime/toolchain, or the intentional divergence must be documented and tested.
- Producer/consumer timing invariants: async producers and consumers must be checked as a DAG with realistic scheduling, queueing, and timeout windows; existence of a produced artifact is not enough if the consumer can start before it is available.
- Retry/attempt identity invariants: reruns, partial reruns, workflow attempts, sharded retries, and manual restarts must resolve the same logical artifact/state when that is intended, and must not silently look for a different attempt-scoped name.
- Schema-derived optionality invariants: config/secret/env validation must classify required versus optional/defaulted keys from the authoritative runtime schema, not only from declaration files or rendered templates.
- Shared test-state invariants: test speedups that reuse databases, caches, workspaces, workers, or ports must prove clean state on process crash, cancellation, retry, and cross-file reuse, not only on the happy-path teardown.
Add domain-specific invariants discovered during context build; do not constrain to this list.
Treat missing parity across equivalent paths as a finding candidate.
Apply any relevant overlay in the Domain-Specific Audit Overlays section below before specialist pass 1; add overlay-specific invariants to the matrix instead of keeping them as separate notes.

3. Pass 1 Specialist Reviews
Run role-specific analysis in this order:
- Security
- Performance
- UX
- DX
- Edge case master
Capture findings using the schema in `references/audit-framework.md`.

4. Tie-Breaker Reconciliation
Resolve disagreements:
- Decide whether contested items are true issues.
- Set severity and confidence.
- Remove duplicates and merge overlapping findings.

5. Cross-Review Pass 2
After edge-case findings, rerun specialists:
- Security/Performance/UX/DX reassess prior findings and new edge-triggered scenarios.
- Edge case master performs a final pass on residual risk after proposed mitigations.

6. Fresh-Eyes Subagent Convergence
When sub-agent tooling is available and permitted by the active instructions/user request, run this loop before the final report:
- Give each available sub-agent only the audit scope, relevant product assumptions, and current tree/diff. Ask for an independent "fresh eyes" audit using this skill's role stack and `references/audit-framework.md`; avoid leaking prior findings, suspected bugs, intended fixes, or conclusions unless needed to define scope.
- Treat a meaningful finding as a validated, actionable, non-duplicate issue with concrete evidence and user/security/performance/operability impact. Ignore style-only preferences, speculative risks without evidence, already-fixed issues, and duplicates.
- Run a fresh-eyes pass with each available sub-agent. If any meaningful findings appear, reconcile them through the tie-breaker lead.
- Fix meaningful findings before continuing when code changes are in scope. If the user explicitly requested report-only/no edits, keep those issues in the report and state that the fix convergence loop was not run.
- After fixes, run the narrowest relevant verification, then rerun the fresh-eyes sub-agent loop. Continue until a complete round across all available sub-agents produces no meaningful findings.
- After the sub-agent loop is clean, rerun the main-thread audit from Build Context through Cross-Review Pass 2. If the main thread finds new meaningful issues, fix them and return to the fresh-eyes sub-agent loop.
- Stop only after both a complete fresh-eyes sub-agent round and the subsequent main-thread rerun produce no meaningful findings, or after documenting an external blocker or task constraint that prevents further convergence.

If sub-agents are unavailable or not permitted, state that constraint and continue with the main-thread audit workflow.

7. External Feedback Reconciliation
When auditing an active PR or change with bot/human reviewer feedback:
- Fetch the latest review threads/comments and evaluate them against the current head, not stale line numbers or prior commit state.
- Classify each item as actionable defect, worthwhile hygiene, false positive, stale/already fixed, or out of scope; only fix or report items with concrete evidence.
- For each actionable item, extract the underlying invariant and add the narrowest regression check that proves the exact failure mode cannot recur.
- Treat permissive fallback predicates in destructive automation as suspect: if the fallback is effectively dead or weaker than the primary provenance check, remove it or document and test why it is safe.
- After fixes, rerun focused verification and re-check whether late review feedback introduced new meaningful findings before finalizing.

When external feedback finds a meaningful issue that prior audit passes missed, also perform a miss analysis:
- Missed invariant: the general rule the audit failed to check.
- Missed evidence: file, runtime behavior, test, log, or reviewer context that should have been inspected.
- Missed role: which role should have caught it and what prompt/checklist wording would have led there.
- Missed verification: the focused test or probe that would have exposed it before review.
- Scope disposition: whether this should update `SKILL.md`, `references/audit-framework.md`, a domain overlay, a repo-local skill, or only the current report.

For reusable lessons, patch this skill or its reference checklist in the same task when allowed. Keep additions invariant-first and stack-agnostic unless the miss is clearly domain-specific.

8. Final Report
Publish one document from the tie-breaker lead with:
- Findings first (ordered by severity, then blast radius, then exploitability).
- Open questions/assumptions.
- Remediation plan with priority, owner type, and verification tests.
- Short executive summary at the end.

## Quality Bar

Enforce these requirements:
- Use concrete evidence with file references and line numbers where available.
- Include reproduction steps for security/performance/edge findings when feasible.
- Prefer actionable fixes over abstract advice.
- Separate confirmed defects from speculative risks.
- Mark confidence for each finding.
- Run a cross-route consistency sweep: equivalent endpoints/jobs must enforce equivalent invariants.
- Run a required runtime-agnostic edge sweep using `references/audit-framework.md` (`Runtime-Agnostic Edge Sweep`).
- Verify deprecation path integrity: explicit failure semantics, replacement guidance, and docs/spec/skill parity.
- For fan-out integration endpoints, verify bounded concurrency and partial-failure behavior expectations.
- Verify state-switch UX integrity for whichever context selector exists in the product (for example workspace/account/tenant/environment): changing it should refresh active views and reset invalid local filters/groupings.
- Verify partial-update invariants against resulting state (`existing + patch`), not only provided fields.
- Verify derived-metric parity: UI formulas and summaries include all policy-required components (for example top-ups, adjustments, and resets), not just base plan values.
- Verify external billing/provider contract explicitness: behavioral requirements (for example proration/cancel timing/status sync) must be set in code/webhook handling, not left to provider defaults.
- Verify pagination/filter carryover safety: user-supplied query params survive page transitions without raw interpolation/encoding drift.
- Verify cross-trigger lifecycle parity: the same business rule is enforced across interactive routes, provider webhooks, and async workers.
- Verify sentinel-value parity: special configuration values (for example `0` limits) have consistent semantics across all interfaces and documentation.
- Verify mutation-outcome integrity: state-changing handlers do not swallow write errors and then emit success UX/audit outcomes.
- Verify deployment artifact policy parity: if CI publishes production artifacts, runbooks/scripts/services should deploy artifacts directly unless explicitly documented otherwise.
- Verify ingress isolation for signed callbacks/webhooks: deployment docs should define dedicated path controls (header gates/rate policy), not only generic catch-all routing.
- Verify simulation endpoint parity: test/sandbox endpoint payloads should match production contract shape plus explicit test marker fields.
- Verify release automation branch parity: branch checks, push targets, and docs use the same canonical branch name.
- Verify claim-worker transition integrity: claimed jobs persist retry/failure metadata and clear claim locks for every error class, not only transport failures.
- Verify editability-to-persistence parity: editable form/API fields have matching datastore writes (or explicit immutable handling) to avoid silent no-op updates.
- Verify contract generator extraction robustness: route/spec generators handle multiline/decorated declarations and fail loudly on omissions.
- Verify evidence boundaries: classify acceptance criteria as repo-verifiable vs environment-verifiable; report external infra items as unverified assumptions unless runtime evidence is provided.
- Verify security-hardening doc parity: when implementation is stricter/safer than spec, treat as doc/criteria drift to reconcile, not an implementation regression.
- Verify detached-work cancellation integrity in client/view-model refresh flows: canceling stale refreshes must stop the actual background work, not only suppress stale UI application.
- Verify helper-path execution stability: scripts that discover trusted helper binaries before later `cd` or subshell changes must normalize them to absolute paths before execution.
- Verify build-artifact proof for channel/store gates: release readiness checks should inspect built outputs, linkage, or runtime metadata instead of source-text greps or comments.
- Verify remediation traceability: findings-to-fixes status should remain mapped in a live checklist/spec so handoff can continue without hidden assumptions.
- Verify parser-fixture realism: layout-sensitive or OCR-adjacent parsers should be regression-tested against real extraction snapshots or source artifacts, not only hand-normalized text fixtures.
- Verify destructive-action blast radius clarity: if a UI surfaces deletion/correction from a child row but the operation actually deletes a shared/batch parent record, the UI or API contract must disclose affected siblings or offer per-allocation correction paths.
- Verify summary/detail loading discipline: list screens should not eagerly fan out into per-row detail requests when a collapsed summary payload plus on-demand detail fetch can preserve the workflow.
- Verify lazy-detail cache freshness: once detail loading becomes on-demand, refresh and collapse paths must invalidate hidden-row detail caches or the UI can surface stale history while the summary row is fresh.
- Verify suggestion/recommendation boundedness: expensive suggestion engines should prefilter candidate pools, reuse shared lookups, and avoid being invoked for every list row by default.
- Verify integration-test env parity: when the app under test runs in a separate process, env mutations in the test runner after process spawn do not affect server behavior; configure env before spawn or move env-sensitive checks to unit-level coverage.
- Verify external automation cleanup provenance: delete/cleanup jobs should correlate records to the intended run or owner with exact, boundary-safe identifiers and should include regression coverage for prefix/collision cases.
- Verify canceled-run external state integrity: if an automation can be canceled after creating external state, a later independent cleanup path or creation-prevention strategy must cover it.
- Verify review-feedback convergence: late bot/human audit comments should be triaged against the current head and converted into invariants plus focused tests when actionable.
- Verify producer/consumer scheduling for artifact promotion: the consumer's wait/poll window must be realistic relative to the producer's full critical path, including cache misses, upload latency, and job queueing.
- Verify rerun identity for CI artifacts and caches: names keyed by run attempt, shard id, branch, or matrix cell must still work for partial reruns and "rerun failed jobs" paths, or intentionally fall back with clear telemetry.
- Verify fast-path/fallback output parity: promoted/restored artifacts and locally rebuilt artifacts should use the same compiler/runtime/build command and required entrypoint checks unless divergence is explicitly tested.
- Verify fail-open/fail-closed consistency for optional optimization paths: every probe/download/cleanup/report step on a best-effort speedup must have the intended nonfatal/fatal behavior, including small "mark reason" and cleanup steps.
- Verify schema-backed env/secret optionality: deploy validation should distinguish required, optional, defaulted, and deprecated keys from the authoritative config schema and fail only for the intended classes.
- Verify shared test-state crash recovery: reused DB/cache/worker slots must run a defensive setup-time cleanup or generation check so a killed prior process cannot leak state into the next process.
- For each High/Critical finding, include at least one focused regression test/check.

## Safety and Policy Guardrails

Apply these guardrails while auditing:
- Do not provide operational abuse instructions or exploit weaponization details.
- Evaluate manipulative UX patterns as legal/trust/reputation risk, not as recommended growth tactics.
- Prioritize user safety, system integrity, and maintainable engineering outcomes.

## Output Format

Follow this response structure:

1. Findings
List only validated issues. Use the finding schema in `references/audit-framework.md`.

2. Open Questions / Assumptions
State missing context that could change priority or validity.

3. Change Summary
Summarize high-impact remediation themes in a few lines.

4. Suggested Verification
List focused tests/checks to confirm each major fix.

## Runtime Heuristics

Always apply the runtime-agnostic checklist in `references/audit-framework.md` (`Runtime-Agnostic Edge Sweep`).
If a stack-specific module exists in that file and matches the target stack, apply it as an additive overlay, not a replacement.
If no module matches, infer and state the top stack-specific risk assumptions, then continue the audit.

## Domain-Specific Audit Overlays

Use these overlays only when the target domain matches. They add to the invariant matrix and role checklists; they do not replace the baseline workflow.

### Queues, Idempotency, And Locks

Use for outboxes, schedulers, claim workers, idempotency records, repo locks, filesystem locks, retries, and distributed dispatch:
- Prove every accepted unit of user work is either durable or explicitly documented as lossy before returning success.
- Verify crash recovery for `running`, `claimed`, `pending`, `retryable`, `degraded`, `terminal`, and manual-review states.
- Verify ambiguous side effects do not create duplicate work or immortal retry loops.
- Verify 4xx/409/425/provider-specific statuses map to retry, review, or terminal states intentionally.
- Release only owner-token locks and prove stale-lock recovery cannot delete a newly reacquired lock.
- Add focused tests for each High/Critical finding that simulate crash, timeout, duplicate, stale owner, and manual recovery paths.

### CI/CD, Test Infrastructure, And Artifact Promotion

Use for GitHub Actions, merge queues, deployment workflows, build caches, artifact promotion, test sharding, reusable workflows, and local/CI test harness speedups:
- Model workflow jobs as a real DAG. Verify `needs`, `if`, `always()`, skipped prerequisite semantics, cancellation, queue delay, and final gate behavior inside the specific job block being audited, not by global string search.
- For promoted artifacts, trace producer completion time to consumer use time. Include cache-cold builds, separate export/upload steps, artifact propagation latency, partial reruns, and manual "rerun failed jobs" attempts.
- Treat optimized paths and fallback paths as equivalent contracts. Compare compiler/runtime/build commands, generated files, required entrypoints, environment variables, permissions, and cleanup side effects between the fast path and fallback path.
- For best-effort speedups, classify every step as intentionally fail-open or fail-closed. Small diagnostic, marker, cleanup, and report steps must not accidentally turn an optional optimization failure into a job failure.
- For deploy-only/redeploy paths, prove deployability with immutable manifests and artifact metadata, not only image/tag/blob existence. Treat auth, network, and non-404 storage errors differently from missing artifacts.
- For secret/env/config checks, derive requiredness from the runtime schema or equivalent authority. Report optional/defaulted drift separately from deploy-blocking missing required values.
- For shared test infrastructure, prove crash/cancel isolation. Reused databases, caches, workers, slots, ports, and temp dirs need setup-time cleanup or generation tokens because teardown hooks do not run after OOM/SIGKILL/cancelled jobs.
- For cache-key changes, include runtime/toolchain/package-manager/workspace-manifest provenance and then inspect whether the cache is actually hit, stale, overbroad, or too expensive to restore.
- For workflow permission changes, audit every job that writes statuses, deployments, artifacts, checks, comments, packages, tags, releases, or dispatches workflows; top-level permission tightening can silently remove needed job capabilities.
- Add focused probes or assertions for non-obvious workflow behavior: parse the workflow, inspect the target job's actual dependencies/conditions, mock artifact names across attempts, and validate archive restore safety before extraction.

### SSH Bootstrap And Remote Worker Trust

Use for worker provisioning, dispatch over SSH, known_hosts pinning, remote doctor checks, and tailnet hosts:
- Normalize SSH user, host, and port once and reuse that tuple for `ssh-keyscan`, known_hosts lookup, config, doctor output, and dispatch.
- Treat TOFU as bootstrap-only. Steady-state dispatch should enforce pinned trust or fail with actionable operator guidance.
- Verify config schema, backwards compatibility, provisioning, doctor, worker dispatch, and documentation together.
- Verify remote non-interactive shells use the same PATH/runtime contract as doctor checks.
- Verify key rotation paths require explicit operator action and do not silently replace pins.

### macOS Release Closure

Use for local macOS apps, `.app` bundles, release helpers, signing, notarization, stapling, Gatekeeper, and package outputs:
- Distinguish local dev/performance install lanes from distributable/notarized release lanes.
- Verify artifact inventory, stale promoted outputs, helper-path trust, cleanup traps, and output-root canonicalization.
- Reject artifact roots inside `.app` bundles or managed package roots.
- Verify direct helper invocation and top-level wrapper behavior, not only wrapper env scrubbing.
- For distributable lanes, verify built output signing, notarization acceptance, stapling, and Gatekeeper evidence.

### SwiftUI/AppKit Preview, Export, And Editor Freshness

Use for timeline editors, previews, exports, SwiftUI/AppKit bridge code, and cached derived artifacts:
- Trace every preview/export surface to the authoritative current draft/settings source.
- Verify async preview cancellation cancels real work, not only stale UI application.
- Verify zoom/scroll/key-monitor state is scoped low enough to avoid broad recomputation or disabled-state bypass.
- Preserve unknown persisted enum cases and avoid coercion on view appearance.
- Verify hidden/disabled semantics across preview loops, compilers, validators, cache keys, and tests.

### Parser, Import, And Personal-Finance Reconciliation

Use for CSV/PDF/OCR-adjacent parsers, financial imports, utility bills, split allocation, and reconciliation:
- Test parsers with real extraction snapshots or source artifacts, not only hand-normalized fixtures.
- Verify original file bytes are stored and hashed before parser APIs can detach or consume buffers.
- Verify duplicate detection uses semantic identity only when strong identifiers exist; do not collapse same-period same-amount records without a strong bill/invoice/reference key.
- Distinguish raw provider period fields from weak date heuristics.
- Keep manual reconciliation authority separate from auto-finalization.

### UI State, Persistence, And Detail Loading

Use for dashboards, admin tools, list/detail screens, bulk actions, and editable forms:
- Verify URL filters, refresh scope, visible data, and bulk-action scope stay aligned.
- Verify editable fields persist or are explicitly immutable.
- Verify collapsed summary rows do not eagerly fan out into detail requests when on-demand detail would preserve the workflow.
- Verify hidden detail caches invalidate on refresh, collapse, context switch, and import/reparse events.
- Verify destructive child-row actions disclose parent/sibling blast radius.
