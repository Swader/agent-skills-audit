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
When prior audits, third-party reviewers, or later fresh-eyes passes produced findings, include a "missed invariant" row for each non-duplicate issue so future passes check the generalized rule rather than the incident narrative.

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
- External event target invariants (provider callbacks prove both actor identity and intended target/resource identity before dispatching work or sending replies; tests include the same valid linked actor speaking in an unintended provider channel/resource, including onboarding/first-touch paths).
- Scoped parser invariants (lightweight config/source parsers must extract values from the intended object/block/scope, not the first matching token in the whole artifact; tests include distracting earlier/later/nested literals and, when available, a smoke check against the real source-of-truth artifact).
- Cross-layer validation invariants (workflow gates, shell preflights, CLI validators, and runtime/library validators reject the same sentinel and placeholder values before privileged side effects; duplicated lists need parity tests or generation).
- Local-config path invariants (user-editable identifiers that derive filenames or paths must be canonicalized, allowlisted, bounded, and prevented from escaping the intended root).
- Config authority invariants (repo/workspace-local config must not downgrade inferred/trusted sensitivity labels, widen read/write scope, grant direct-write modes, or bind local secret env names, network destinations, clone remotes, or privileged local paths unless those values match user-controlled trust; queued/retry paths revalidate the same boundary).
- Hidden-artifact invariants (identifiers that derive filenames should not create dot-prefixed or otherwise scanner-skipped artifacts that the product can no longer discover or manage).
- Sentinel semantics invariants (special values such as `0`, empty, and `NULL` have consistent meaning across interfaces and persistence logic).
- Uniqueness/conflict invariants (business uniqueness and conflict rules are datastore-enforced, not only app pre-checks).
- Flat-file mutation invariants (edits to shared line-oriented configs such as crontabs or env files use stable markers/fingerprints or fail safely on concurrent external edits; stale indices alone are insufficient).
- Lifecycle/state-machine invariants (active/archived/deleted/expired transitions are explicit and enforced consistently in read + write + destructive paths).
- Cross-trigger policy invariants (the same business policy remains consistent across user/API flows, provider callbacks, and asynchronous workers).
- Mutation-outcome invariants (success responses, audit events, and side effects are emitted only after durable write success).
- Write-freshness invariants (callback/verification/reconciliation paths avoid stale full-record rewrites; concurrent edits cannot be silently reverted).
- Scan-resilience invariants (one malformed or unreadable local config artifact should degrade per item with diagnostics, not blank the entire view or refresh result).
- Runtime/persistence source-of-truth invariants (enabled/disabled/loaded state must have one canonical authority or explicit reconciliation across persisted config and runtime override layers).
- Background-cancellation invariants (canceling superseded scans or refreshes must cancel the actual worker task and expensive filesystem/process work, not only suppress stale UI application).
- Idempotency/order invariants (retries, duplicate events, and out-of-order delivery cannot produce duplicate side effects or invalid state).
- Claim/lease lifecycle invariants (claim-based workers clear claim markers and persist attempts/status on every success/failure exit, avoiding zombie pending work).
- Time semantics invariants (timezone/DST/window boundaries and expiry logic are deterministic).
- Resource-boundedness invariants (pagination, fan-out, in-memory maps, queue growth, and retries have bounds/backpressure).
- Nested-helper boundedness invariants (direct provider-client loops and per-record fanout helpers must not bypass central pagination, retry, timeout, or cap guards).
- Layered-boundary preservation invariants (sentinel records such as triggering events, roots, idempotency markers, provenance rows, and user-visible anchors survive each independent limiter: time windows, provider pagination/page caps, local count caps, char/byte caps, and formatting/truncation passes).
- External dependency degradation invariants (timeouts, retries, fallback, and partial-failure behavior are explicit and testable).
- Lazy initialization invariants (memoized dynamic imports, provider clients, auth material, and singleton startup promises retry after rejection or intentionally enter a surfaced degraded state instead of poisoning the process until restart; direct equivalent import/client call sites should not bypass a central retryable loader without an adjacent justification).
- External lifecycle freshness invariants (webhook/lifecycle handlers that reuse earlier provider/auth reads still have a write-adjacent freshness check or datastore CAS for external token/install generations not durably fenced in local state).
- Provider retry/idempotency invariants (retry predicates distinguish transport failures from code bugs, ambiguous POST outcomes are not retried without provider-side dedupe, and informational grouping keys are not mislabeled as idempotency guarantees).
- External-send metadata invariants (action/tool flags for side effects, idempotency, confirmation, or safe retry reflect the real provider behavior; real sends/posts/payments/invites/provider writes are not marked side-effect-free to bypass confirmation friction, and any no-confirmation policy uses an explicit separate exemption).
- Downstream-idempotency retry invariants (a webhook/provider retry is only useful if downstream idempotency has not already consumed the work marker; otherwise prefer terminal ACK plus diagnostics over retry loops that cannot make progress).
- Bounded-state scan invariants (limits used to protect hot paths must not hide older valid state and fall back to duplicate work; page, cursor, or fail explicitly when the bound is reached).
- Bounded-scan retry invariants (deterministic local scan caps inside webhook/provider handlers should not throw and release a claim into retries that cannot make progress. Prefer terminal ACK plus actionable diagnostics, a durable degraded/manual-repair state, or a resumable cursor path; only fall through to new work if the cap cannot be hiding equivalent live work).
- Metrics truthfulness invariants (rates use policy-correct denominators, all-failure/no-success windows do not report healthy zeroes, score inputs are not double-counted unless intentional, sampled/capped windows are marked partial, and duration metrics use timestamp pairs that actually measure the named interval rather than a related event's own runtime; field names and docs should state the exact measured interval and any cancellation/timeout dropout window).
- Primary/fallback readback invariants (malformed primary-source records degrade per item, partial primary responses do not suppress useful fallback rows, fallback records require complete provenance/identity tags, producer-emitted fields round-trip through the stricter consumer, compatibility defaults distinguish absent legacy fields from present-but-invalid fields, fallback timestamps are validated for unit and requested window, and marker metrics are emitted only for the environment/replay scope that the fallback consumer actually reads).
- Producer/consumer fixture realism invariants (when code tightens parsers, validators, dedupe keys, or required fields, tests should use real producer output, recorded provider responses, or generated fixtures at the consumer boundary; hand-built mocks must be treated as insufficient unless paired with a round-trip test that catches missing tags, field placement drift, and provider naming differences).
- No-data metric invariants (missing series, null datapoints, empty provider responses, and zero baselines remain distinct states; current-window silence should emit/record no-data or omit the delta instead of synthesizing zero-valued trends, and downstream highlights, scores, notification copy, monitors, dashboards, and exported JSON must preserve that no-data state instead of reintroducing a false zero, while zero-to-positive lower-is-better regressions should still be scored as regressions).
- Operational-visibility invariants (diagnostic tags, guardrail states, and alert-worthy classifier outputs are surfaced by at least one intentional consumer such as a dashboard widget, monitor, digest row, searchable event, or documented artifact; emitted-but-unconsumed metadata is not sufficient observability).
- Telemetry secrecy invariants (free-form PR titles, deploy reasons, incident summaries, operator inputs, and provider messages are sanitized on live submission paths, not only dry-run output).
- External-authority reconciliation invariants (provider state changes map deterministically to local entitlement/status fields, including all required update events).
- Observability/auditability invariants (high-risk mutations and failures are traceable with actionable context and required schema fields such as actor/target and before/after context where policy expects it).
- Editability/persistence invariants (fields exposed as mutable in UI/API are either durably persisted or explicitly immutable with enforced validation and UX clarity).
- Contract evolution invariants (deprecations and replacements fail explicitly and remain docs/config/spec parity-safe).
- Contract generation invariants (route/schema generators correctly parse real declaration styles and surface omissions via deterministic stale checks).
- Security-hardening doc invariants (if implementation intentionally strengthens security over spec/docs, documentation and acceptance criteria are updated to the new baseline).
- Deployment/runbook invariants (CI artifact strategy, service bootstrap steps, ingress snippets, and release scripts remain mutually consistent and executable).
- Workflow gate DAG invariants (preflight, validation, approval, and integrity jobs must be explicit `needs` dependencies with result guards for every privileged, expensive, or side-effecting job they are meant to block; a failing parallel job is not a gate).
- Effective secret/config invariants (validation and protected runtime jobs must resolve the same repo/org/environment secret scope, alias precedence, default values, and blank-string semantics; compare effective workflow env and secret update metadata rather than local env presence alone).
- Privileged workflow trust-boundary invariants (manual dispatch inputs and refs are validated before checkout, local actions, package install, cloud/registry/Kubernetes auth, or secret-bearing env; production paths execute trusted default-branch code or an explicit inert fixture path).
- Scheduled workflow degradation invariants (cron delay, provider silence, empty discovery, and transient auth/RBAC failures produce deliberate skip/warning/fail-closed behavior rather than unbounded queueing or repeated hard-failure loops).
- Scheduled workflow timeout invariants (every recurring workflow job with a concurrency group has an explicit timeout shorter than the cadence or an intentional queueing policy).
- Workflow shell-semantics invariants (process substitution, command substitution, redirections, grouped commands, `set -e`, and `pipefail` propagate producer failures as intended under the actual runner shell).
- Third-party workflow signal invariants (action outputs, step conclusions, logs, and fail-on-miss flags have verified semantics; ambiguous signals are normalized once, empty/missing states and intentionally redundant defensive branches are explicit, cache/action hit booleans distinguish real miss from skipped/failed/unknown by checking step outcome as well as raw output, unused derived outputs are removed, and gates/telemetry/summaries consume the same normalized state without silently changing existing telemetry field types or allowed values).
- Post-side-effect workflow invariants (if a rollout, external write, or durable publication can succeed before a later gate fails, downstream jobs, compensation paths, observability, and alert wording represent the split state honestly).
- Resource-selector parity invariants (rollout, verification, telemetry, cleanup, and dashboard/digest queries that claim to describe the same deployed resource set must use equivalent selectors or explicitly document and test the divergence).
- Fallback-telemetry aggregation invariants (no-data or fallback emissions that preserve post-side-effect observability must not inflate headline success/frequency aggregates unless that is the explicit product meaning; dedicated no-data consumers should be paired with exclusion/collapse logic in primary metrics).
- Post-side-effect handoff invariants (after a privileged side effect succeeds, later verifier/collector failures should still persist enough summary or marker state for downstream telemetry/fallback paths to run honestly).
- Post-execution callback replay invariants (persisted finalization or post-check states must retain enough execution identity, input, and callback/plugin context to replay exactly-once/best-effort side effects after worker crash or retry; tests should resume from the persisted state and prove callbacks still run once).
- Post-fix dead-branch invariants (after filtering or narrowing a telemetry path, remove or test stale else-branches and tag values so future maintainers do not revive a known-bad semantic accidentally).
- Fallback-behavior docs invariants (when production code intentionally emits, suppresses, fans out, or excludes fallback/no-data telemetry, operator docs should name the tags, dashboard/monitor surfaces, and headline-metric treatment).
- Partial-submission reporting invariants (best-effort multi-step submissions must track/log success per external side effect; a later failure must not be followed by generic "submitted" copy for the failed sub-step).
- Schedule-copy parity invariants (alert/monitor messages that mention business hours, weekdays, cadence, or recovery windows must match the actual cron, query, and no-data configuration).
- Environment-parity invariants (prod, staging, preview, dry-run, and replay paths that share a telemetry contract should either share fallback behavior or document/test the intentional asymmetry).
- Mirrored workflow parity invariants (manual, force, scheduled, redeploy, and standard variants that share a side-effect contract must share equivalent markers, outputs, `if:` gates, telemetry emission, and fallback behavior; fixing one path does not prove its sibling paths are safe).
- Runtime-command parity invariants (when workflows call CLIs/scripts after a side-effect, verify the runtime command's validation/refusal paths match the workflow comment and gate; a downstream library can still drop or relabel telemetry after YAML conditions are fixed).
- Replay-state invariants (manual backfills, replays, imports, and idempotency logs must distinguish claimed/started/submitted/failed states, and retry readers should honor the latest terminal state rather than treating any historical sighting as complete).
- Classifier/tagging invariants (risk classes, change types, ownership/team tags, and route/path classifiers need explicit false-positive probes for overloaded tokens and filename conventions, not only high-confidence positive fixtures).
- Automation provenance invariants (destructive cleanup of external records such as deployments, releases, statuses, tags, caches, or cloud objects must use stable provenance evidence with exact/boundary-safe identifiers, not only broad branch/SHA/environment/name filters).
- Cancellation/stale-run invariants (automation that creates external side effects before cancellation, skip, or failure must either prevent those side effects or provide an independent cleanup path on a later run).
- Producer/consumer timing invariants (artifact/cache/status producers and consumers must be checked as a scheduled DAG with realistic wait windows, queue delays, upload propagation, and cache-cold paths).
- Retry/attempt identity invariants (partial reruns, workflow attempts, matrix shards, and manual restarts must resolve the intended logical artifact/cache/state, not silently switch to a new attempt-scoped name).
- Fast-path/fallback parity invariants (optimized paths and fallback paths must produce equivalent outputs with the same compiler/runtime/toolchain and required entrypoint checks, or document and test the divergence).
- Fast-path engagement invariants (after guardrails, probes, or fallback reasons are added, representative eligible inputs still reach the optimized path and record the intended reason; fallback-only validation can leave a green but neutered optimization).
- Fail-open/fail-closed optimization invariants (optional speedups must consistently degrade to the safe baseline; tiny marker, cleanup, and reporting steps cannot accidentally become fatal unless intended).
- Filtered/sharded test shortcut invariants (test selectors, project names, shard math, and changed-file filters are validated against authoritative test configuration; zero-test shards are intentional and distinguishable from global zero-test selection caused by filter drift, especially when `--passWithNoTests` or equivalent flags are used).
- Schema-derived config invariants (secret/env/config validators must derive required/optional/defaulted status from the authoritative runtime schema or equivalent source).
- Clean-runner env invariants (workflow tests and dry-runs should cover missing local env files and blank GitHub expression values as separate cases so developer shells do not mask CI-only failures).
- Live-scale command invariants (external CLI/API reads used in deploy, telemetry, or verification paths should be bounded and tested with realistic payload sizes and output-buffer limits, not only small fixtures).
- Additive telemetry rerun invariants (metrics, events, or records that are additive and emitted before later verification can produce duplicate/partial samples on manual retry; require idempotency, slotting, or explicit operator guidance).
- Shared test-state invariants (test speedups that reuse databases, caches, workers, slots, temp dirs, or ports must prove setup-time cleanup/generation safety after crash, cancel, OOM, and cross-file reuse).
- Simulation/contract invariants (test/sandbox/debug endpoints preserve production payload shape except explicit test markers).
- Evidence-scope invariants (repo audit conclusions distinguish code-verifiable vs environment-verifiable requirements).

## Role Checklists

### Security Expert

Check for:
- Authn/authz bypasses, privilege escalation, and missing tenant isolation.
- Cookie-authenticated CORS trust expansion: exact origins should be allowlisted; hostname-only fallbacks are unsafe because cookies ignore ports and can cross from sibling origins on the same host.
- Injection vectors (SQL/command/template), unsafe deserialization, and tainted sinks.
- Secrets handling, key management, token lifetime/revocation, and session fixation.
- Secret leakage through telemetry/logging/event text: live emitters must sanitize free-form strings such as PR titles, incident titles, deploy reasons, and operator messages, even when dry-run output is redacted.
- Idempotency/replay gaps, webhook signing/verification errors, race-prone state transitions.
- DDoS abuse surfaces: unbounded endpoints, expensive queries, amplification paths, missing rate limits.
- Entitlement bypass surfaces: expensive operations must enforce plan/tier gates on all invocation paths, not only UX entry points.
- Signed-callback ingress surfaces: webhook/provider callback paths should have dedicated ingress controls (path isolation, signature-header prefiltering, and policy-appropriate rate limits).
- Debug/dev fixture routes: data-seeding or test-only endpoints must be environment-gated or stripped from production surfaces; authentication alone is not a sufficient safeguard.
- Deactivation semantics: disabling users/admins must revoke active sessions/tokens/keys and auth middleware must re-check active status.
- Parser/policy bypasses: endpoints should not allow oversized or unexpected payload classes through content-type exceptions.
- Body-size enforcement: application handlers should enforce byte caps while streaming or before full buffering; reverse-proxy limits alone are not enough.
- Signed-token payload safety: avoid delimiter-joined signed payloads when fields can contain delimiters; prefer structured/length-safe encoding and test expiry/claim parsing.
- Destructive automation guardrails: delete/cleanup scripts should prove record ownership via exact provenance evidence; weak substring, prefix, name, or broad SHA/environment matching can delete the wrong external record.
- In-memory abuse controls: request-keyed maps (for example login attempts by IP) must have stale-key eviction and hard caps to prevent memory growth under scans.
- Canonicalization and parser split-brain risks (Unicode normalization, case-folding, path normalization, mixed parser behavior across interfaces).
- High-risk operation safeguards (step-up auth/explicit confirmation/anti-automation controls where irreversible actions exist).
- Privileged self-lockout prevention (destructive or identity-changing actions should preserve at least one recoverable admin/operator path).
- Native-action replacement parity: if a desktop or client app intercepts OS-level behavior (hotkeys, file handlers, share actions), verify it only suppresses the native path when every prerequisite needed to complete the replacement flow is present; otherwise it must fail open.
- Local privacy surfaces: inspect temp files, clipboard/pasteboard use, caches, logs, crash leftovers, and other same-user-readable artifacts for privacy-sensitive flows.
- Release helper trust: on developer, signing, or notarization machines, helper binaries/scripts resolved from `PATH` or broad filesystem searches are supply-chain risks unless explicitly pinned or verified.

### Performance Expert

Check for:
- N+1 queries, full scans, missing indexes, lock contention, and transaction scope bloat.
- Hot-path CPU/memory pressure, heavy sync work in request loops, and avoidable serialization cost.
- Inefficient build/runtime workflows: tasks that should move to async queues, batch jobs, or cron.
- CI producer/consumer timing: artifact/cache promotion must be measured against actual job DAG timing, cache-cold producer duration, upload latency, and consumer wait windows.
- Frontend payload bloat, hydration/render hotspots, and cache invalidation failures.
- Throughput/latency tail behavior under contention and degraded dependency modes.
- Cost-amplification paths: heavy operations (for example backfills/replays/rebuilds) should require explicit entitlement checks plus bounded execution.
- Fan-out dependency resilience: aggregation endpoints calling multiple upstream entities should use bounded concurrency and per-entity error handling so one failure does not blank the full response.
- Retry-storm and backpressure behavior under provider latency/failure, including queue saturation risk.
- Paid-work cancellation boundaries: cancellation/refund should stop at the point external execution is claimed/submitted unless the provider offers confirmed cancellation semantics.
- Pagination/sort stability and large-cardinality behavior (no unbounded response assembly paths).
- API pagination helpers should have explicit page/item caps and tests for never-ending full pages; provider "view" helpers that silently cap results need paginated alternatives for classifiers and metrics.
- Search for pagination implemented outside the obvious helper. A capped central client does not protect direct `getJson`/fetch loops in workflow jobs, enrichment helpers, or nested fanout code.

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
- Native-action replacement integrity: if the product replaces a platform-native action, users must not lose both the native action and the app’s replacement because the app can intercept earlier than it can complete.

### DX Expert

Check for:
- API clarity: stable contracts, explicit errors, pagination/filter semantics, and versioning hygiene.
- Provider-contract completeness: external API defaults that affect user-visible billing/lifecycle behavior should be explicit in request/webhook code and test-covered.
- Build/deploy/release parity: deployment docs, CI artifacts, system service units, and release scripts should agree on branch policy and artifact-vs-compile execution model.
- Workflow shell reliability: review runner shell flags and constructs (`mapfile`, process substitution, pipelines, redirections, grouped writes) with failure-propagation tests for any privileged or release-critical step.
- Fast-path/fallback parity: promoted/restored artifacts, deploy-only paths, and locally rebuilt fallbacks should use equivalent build commands, runtime versions, env, permissions, and entrypoint validation.
- Secret/config validation parity: deployment blockers should match runtime-required config, with optional/defaulted drift reported separately.
- Input validation parity: duplicated YAML/shell and TypeScript/runtime validators should share or regression-test placeholder and sentinel sets, especially when the later step is `continue-on-error`.
- Simulation payload parity: test/probe endpoints should emit production-shaped payloads with explicit markers rather than ad hoc minimal objects.
- Audit schema parity: audit events should include required target and state-diff fields defined by policy/spec.
- Mutation error signaling parity: mutating handlers should propagate write failures (or explicit failure UX) and avoid success logs/redirects on failed writes.
- Editability/write parity: fields offered as editable in docs/UI/API should map to actual datastore writes (or be explicitly immutable and rejected when submitted).
- Sentinel-config parity: shared config values (for example numeric limits) should have one documented meaning across UI/API/worker paths.
- Security hardening parity: stronger implementation controls should be reflected in specs/runbooks to prevent stale weaker guidance.
- Code readability/extensibility: module boundaries, coupling, dead abstractions, and naming quality.
- Primitive reuse: before accepting bespoke lock/cache/queue/rate-limit/provider-client code, search for existing project primitives and require a concrete mismatch before custom infrastructure logic remains.
- Test strategy gaps: missing integration/contract/load tests for critical paths.
- Onboarding quality: concise docs, runbooks, architecture notes, and executable examples.
- LLM/operator friendliness: discoverable conventions and deterministic workflows.
- CLI state secrecy: session/key state files should be written with restrictive permissions and chmodded after overwrites, not only created with a restrictive mode.
- Cross-route consistency: equivalent capabilities must enforce equivalent validation/invariants.
- Deprecation contract parity: deprecated endpoints should return explicit, machine-readable replacement details and consistent status semantics.
- Bot-instruction parity: API docs and agent/skill guidance must match live endpoint behavior (including batch orchestration limits and lifecycle semantics).
- Contract-generation robustness: route/spec generation should be resilient to multiline/decorated declarations and protected by freshness checks that fail on omissions.
- Manual debug secrecy: docs and runbooks that query credential models must redact top-level and nested token/refresh-token fields, not only the most obvious access token.
- Remediation traceability: findings should map to an implementation checklist/spec with closure status so handoffs can continue without re-auditing from scratch.
- Review-feedback hygiene: bot/human review comments should be evaluated against the current head, classified as actionable/stale/false-positive/hygiene, and converted into invariants plus focused checks when real.
- Original-thread closure: when bot summaries claim reviewer feedback is fixed, verify the original inline thread/comment against the current head and leave an evidence reply or explicit stale/false-positive disposition.
- Current-head CI evidence: distinguish current-head required checks from superseded workflow runs; if current-head CI is pending behind an older run, inspect and cancel only obsolete blockers.
- Change safety rails: feature flags/migrations/config toggles should include rollback and compatibility strategy.
- Release tooling trust parity: scripts used on signing/notarization/release machines should resolve helper binaries from trusted explicit paths, not from `PATH` or broad workspace / DerivedData searches.
- Preview/export source-of-truth parity: exported output should render from current authoritative settings or state, not from stale async preview caches.

### Edge Case Master

Check for:
- Rare-state transitions and multi-step flow interactions that break invariants.
- Time boundaries, timezone drift, retries, duplicate events, and out-of-order processing.
- Layered cap edge cases: bounded context, prompt, attribution, and notification pipelines that preserve a triggering/root record should test each independent limiter separately, including time windows, provider pagination, local count caps, and char/byte truncation.
- Derived delivery metrics under pathological windows: all failures, no successes, empty windows, capped samples, duplicate score inputs, date-only backfills, and fallback recovery signals should have focused tests before dashboards are trusted.
- Provider no-data windows are their own edge case: "no datapoint" should not become 0, -100%, or a healthy zero-denominator rate unless that is the explicitly tested product meaning. Follow the state into every operator-facing renderer, highlight selector, notification payload, monitor, dashboard tile, and score formula because false zeroes often reappear after the data-access layer was fixed. Also distinguish a missing prior baseline from a previous value of zero; zero-to-bad transitions often need stronger treatment than ordinary percentage deltas can express.
- Recovery and duration metrics should be audited against their lifecycle semantics. A rollback or force-deploy event can confirm a change failure, but its own start/finish timestamps measure recovery-action duration, not time-to-recovery, unless the code joins it to the original failure's detection/start time.
- Cross-system races (jobs, webhooks, external providers, same-host side effects).
- Callback stale-write races: verify async verification/reconciliation responses cannot overwrite newer user edits via full-record updates.
- Policy drift across trigger classes: verify lifecycle rules are not implemented differently in route handlers, webhook handlers, and workers.
- Silent-failure success paths: verify transient write failures cannot produce success UX, success audit rows, or state-desync side effects.
- Non-obvious abuse chains combining medium findings into critical outcomes.
- Spec-vs-implementation mismatches hidden in user stories rather than obvious code smells.
- Structural anomalies: self-links and indirect cycles in hierarchical data.
- Context-shift races: switching global context while a page/process is active must not leave stale data, invalid query shapes, or half-updated summaries.
- Matcher disambiguation edges: explicit mappings should beat heuristics, and near-matches should avoid partial-token false positives.
- Automation identity edges: exact IDs and provenance links should beat workflow/job/display-name fallbacks; test prefix collisions, stale runs, skipped jobs, and cancellation windows for cleanup logic.
- Retry/attempt edges: test partial reruns, rerun-failed-jobs, incremented attempts, sharded retries, and manual restart paths for artifact/cache lookup correctness.
- Shared-state crash edges: test reused test DB/cache/worker/slot state after prior process crash or cancellation, not only after normal teardown hooks run.
- Optional-optimization edges: verify every probe, marker, cleanup, and report step on a best-effort speedup has the intended fatal/nonfatal behavior under failure.
- Partial-update edge cases: validate resulting state (`existing + patch`) so cross-field invariants cannot be bypassed.
- Null/empty/zero/missing ambiguity: ensure business logic does not conflate sentinel states.
- Cursor/page drift: ensure stable ordering and deterministic pagination under concurrent writes.
- Numeric precision and serialization edges: float/decimal/BigInt conversions must not silently corrupt values.
- Boundary payload behavior: deep nesting, large arrays/files, and malformed encodings should fail safely.
- Interception/completion mismatches: verify observe/suppress permission checks match complete-execution prerequisites for privileged desktop flows.
- Global-trigger serialization: repeated hotkeys, auto-repeat, re-entrant callbacks, or duplicate monitors must not overlap privileged flows or create window/process storms.
- No-op edit/export behavior: if product semantics allow “open then save unchanged”, verify unchanged editor sessions still export successfully.

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
- Single-use provider token refresh: verify a durable generation/lease fence, not only a volatile cache lock, prevents a second worker from consuming or revoking the same token generation after lock expiry.
- Single-use provider refresh degradation: if a consumed/invalid refresh token is intentionally left active for reconciliation instead of revoked, verify retries are backoff-bounded and operator/user-visible rather than hammering the provider on every send.
- Provider secret minimization: verify token fanout/sync writes fresh credentials only into active/intended credential rows and does not refresh revoked/inactive records.
- Provider fanout path coverage: when one provider token fanout path is hardened, search for sibling fanout/sync paths that copy the same credential material; the invariant must hold across user-token, bot-token, metadata, and linked install syncs.
- Provider lifecycle timestamp precision: if provider lifecycle events use coarse timestamps, verify every destructive same-bucket event type, including uninstall and token-revoked, cannot corrupt a newer install; prefer explicit install generation IDs over timestamp-only fencing.
- Dedupe claim release safety: verify failed handlers release event/claim locks with compare-and-delete or equivalent owner checks, not unconditional deletion.
- OAuth post-exchange rejection cleanup: verify every local-policy rejection after a provider issues tokens has an explicit cleanup decision, including malformed/missing token-rotation fields and ownership/membership failures.
- OAuth cleanup freshness: cleanup decisions must re-read current local state at cleanup time and suppress provider uninstall/revoke if the current request or a concurrent request has already persisted an active local install; stale pre-transaction snapshots are not sufficient.
- OAuth cleanup in-flight safety: a cleanup-time re-read cannot see an uncommitted valid install. Destructive provider cleanup after token exchange should either serialize cleanup and install persistence by the provider install key, use a delayed cleanup/reconciliation job, or fail safe by skipping cleanup when coordination is unavailable.
- OAuth retry freshness: duplicate-key or transaction retries must re-read current ownership/install state inside the retried operation, not capture a pre-retry snapshot that can become stale after a concurrent writer commits.
- Handoff/archive verification: verify generated archives against a saved manifest/file list rather than relying on `producer | grep -q` under `pipefail`; early grep exit can SIGPIPE the producer and create false missing-file results.
- Signed URL/token parsing: verify structured payload encoding, delimiter safety, expiry parsing, and claim binding.
- Paid job cancellation/refund boundaries: verify refunds are only available before external execution can incur cost, or that provider-side cancellation is confirmed.
- Claim-worker progress semantics: verify claimed/dequeued jobs always transition attempts/status and clear claim markers across all failure classes.
- Cache/permission freshness: verify permission and lifecycle changes invalidate stale cache/session views.
- Resource ceilings: verify caps/backpressure/TTL for queues, maps, fan-out loops, and payload parsing paths.
- Workflow shell failure propagation: verify release-critical shell constructs preserve producer failures under the actual runner shell, including process substitution, command substitution, and output-file redirections.
- Post-side-effect failure states: verify workflows that can fail after a rollout/publication/external write have explicit continuation or compensation for post-rollout jobs, accurate telemetry, and non-misleading alerts.
- Mirrored workflow parity: after fixing a release/deploy path, search sibling workflows for the same side-effect sequence and compare markers, outputs, `continue-on-error`, `if:` conditions, fallback emits, and notification copy line by line.
- Runtime refusal path check: for every workflow command expected to emit telemetry or compensation after a failure, inspect the CLI/library guard conditions using the exact flags passed by the workflow; do not stop at "the step now runs."
- Replay state progression: inspect append/read behavior together; preflight or started markers written before side effects should not suppress future retries unless a later submitted/success marker exists.
- Scheduled collector no-data handling: verify empty upstream discovery and missing provider data are surfaced as no-data/warning states or explicitly fail-closed; do not let a recurring job fail-loop every interval because no-data is collapsed into mismatch.
- Scheduled workflow timeout coverage: verify each cron job's explicit timeout and concurrency policy together; fixing one scheduled workflow does not prove sibling scheduled workflows cannot queue for the platform default timeout.
- Metric truth edge cases: verify failure rates, health scores, highlights, notification copy, and sampled digests across all-failure/no-success windows, no-data windows, zero-denominator windows, duplicate inputs, and cap-hit partial windows.
- Startup/recovery resilience: verify transient dependency failures do not permanently poison initialization state.
- Entitlement parity and async revalidation: verify policy checks exist on all trigger paths and are revalidated when work is dequeued/executed.
- Callback write safety: verify callback/verification handlers use conditional field-scoped updates (or version checks) rather than stale full-object rewrites.
- Provider-policy explicitness: verify app-required billing/lifecycle semantics are encoded explicitly, not left to third-party defaults.
- URL/query safety in navigation: verify pagination/filter controls preserve reserved characters via encoding-safe mechanisms (for example GET forms with hidden fields).
- Cross-trigger policy parity: verify equivalent business transitions (for example downgrade timing, reset authority, pause/resume rules) do not diverge between interactive, callback, and worker paths.
- Sentinel semantics parity: verify special values (for example `0`) keep identical behavior across APIs, admin panels, and background logic.
- Mutation acknowledgement integrity: verify state-changing handlers do not ignore datastore write errors before emitting success responses/audit signals.
- Editability/persistence parity: verify user-editable fields are not silently dropped between handler and datastore update statements.
- Artifact-first deploy parity: if build pipeline ships production artifacts, verify runbooks/scripts avoid unnecessary on-host compilation and toolchain coupling.
- Deploy-only artifact provenance: redeploy/rollback paths that skip build or tests must prove the selected artifact came from a successful gated run, not merely that mutable image tags or object prefixes exist; validate an artifact manifest with image digest, build timestamp, and CDN/object metadata before rollout.
- Signed webhook ingress isolation: verify deployment guidance includes dedicated callback ingress snippets with explicit prefilter and rate policy, not only generic catch-all locations.
- Simulation endpoint contract parity: verify synthetic/test payloads mirror production schema plus explicit test marker fields.
- Release branch consistency: verify branch checks and push targets in release automation use one canonical default branch.
- Contract generator extraction safety: verify spec generators parse real route declaration formatting (including multiline routes) and stale checks catch omissions.
- Evidence boundary classification: verify final report separates repository-validated findings from environment-only checks requiring runtime/infra access.
- Spec-hardening drift: verify stronger security implementation details are propagated into specs/acceptance criteria to avoid false regression labeling.
- Native-action interception parity: whenever a system intercepts or replaces platform-native actions, verify it can actually complete the replacement flow before suppression and that missing prerequisites fail open to the native path.
- Clipboard transport parity: local desktop flows should not route sensitive payloads through the global pasteboard unless clipboard output is the explicit requested result; otherwise audit same-user listeners, clipboard managers, restoration races, and rich-payload memory spikes.
- Async preview/export parity: verify exported output is derived from current source-of-truth settings, not stale asynchronously rendered preview state.
- Helper-binary trust: verify release / packaging scripts do not execute helper tools from untrusted `PATH` entries or broad filesystem discovery.
- Relative helper execution safety: if packaging scripts later change directories, helper paths resolved earlier must already be absolute and still valid at execution time.
- External automation cleanup provenance: destructive CI/CD cleanup should correlate records to the intended run or owner via exact/boundary-safe identifiers and include regression coverage for prefix/collision cases.
- Manual privileged workflow boundary: verify `workflow_dispatch` and reusable deployment workflows validate branch/ref and untrusted inputs before checkout, local actions, package install, auth, or secrets; gates that only wrap final writes are too late.
- Canceled-run side-effect cleanup: if automation can be canceled after creating external records, verify a later independent cleanup path or creation-prevention strategy handles the orphaned state.
- Classifier false-positive sweep: for every path/tag classifier, test representative tooling files, docs with risky words, generated configs, and benign filenames that contain high-risk tokens; noisy labels erode operator trust in dashboards and alerts.
- Artifact producer/consumer scheduling: verify consumer wait/poll windows account for the producer's full critical path, including separate export/upload work and cache-cold rebuilds.
- CI rerun artifact identity: verify artifact/cache names and lookup logic work for partial reruns, incremented run attempts, and shard-specific retries, or fall back with explicit telemetry.
- Third-party workflow action state: verify exported hit/status outputs against actual restore/save logs and step conclusions, then audit every downstream consumer for consistent normalized semantics rather than duplicating local interpretations; if a boolean telemetry tag needs richer state, preserve the boolean field and add a separate status/versioned field unless all consumers are updated together.
- Fast-path/fallback artifact equivalence: verify optimized artifact restore/promotion and baseline local rebuild use the same compiler/runtime/toolchain and validate the same runtime-loaded entrypoints.
- Optional speedup failure policy: verify best-effort speedups consistently fail open to the baseline, while deployability/safety gates fail closed; do not let diagnostic/cleanup/report steps invert that policy.
- Schema-backed secret optionality: verify missing env/secret checks classify required, optional, defaulted, and deprecated keys from the runtime schema rather than only rendered manifests.
- Shared test-state crash isolation: verify reused test databases/caches/workers/ports have setup-time cleanup or generation tokens so a killed process cannot contaminate the next test file or shard.
- Active-review convergence: when auditing a PR with bot/human feedback, verify latest comments against the current head and treat actionable comments as new evidence that must either be fixed or explicitly dispositioned.
- External-review miss analysis: when a third-party reviewer finds an issue the audit missed, classify whether the miss came from absent invariant coverage, insufficient evidence gathering, stale scope, weak role prompting, missing runtime verification, or missing regression tests.

## Runtime Module: macOS Desktop Utility

Apply these checks only when the target is a local macOS app or helper with AppKit/SwiftUI/global input/system integration:
- Hotkey/event-tap permission parity: the app should not intercept screenshot or system shortcuts unless the full permission set required to complete the replacement capture/export flow is available.
- Fail-open native behavior: when required permissions are missing or revoked, native macOS behavior should continue instead of being silently suppressed.
- Same-user artifact exposure: privacy-sensitive captures should avoid raw temp files and non-explicit clipboard writes when possible; if temp storage is unavoidable, verify exposure window, immediate deletion, migration cleanup, and crash leftovers.
- Override-state UX parity: when interception can fail open or self-disable, the app should surface active/inactive override state and a clear retry path in Settings or equivalent UI, not only in logs or internal model state.
- Launchd/cron capability parity: read-only or non-owned jobs must be gated in both the UI and execution layer, and "run now" flows must not persistently change disabled state merely to force execution.
- Launchd state-reconciliation parity: reconcile plist `Disabled` flags and `launchctl` disabled overrides consistently across load, save, enable/disable, and run-now paths, especially when app-managed jobs coexist with hand-authored plists.
- Launchd label/path integrity: job labels or names that derive plist paths must use strict allowlists, bounded length, and no direct path-component reuse.
- Hidden-config discoverability: labels or names that derive config filenames should reject leading-dot or equivalent hidden-file cases when scanners intentionally skip hidden entries.
- Local scheduler scan resilience: malformed plists, unreadable job files, or unusual filesystem entries should produce per-item warnings rather than a full refresh failure.
- Runtime probe diagnostics: nonzero `launchctl` or equivalent scheduler-state probe exits should surface structured warnings or blocking errors on mutating paths instead of silently degrading to empty state.
- Flat-file scheduler mutation safety: cron or other line-oriented scheduler edits should use fingerprints/markers and conflict errors instead of blind index-based rewrites.
- Refresh cancellation integrity: view-model generation guards alone are insufficient; verify detached refresh work is canceled directly and has cooperative cancellation checkpoints inside scans/probes.
- Capture and monitor serialization: repeated global shortcuts, auto-repeat, and duplicate event monitors should not spawn overlapping `screencapture` processes, window stacks, or resource spikes.
- Editor/export correctness: unchanged editor sessions should export if expected, and immediate export after setting changes should use current settings, not stale preview renders.
- Channel/store gate proof: release-channel packaging should validate built artifact metadata, linkage, resources, or runtime markers instead of source-text greps or comments.
- Release host trust: packaging, signing, and appcast tooling should use trusted pinned helper binaries and avoid “first match wins” discovery across the repo, `PATH`, or DerivedData.

## Runtime Module: Bun + SQLite

Apply these checks only when stack includes Bun server routes and SQLite:
- JSON parsing downgrade check: flag endpoints that do `await request.json()` and on `catch` silently set `payload = {}`. Invalid JSON should return `400`; only truly empty bodies should default.
- Content-Type normalization check: media-type policy comparisons should normalize header casing (`toLowerCase()`), especially for multipart gating.
- Broad-catch downgrade check: in reconciliation/import loops, flag `catch { skipped++ }` patterns that convert unknown failures into success-like responses. Only known recoverable codes should be downgraded.
- SQLite trigger accounting check: when using `run().changes` for conflict detection, remember AFTER UPDATE triggers can increase reported changes; treat `< 1` as no-op/conflict, not `!== 1`.
- In-memory map growth check: for Maps/objects keyed by request-derived values (IP, token, path), require TTL cleanup and/or max-key caps, plus a regression test that simulates many unique keys.
