# Effect and boundary contracts

## Formal guardrails

Source: *A Programming Paradigm for Spatiotemporal Composability* by Yifan Shi, Wei Zhang, and Tianyi Cui.

- Theorem 16, page 15: reversing effects in strict application-reverse order recovers the preceding states without an independence premise. This is a local LIFO result.
- Definition 19 and Theorem 20, pages 16-17: withdrawing one effect while foreign effects remain requires pairwise independence. Forward maps and yielded inverses must commute, and one effect must not change which inverse or continuation the other yields.
- Corollary 21, page 17: arbitrary inverse order reaches the initial state only for a pairwise-independent family.
- Theorem 61 and Corollary 62, pages 44-45: system-level recovery through interleaved fibers again assumes pairwise-independent steps.
- Section 5.1.1, page 56: Cordis does not verify that a callback's inverse actually recovers its effect. The inverse witness remains an obligation on the component author.
- Section 6.1, pages 67-68: the guarantee is location-specific and stops at the system boundary. Acquisition can be tracked; emission generally cannot. External output needs withholding or an application-defined compensation, whose algebra must be re-established.

Never infer any of the following:

- that every callback called `dispose` is a valid inverse;
- that LIFO call order implies LIFO asynchronous completion;
- that local reversal proves arbitrary withdrawal among components;
- that process, network, filesystem, payment, or user-visible output is automatically reversible;
- that observational equivalence covers a representation or emission not named by the observer model.

## Resource ledger

Record one row for every durable change:

| Acquisition or mutation | Owner | Inverse or compensation | Registration atomic? | Settlement awaited? | Boundary | Evidence |
| --- | --- | --- | --- | --- | --- | --- |

An ownership handoff is atomic only if failure cannot leave a published resource without a registered inverse. Inspect the order of publication and owner bookkeeping, including partial setup and invalid lifecycle states.

## Ordering model

Keep these relations separate:

1. setup registration order;
2. disposer invocation order;
3. asynchronous cleanup start order;
4. asynchronous cleanup completion order;
5. provider visibility during dependent cleanup;
6. parent-child retirement order.

A test that records only final completion order cannot prove when a provider disappeared. Observe the service store or a dependency read while another cleanup is intentionally blocked.

## DSH/Cordis evidence map

At the audited DSH base commit `47f943859bef60e4160492346772ded9b24f765a`, the GitHub fork matched its configured research upstream. Re-read the current worktree before applying these observations.

- `vendor/cordis/src/fiber.ts`: `Fiber.effect`, lifecycle inertia, `_reload`, `_unload`, effect ownership, nested-disposer composition, and sibling-effect draining.
- `vendor/cordis/src/reflect.ts`: provider publication, store mutation, owner bookkeeping, and withdrawal notification.
- `vendor/cordis/src/service.ts`: service provision through the reflective layer.
- `vendor/cordis/src/context.ts`: derived contexts, isolation, and interception boundaries.
- `packages/extensions/tool-cordis/tests/cordis-lifecycle.spec.ts`: direct ownership and lifecycle regressions.
- `vendor/README.md`: local departures from upstream; update it when changing vendored source.

High-value DSH probes:

- Confirm nested cleanup attempts all disposers after synchronous and asynchronous failure.
- Inspect whether separate top-level async effects begin concurrently during fiber unload; do not describe reverse registration as serialized completion unless a test proves it.
- Call provider registration from pending, failed, and unloading owners and verify that a rejected registration leaves no visible binding.
- Block a provider-consuming cleanup while unloading and assert the provider stays readable until that cleanup drains.

### Forward-test seeds from 2026-08-15

A fresh in-memory probe against the working tree produced two deterministic counterexamples. Treat these as regression seeds and re-run them before citing their status:

- Calling `provide()` from a `PENDING` fiber threw, but the raw root lookup still returned the published value. Publication therefore preceded successful owner registration.
- While one sibling cleanup was blocked, fiber unload had already withdrawn a service owned by another sibling effect. The blocked cleanup's later self-read failed, showing that reverse registration order did not serialize sibling async completion.

These observations do not contradict the demonstrated attempt-all behavior inside one nested effect; they occur at different ownership levels.

Use `rg` to locate behavior instead of relying on stale line numbers:

```bash
rg -n "effect\(|_unload|inertia|provide\(|internal/service|Promise\.all" \
  vendor/cordis/src packages/extensions/tool-cordis/tests
```

## Fix discipline

Place a fix at the ownership layer that can enforce the invariant. Preserve documented synchronous behavior where possible. When cleanup continues after failure, retain a single failure's identity and aggregate multiple failures deterministically. Verify focused tests, type checking, lint, generated API documentation, the vendor modification log, and the repository's required documentation gates.
