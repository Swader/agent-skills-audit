# Audit Framework

Use this file as the operating checklist and output schema for the `audit-code` skill.

## Severity Rubric

- Critical: Immediate compromise, major data loss, financial loss, legal exposure, or service-wide outage likely.
- High: Exploitable or user-impacting defect with significant business risk but not immediate systemic collapse.
- Medium: Material weakness that can compound under scale/load or specific conditions.
- Low: Limited impact, hygiene issues, or improvements with small downside if deferred.

## Finding Schema

Use this structure for every finding:

- ID: Stable identifier, for example `SEC-001`, `PERF-003`.
- Role: `Security`, `Performance`, `UX`, `DX`, or `Edge`.
- Title: One-line defect statement.
- Severity: `Critical | High | Medium | Low`.
- Confidence: `High | Medium | Low`.
- Impact: Business/user/system impact in plain language.
- Evidence: File path + line reference + behavior summary.
- Trigger Conditions: Inputs, load profiles, user flows, or race conditions required.
- Reproduction: Minimal steps (or reason not reproducible in current context).
- Proposed Fix: Specific code or architecture change.
- Verification: Targeted test/check to validate the fix.
- Dependencies: Cross-team or sequencing constraints, if any.

## Invariant Coverage Matrix (Required)

Build this before role pass 1, then reuse it in pass 2.

For each invariant, list all mutating entry points (routes, webhooks, workers, scripts) and verify parity:
- Invariant: what must always remain true.
- Entry Points: every code path that can violate it.
- Guard Type: transactionality, conflict checks, auth checks, validation, media-type policy.
- Gap: missing or inconsistent enforcement.

Minimum invariants to include in every audit:
- Data integrity invariants (linked writes remain atomic/consistent across stores and async boundaries).
- Access-control and scope isolation invariants (authz checks and tenant/workspace/account boundaries are enforced on every read/write path).
- Entitlement and policy invariants (plan/tier/feature flags and expensive-operation rights are enforced across API/UI/webhook/job paths, with revalidation before queued execution).
- Identity lifecycle invariants (disable/revoke/role changes take effect across active sessions/tokens/keys).
- Input/protocol invariants (validation, canonicalization, parser behavior, and payload/media-type limits are consistent across equivalent entry points).
- Sentinel semantics invariants (special values such as `0`, empty, and `NULL` have consistent meaning across interfaces and persistence logic).
- Uniqueness/conflict invariants (business uniqueness and conflict rules are datastore-enforced, not only app pre-checks).
- Lifecycle/state-machine invariants (active/archived/deleted/expired transitions are explicit and enforced consistently in read + write + destructive paths).
- Cross-trigger policy invariants (the same business policy remains consistent across user/API flows, provider callbacks, and asynchronous workers).
- Mutation-outcome invariants (success responses, audit events, and side effects are emitted only after durable write success).
- Write-freshness invariants (callback/verification/reconciliation paths avoid stale full-record rewrites; concurrent edits cannot be silently reverted).
- Idempotency/order invariants (retries, duplicate events, and out-of-order delivery cannot produce duplicate side effects or invalid state).
- Time semantics invariants (timezone/DST/window boundaries and expiry logic are deterministic).
- Resource-boundedness invariants (pagination, fan-out, in-memory maps, queue growth, and retries have bounds/backpressure).
- External dependency degradation invariants (timeouts, retries, fallback, and partial-failure behavior are explicit and testable).
- External-authority reconciliation invariants (provider state changes map deterministically to local entitlement/status fields, including all required update events).
- Observability/auditability invariants (high-risk mutations and failures are traceable with actionable context and required schema fields such as actor/target and before/after context where policy expects it).
- Contract evolution invariants (deprecations and replacements fail explicitly and remain docs/config/spec parity-safe).
- Security-hardening doc invariants (if implementation intentionally strengthens security over spec/docs, documentation and acceptance criteria are updated to the new baseline).
- Deployment/runbook invariants (CI artifact strategy, service bootstrap steps, ingress snippets, and release scripts remain mutually consistent and executable).
- Simulation/contract invariants (test/sandbox/debug endpoints preserve production payload shape except explicit test markers).
- Evidence-scope invariants (repo audit conclusions distinguish code-verifiable vs environment-verifiable requirements).

## Role Checklists

### Security Expert

Check for:
- Authn/authz bypasses, privilege escalation, and missing tenant isolation.
- Injection vectors (SQL/command/template), unsafe deserialization, and tainted sinks.
- Secrets handling, key management, token lifetime/revocation, and session fixation.
- Idempotency/replay gaps, webhook signing/verification errors, race-prone state transitions.
- DDoS abuse surfaces: unbounded endpoints, expensive queries, amplification paths, missing rate limits.
- Entitlement bypass surfaces: expensive operations must enforce plan/tier gates on all invocation paths, not only UX entry points.
- Signed-callback ingress surfaces: webhook/provider callback paths should have dedicated ingress controls (path isolation, signature-header prefiltering, and policy-appropriate rate limits).
- Deactivation semantics: disabling users/admins must revoke active sessions/tokens/keys and auth middleware must re-check active status.
- Parser/policy bypasses: endpoints should not allow oversized or unexpected payload classes through content-type exceptions.
- In-memory abuse controls: request-keyed maps (for example login attempts by IP) must have stale-key eviction and hard caps to prevent memory growth under scans.
- Canonicalization and parser split-brain risks (Unicode normalization, case-folding, path normalization, mixed parser behavior across interfaces).
- High-risk operation safeguards (step-up auth/explicit confirmation/anti-automation controls where irreversible actions exist).

### Performance Expert

Check for:
- N+1 queries, full scans, missing indexes, lock contention, and transaction scope bloat.
- Hot-path CPU/memory pressure, heavy sync work in request loops, and avoidable serialization cost.
- Inefficient build/runtime workflows: tasks that should move to async queues, batch jobs, or cron.
- Frontend payload bloat, hydration/render hotspots, and cache invalidation failures.
- Throughput/latency tail behavior under contention and degraded dependency modes.
- Cost-amplification paths: heavy operations (for example backfills/replays/rebuilds) should require explicit entitlement checks plus bounded execution.
- Fan-out dependency resilience: aggregation endpoints calling multiple upstream entities should use bounded concurrency and per-entity error handling so one failure does not blank the full response.
- Retry-storm and backpressure behavior under provider latency/failure, including queue saturation risk.
- Pagination/sort stability and large-cardinality behavior (no unbounded response assembly paths).

### UX Expert

Check for:
- User journey friction: unnecessary steps, dead-ends, poor defaults, weak state feedback.
- Error/empty/loading states and perceived performance.
- Derived metric correctness: displayed formulas and progress denominators should match policy math (including top-ups/adjustments/reset windows), not simplified approximations.
- Accessibility basics: keyboard flow, labels, focus handling, contrast, ARIA correctness.
- Human and bot operator flows where relevant (APIs, machine-consumable outputs, predictable contracts).
- API error actionability for bots: verify error `details` includes actionable next-step context when policy blocks input classes (for example allowed routes and received content type on multipart rejection).
- Minimal internal auth UX story: ensure there is a login path, core navigation to operational pages, and clear session-expired re-auth guidance.
- Global context-switch behavior: whichever context selector exists (mode/tenant/workspace/environment) should refresh visible data in-place and clear now-invalid local filters/groupings.
- Trust risks from coercive or manipulative patterns; flag compliance/reputation exposure.
- Destructive-action ergonomics: explicit confirmation, recoverability/undo expectations, and clear blast-radius communication.

### DX Expert

Check for:
- API clarity: stable contracts, explicit errors, pagination/filter semantics, and versioning hygiene.
- Provider-contract completeness: external API defaults that affect user-visible billing/lifecycle behavior should be explicit in request/webhook code and test-covered.
- Build/deploy/release parity: deployment docs, CI artifacts, system service units, and release scripts should agree on branch policy and artifact-vs-compile execution model.
- Simulation payload parity: test/probe endpoints should emit production-shaped payloads with explicit markers rather than ad hoc minimal objects.
- Audit schema parity: audit events should include required target and state-diff fields defined by policy/spec.
- Mutation error signaling parity: mutating handlers should propagate write failures (or explicit failure UX) and avoid success logs/redirects on failed writes.
- Sentinel-config parity: shared config values (for example numeric limits) should have one documented meaning across UI/API/worker paths.
- Security hardening parity: stronger implementation controls should be reflected in specs/runbooks to prevent stale weaker guidance.
- Code readability/extensibility: module boundaries, coupling, dead abstractions, and naming quality.
- Test strategy gaps: missing integration/contract/load tests for critical paths.
- Onboarding quality: concise docs, runbooks, architecture notes, and executable examples.
- LLM/operator friendliness: discoverable conventions and deterministic workflows.
- Cross-route consistency: equivalent capabilities must enforce equivalent validation/invariants.
- Deprecation contract parity: deprecated endpoints should return explicit, machine-readable replacement details and consistent status semantics.
- Bot-instruction parity: API docs and agent/skill guidance must match live endpoint behavior (including batch orchestration limits and lifecycle semantics).
- Remediation traceability: findings should map to an implementation checklist/spec with closure status so handoffs can continue without re-auditing from scratch.
- Change safety rails: feature flags/migrations/config toggles should include rollback and compatibility strategy.

### Edge Case Master

Check for:
- Rare-state transitions and multi-step flow interactions that break invariants.
- Time boundaries, timezone drift, retries, duplicate events, and out-of-order processing.
- Cross-system races (jobs, webhooks, external providers, same-host side effects).
- Callback stale-write races: verify async verification/reconciliation responses cannot overwrite newer user edits via full-record updates.
- Policy drift across trigger classes: verify lifecycle rules are not implemented differently in route handlers, webhook handlers, and workers.
- Silent-failure success paths: verify transient write failures cannot produce success UX, success audit rows, or state-desync side effects.
- Non-obvious abuse chains combining medium findings into critical outcomes.
- Spec-vs-implementation mismatches hidden in user stories rather than obvious code smells.
- Structural anomalies: self-links and indirect cycles in hierarchical data.
- Context-shift races: switching global context while a page/process is active must not leave stale data, invalid query shapes, or half-updated summaries.
- Matcher disambiguation edges: explicit mappings should beat heuristics, and near-matches should avoid partial-token false positives.
- Partial-update edge cases: validate resulting state (`existing + patch`) so cross-field invariants cannot be bypassed.
- Null/empty/zero/missing ambiguity: ensure business logic does not conflate sentinel states.
- Cursor/page drift: ensure stable ordering and deterministic pagination under concurrent writes.
- Numeric precision and serialization edges: float/decimal/BigInt conversions must not silently corrupt values.
- Boundary payload behavior: deep nesting, large arrays/files, and malformed encodings should fail safely.

## Two-Pass Execution Rule

1. Complete pass 1 for Security, Performance, UX, DX, then Edge.
2. Run tie-breaker review to reconcile conflicts.
3. Re-run Security/Performance/UX/DX using edge findings as new attack/load/flow assumptions.
4. Finish with Edge final pass to validate residual risk after proposed mitigations.
5. Produce one merged final report from the tie-breaker lead.

## Final Report Template

Use this exact section order:

1. Findings (sorted by severity, blast radius, exploitability)
2. Open Questions / Assumptions
3. Remediation Plan (Now / Next / Later)
4. Verification Plan
5. Executive Summary

## Runtime-Agnostic Edge Sweep

Run this sweep for every audit, regardless of stack:
- Partial update vs full replace semantics: verify cross-field invariants on resulting state, not just provided keys.
- Null/empty/zero/missing differentiation: verify defaults and validations do not collapse distinct states.
- Stable ordering and pagination: verify deterministic sort keys, cursor shape validation, and no duplicate/skip drift under writes.
- Canonicalization and encoding: verify Unicode/case/path normalization and parser parity across interfaces.
- Numeric and temporal boundaries: verify precision/overflow/rounding handling and timezone/DST/expiry boundary behavior.
- Retry/idempotency behavior: verify dedupe keys, idempotency key claiming order, and safe replay semantics.
- Cache/permission freshness: verify permission and lifecycle changes invalidate stale cache/session views.
- Resource ceilings: verify caps/backpressure/TTL for queues, maps, fan-out loops, and payload parsing paths.
- Startup/recovery resilience: verify transient dependency failures do not permanently poison initialization state.
- Entitlement parity and async revalidation: verify policy checks exist on all trigger paths and are revalidated when work is dequeued/executed.
- Callback write safety: verify callback/verification handlers use conditional field-scoped updates (or version checks) rather than stale full-object rewrites.
- Provider-policy explicitness: verify app-required billing/lifecycle semantics are encoded explicitly, not left to third-party defaults.
- URL/query safety in navigation: verify pagination/filter controls preserve reserved characters via encoding-safe mechanisms (for example GET forms with hidden fields).
- Cross-trigger policy parity: verify equivalent business transitions (for example downgrade timing, reset authority, pause/resume rules) do not diverge between interactive, callback, and worker paths.
- Sentinel semantics parity: verify special values (for example `0`) keep identical behavior across APIs, admin panels, and background logic.
- Mutation acknowledgement integrity: verify state-changing handlers do not ignore datastore write errors before emitting success responses/audit signals.
- Artifact-first deploy parity: if build pipeline ships production artifacts, verify runbooks/scripts avoid unnecessary on-host compilation and toolchain coupling.
- Signed webhook ingress isolation: verify deployment guidance includes dedicated callback ingress snippets with explicit prefilter and rate policy, not only generic catch-all locations.
- Simulation endpoint contract parity: verify synthetic/test payloads mirror production schema plus explicit test marker fields.
- Release branch consistency: verify branch checks and push targets in release automation use one canonical default branch.
- Evidence boundary classification: verify final report separates repository-validated findings from environment-only checks requiring runtime/infra access.
- Spec-hardening drift: verify stronger security implementation details are propagated into specs/acceptance criteria to avoid false regression labeling.

## Runtime Module: Bun + SQLite

Apply these checks only when stack includes Bun server routes and SQLite:
- JSON parsing downgrade check: flag endpoints that do `await request.json()` and on `catch` silently set `payload = {}`. Invalid JSON should return `400`; only truly empty bodies should default.
- Content-Type normalization check: media-type policy comparisons should normalize header casing (`toLowerCase()`), especially for multipart gating.
- Broad-catch downgrade check: in reconciliation/import loops, flag `catch { skipped++ }` patterns that convert unknown failures into success-like responses. Only known recoverable codes should be downgraded.
- SQLite trigger accounting check: when using `run().changes` for conflict detection, remember AFTER UPDATE triggers can increase reported changes; treat `< 1` as no-op/conflict, not `!== 1`.
- In-memory map growth check: for Maps/objects keyed by request-derived values (IP, token, path), require TTL cleanup and/or max-key caps, plus a regression test that simulates many unique keys.
