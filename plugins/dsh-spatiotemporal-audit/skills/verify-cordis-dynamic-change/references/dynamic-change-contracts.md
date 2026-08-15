# Dynamic-change contracts

## Paper claim and its executable obligation

Source: *A Programming Paradigm for Spatiotemporal Composability* by Yifan Shi, Wei Zhang, and Tianyi Cui.

Section 5.2, pages 61-66, describes declarative reconciliation and HMR in three phases: module classification, stale-entry detection, and transactional reload. Algorithm 10 backs up caches, disposes stale entries, imports replacements, and restores the previous modules after failure.

Treat that algorithm as design intent, not executable proof. Its pseudocode does not expose or await old-fiber teardown or replacement activation. In an asynchronous implementation, a `try`/`catch` around lifecycle initiation cannot catch a later activation rejection, and starting a replacement immediately after initiating disposal permits generations to overlap. A valid implementation must identify and await the lifecycle's actual settlement handles.

The paper's broader theorems remain conditional:

- Theorem 66, pages 47-48, gives progress only with acyclic provider precedence, finite fibers, and bounded activation work.
- Theorem 73, pages 52-53, gives confluence only at quiescence, with no failed fiber, pairwise-independent steps, total provision, and the same orchestration inputs.
- Section 6.1, pages 67-68, excludes external emissions from automatic rollback. Dynamic reload is therefore generally a compensating transaction, not isolation from all intermediate observations.

## Settlement questions

For every lifecycle API used by the update path, answer from code and tests:

1. Does it initiate work, return a join handle, or await stable state?
2. Can a second call join in-flight work, or is the public disposer single-shot?
3. Where is startup failure stored and rethrown?
4. When does provider availability change relative to cleanup?
5. Which object owns the authoritative entry-to-fiber pointer?
6. What signal proves compensation is active rather than merely scheduled?

Never substitute API naming for these answers.

## DSH/Cordis evidence map

At the audited DSH base commit `47f943859bef60e4160492346772ded9b24f765a`, the fork and configured research upstream matched. The current worktree may contain later hardening; inspect it rather than assuming either state.

- `vendor/hmr/src/index.ts`: watcher ingress, accepted/declined classification, cache backup, stale runtime discovery, registry replacement, success publication, and reload serialization.
- `vendor/cordis/src/registry.ts`: plugin registration and deletion; verify whether deletion returns or discards fiber disposal work.
- `vendor/cordis/src/fiber.ts`: lifecycle state, `inertia`, `await`, startup failure, unload, and public disposer semantics.
- `vendor/loader/src/config/entry.ts`: authoritative entry-to-fiber pointer and entry update lifecycle.
- `vendor/loader/src/config/tree.ts`: entry/fiber lifecycle settlement.
- `packages/boot/app-boot/tests/hmr-module-lifecycle.spec.ts`: real module-watcher transaction regressions.
- `packages/boot/app-boot/tests/hmr-config.spec.ts`: exact configuration watching and serialization.
- `packages/boot/app-boot/tests/config-reload.spec.ts`: configuration candidate validation, reconciliation, and rollback.
- `vendor/README.md`: local vendored changes and their owning tests.

The audited base exposed three high-value hypotheses that every later version should retain as regression tests:

- registry deletion initiated fiber disposal without letting HMR await it;
- HMR published success before asynchronous candidate activation could fail;
- module reloads shared mutable stashed state without an explicit serializer, so edits could overlap or be lost while caches were absent.

Useful discovery command:

```bash
rg -n "partialReload|stashed|loadCache|registry\.delete|fiber\.await|inertia|hmr/reload" \
  vendor/hmr/src vendor/cordis/src vendor/loader/src packages/boot/app-boot/tests
```

## Correct transaction shape

The robust shape is:

1. snapshot one change batch;
2. keep in-flight accepted URLs classified as reloadable;
3. back up both ESM and CommonJS cache entries;
4. import all candidates before live mutation;
5. snapshot every affected old runtime and fiber;
6. remove old registry identities and await old lifecycle quiescence;
7. start and await every candidate;
8. publish success only after all candidates are active;
9. on any failure, drain candidates, restore caches, restore all removed plugins, and await them;
10. process a dirty follow-up batch after the transaction settles.

When a public disposer cannot join already-running cleanup, await the lifecycle's documented inertia or stable-state handle after initiating disposal. Verify that this does not trigger a different self-disposal path in the loader.

## Validated DSH transaction ledger

| Phase | Preconditions | Mutation | Completion signal awaited | Compensation and failure probe |
| --- | --- | --- | --- | --- |
| Ingress and classification | Watched change belongs to `loadCache` or the in-flight `refreshing` set. | Add the URL to `stashed`; set module dirty state. | One serialized `moduleTask`. | An edit delivered during a cache gap stays classified and enters a later dirty pass. |
| Preflight | Accepted dependency closure and affected plugin entries are known. | Back up and evict ESM and CommonJS caches; import every candidate. | Every candidate import or module evaluation. | Import rejection restores both caches without retiring the active generation or publishing success. |
| Retire | Candidate imports succeeded and old runtime/fiber snapshots exist. | Delete old registry identities. | Every old fiber's lifecycle inertia. | A blocked disposer prevents candidate start; Cordis contains cleanup failures according to its unload contract. |
| Activate | The complete old generation is quiescent. | Register candidate fibers and update their entry pointers. | Every candidate `fiber.await()`. | One candidate failure drains all started candidates and enters compensation for the whole affected group. |
| Commit | Every candidate in the group is active. | Retain candidate caches, registry identities, and entry pointers. | No additional lifecycle work remains. | Emit `hmr/reload` exactly once after commit. |
| Compensate | Candidate activation or teardown transaction failed. | Delete candidates, restore caches, and reactivate every removed old plugin. | Candidate drain plus every restored old `fiber.await()`. | Compensation failure remains visible as a failed entry fiber and never publishes success. |
| Shutdown | HMR has begun stopping. | Reject new module requests, close watchers, and drain tracked config/module tasks. | Active `moduleTask` and config refresh tasks. | A reload blocked in old teardown keeps service disposal pending until the transaction settles. |

Loader config reconciliation uses the same settlement rule. A failed in-place config update restores the previous raw config through `Fiber.update()`; a non-active update must return `fiber.await()` so `Entry.update()` cannot reject while compensation activation is still in flight. Stable missing dependencies may settle as `PENDING` rather than waiting for a future provider.

The executable matrix lives in `packages/boot/app-boot/tests/hmr-module-lifecycle.spec.ts` and `packages/boot/app-boot/tests/config-reload.spec.ts`. It covers import evaluation rejection, delayed teardown, synchronous and post-await activation failure, multi-plugin rollback, cache gaps, overlapping edits, compensation failure, shutdown drain, blocked config compensation, and failed config restoration.

Verdict for the audited DSH implementation: `transaction demonstrated within named boundary`. The boundary includes managed module caches, Loader entry pointers, registry identities, Cordis fibers, services, and HMR success publication for the affected change group. It excludes arbitrary external emissions and direct host state, so the implementation remains compensating rather than fully isolated.

## Test barriers

Use observable barriers instead of sleeps:

- `internal/status` or an equivalent state event for failed/active fibers;
- an old cleanup's `started` and `release` promises for teardown ordering;
- a candidate apply promise for activation timing;
- a cache lookup or watcher event for delivery of an overlapping edit;
- the public success event count for commit timing;
- entry fiber state plus application-owned generation markers for final state.

Fixed-duration waits may remain as delivery retry intervals, but never as the assertion that no overlapping transaction occurred.
