# Domain Audit Overlays

Use these overlays only when the target domain matches. They add to the invariant matrix and role checklists; they do not replace the baseline workflow.

### Queues, Idempotency, And Locks

Use for outboxes, schedulers, claim workers, idempotency records, repo locks, filesystem locks, retries, and distributed dispatch:
- Apply this overlay only when such a system is already in the audited mission. If a small fix unexpectedly creates one, treat that introduction as a scope tripwire and first look for a smaller design.
- Prove every accepted unit of user work is either durable or explicitly documented as lossy before returning success.
- Verify crash recovery for `running`, `claimed`, `pending`, `retryable`, `degraded`, `terminal`, and manual-review states.
- Verify ambiguous side effects do not create duplicate work or immortal retry loops.
- Verify 4xx/409/425/provider-specific statuses map to retry, review, or terminal states intentionally.
- Release only owner-token locks and prove stale-lock recovery cannot delete a newly reacquired lock.
- Add the smallest focused checks needed for each admitted High/Critical finding. Cover crash, timeout, duplicate, stale owner, or manual recovery only when that failure mode is demonstrated or created by the patch.

### CI/CD, Test Infrastructure, And Artifact Promotion

Use for GitHub Actions, merge queues, deployment workflows, build caches, artifact promotion, test sharding, reusable workflows, and local/CI test harness speedups:
- If the app under test is a separate process, changing the runner's environment after spawn does not change the server's environment. Configure it before spawn or test the environment-sensitive behavior at the appropriate boundary.
- Model workflow jobs as a real DAG. Verify `needs`, `if`, `always()`, skipped prerequisite semantics, cancellation, queue delay, and final gate behavior inside the specific job block being audited, not by global string search.
- For promoted artifacts, trace producer completion time to consumer use time. Include cache-cold builds, separate export/upload steps, artifact propagation latency, partial reruns, and manual "rerun failed jobs" attempts.
- Treat optimized paths and fallback paths as equivalent contracts. Compare compiler/runtime/build commands, generated files, required entrypoints, environment variables, permissions, and cleanup side effects between the fast path and fallback path.
- After adding or tightening safety guards around an optimization, run a representative positive-path smoke/probe that proves the optimization still engages and records the expected reason; fallback-only validation can leave the ROI silently disabled.
- For best-effort speedups, classify every step as intentionally fail-open or fail-closed. Small diagnostic, marker, cleanup, and report steps must not accidentally turn an optional optimization failure into a job failure.
- For filtered or sharded test shortcuts, verify selector/filter names against the authoritative test configuration, model shards with legitimately zero selected tests, and ensure `--passWithNoTests` or equivalent flags cannot hide a global zero-test run caused by filter drift.
- For deploy-only/redeploy paths, prove deployability with immutable manifests and artifact metadata, not only image/tag/blob existence. Treat auth, network, and non-404 storage errors differently from missing artifacts.
- For manually triggered privileged workflows, verify the trust boundary before checkout, local action execution, package install, cloud auth, registry/Kubernetes auth, or secret-bearing env. Non-default refs should fail immediately or run only an inert fixture path, and production paths should check out trusted default-branch code.
- For scheduled privileged workflows, verify empty upstream discovery, missing credentials, and transient provider silence do not become repeated hard-failure loops unless fail-closed behavior is explicitly intended and separately alerted.
- For scheduled workflows with concurrency groups, require explicit job timeouts on every recurring job, not only the most obvious collector, so a hung run cannot park the group for the platform default timeout.
- For secret/env/config checks, derive requiredness from the runtime schema or equivalent authority. Report optional/defaulted drift separately from deploy-blocking missing required values.
- For shared test infrastructure, prove crash/cancel isolation. Reused databases, caches, workers, slots, ports, and temp dirs need setup-time cleanup or generation tokens because teardown hooks do not run after OOM/SIGKILL/cancelled jobs.
- For cache-key changes, include runtime/toolchain/package-manager/workspace-manifest provenance and then inspect whether the cache is actually hit, stale, overbroad, or too expensive to restore.
- For third-party workflow actions and CLIs, verify exported outputs against the action's real success/failure semantics, logs, and documentation/source. Inspect the exact action entrypoint being invoked (`action.yml` for root vs sub-actions such as `/restore` or `/save`); output contracts can differ inside one pinned action repository. Normalize ambiguous signals once, remove unused derived outputs, and make gates, telemetry, summaries, and install/deploy guards consume the same normalized state.
- For workflow permission changes, audit every job that writes statuses, deployments, artifacts, checks, comments, packages, tags, releases, or dispatches workflows; top-level permission tightening can silently remove needed job capabilities.
- For deployment metrics and notifications, align terminology with the side-effect boundary: a rollout that already reached serving traffic but fails a later integrity gate is not the same as a rollout failure. DORA events, Slack copy, Sentry/index continuations, and follow-up jobs must use that split state consistently.
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
