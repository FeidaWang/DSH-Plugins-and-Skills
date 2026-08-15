---
name: verify-cordis-dynamic-change
description: Verify Cordis and Cordis-like HMR, configuration reconciliation, plugin update, restart, and rollback paths as awaited lifecycle transactions. Use when reviewing module cache invalidation, registry replacement, async activation or teardown, success events, compensation, overlapping file changes, multi-plugin reloads, or claims that a live update is atomic.
---

# Verify Cordis Dynamic Change

Treat a dynamic change as a protocol across module caches, configuration, registry identity, fiber lifecycle, service ownership, and observable events. Read [dynamic-change-contracts.md](references/dynamic-change-contracts.md) before accepting a transaction or confluence claim.

## Build the transaction ledger

List every state surface that can diverge:

- declarative configuration and entry pointers;
- ESM and CommonJS caches;
- import/dependency classification state;
- plugin registry identity and runtime fiber set;
- fiber state, committed view, target, and in-flight inertia;
- provided services and other owned resources;
- success, failure, and change events visible to observers.

For each phase, record:

| Phase | Preconditions | Mutation | Completion signal awaited | Compensation | Failure probe |
| --- | --- | --- | --- | --- | --- |

An API call returning is not a completion signal unless its contract says the lifecycle has settled.

## Transaction workflow

1. Read repository instructions and identify the authoritative configuration, loader, module cache, registry, fiber lifecycle, and watcher code. Do not infer implementation from a paper's pseudocode.
2. Define the intended atomic group: one entry, all stale entries, a configuration subtree, or another explicit unit. Identify which observations are permitted between phases.
3. Preflight before mutating live state: parse and validate configuration, classify dependencies, invalidate or version caches with backups, and import every candidate that can be imported without retiring the active graph.
4. Snapshot the last active generation, including cache records, old plugin identities, source fibers, entry pointers, and configuration data needed for compensation.
5. Retire the old registry identities and await every old fiber's lifecycle inertia. Do not start replacements while old cleanup still owns ports, services, listeners, or persistent hooks.
6. Start all candidates, retain their fibers, and await activation settlement. A created fiber is not an active fiber; surface startup failure through the lifecycle's error channel.
7. Commit only after the entire atomic group is active. Update authoritative state and publish success exactly once after commit.
8. On failure, drain every candidate that started, restore cache and configuration snapshots, reactivate every old plugin that was retired, await compensation, and suppress success. Report compensation failures separately.
9. Serialize overlapping changes with snapshot-and-dirty semantics. A change arriving during import, teardown, activation, or compensation must enter a later pass and must not be erased by the current pass.
10. State the limit: plugin-owned state can be compensated; external emissions remain observable unless explicitly withheld or compensated.

## Required failure matrix

Exercise the applicable rows:

| Boundary | Injected failure or race | Required observation |
| --- | --- | --- |
| Import | syntax error or top-level rejection | Active generation stays active; no success event. |
| Old teardown | delayed or rejecting disposer | No candidate starts before teardown settles; failure is contained or compensated by contract. |
| Candidate activation | synchronous throw and post-await rejection | Candidate becomes failed, no success event, last active generation returns. |
| Multi-plugin activation | one succeeds, another fails | The whole declared group returns to one coherent generation. |
| Cache gap | edit arrives while accepted URL is absent | Edit remains classified as reload work. |
| Overlapping edits | generation N+1 arrives while N reloads | Transactions do not overlap; newest generation is active last. |
| Compensation | old generation fails to reactivate | Failure is visible and never mislabeled as success. |
| Shutdown | service disposes during an active reload | Watchers close and active transaction work drains. |

## Deterministic test design

- Drive real public entry points when practical: real loader boot, watcher, configuration tree, file writes, and lifecycle events.
- Use unique temporary modules and global keys; canonicalize paths with `realpath` before comparing file URLs.
- Use promise resolvers or explicit events to pause teardown, activation, or import evaluation.
- Observe watcher delivery through a supported cache lookup, event, or lifecycle state before releasing a gate.
- Make every `finally` release outstanding latches before disposing the root context.
- Prefer polling watcher configuration for cross-platform tests. Do not rely on a fixed sleep as the correctness assertion.
- Avoid fake timers or private-method invocation when Node loader cache and watcher behavior are the subject of the test.

## Verdict

Return a phase ledger followed by one verdict:

- `transaction demonstrated`;
- `transaction demonstrated within named boundary`;
- `compensating update only`;
- `half-applied state reachable`;
- `not demonstrated`.

Name the exact missing wait, snapshot, serialization edge, or compensation step. Do not expose hidden chain-of-thought; provide source locations, test traces, and the minimal counterexample instead.
