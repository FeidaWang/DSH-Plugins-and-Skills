# Dependency and realm contracts

## Formal condition matrix

Source: *A Programming Paradigm for Spatiotemporal Composability* by Yifan Shi, Wei Zhang, and Tianyi Cui.

| Result | What it establishes | Required premises |
| --- | --- | --- |
| Theorem 63, pages 45-46 | A consumer activates only with provided dependencies; the provider episode begins first and, if it closes, closes after the consumer; the committed binding remains installed and stable during the consumer episode. | The guarded lifecycle calculus and its preservation invariant, including provider uniqueness. |
| Theorem 64, pages 46-47 | A transition either completes against one committed resolution or diverts/raises and recovers its contribution. | Inertial lifecycle and the recovery conditions used by Corollary 62. |
| Theorem 66, pages 47-48 | No lifecycle deadlock and termination in a quiescent state. | Acyclic provider precedence, finite fiber names, bounded iterator length, and a sequence of lifecycle rules. Finiteness is assumed, not derived. |
| Lemma 70, page 50 | The support set equals active fibers at quiescence. | Acyclic precedence, quiescence, no failed fiber, and total provision. |
| Theorem 73, pages 52-53 | Canonical form and confluence for the same orchestration inputs. | Quiescence, no failed fiber, pairwise-independent steps, total provision, and the well-founded support relation. |

Failure is explicitly excluded from confluence. The theorem compares final states for the same orchestration inputs; it does not erase observable emissions produced along the path.

## Graph semantics

Use these distinct relations:

- `requires(fiber, key)`: the declared coeffect specification;
- `mayProvide(component, key)`: the component's declared provision set;
- `installed(fiber, key, realm)`: the active fiber actually installed the binding;
- `committed(consumer, key, provider)`: the provider identity the consumer activated against;
- `parent(parentFiber, childFiber)`: the child was registered by the parent's activation;
- `precedes(provider, consumer)`: the provider may supply a declared key of the consumer;
- `supports(a, b)`: provider precedence or parent registration, used by the confluence support relation.

Do not infer `installed` from `mayProvide`. Total provision is precisely the extra condition that equates them after successful activation.

## Realm and interception rules

The paper's implementation correspondence appears on pages 55 and 57-61:

- a key first resolves to a realm identity and then to that realm's store entry;
- isolation redirects a key to another realm without modifying the parent;
- interception changes metadata used when accessing a binding, not which provider satisfies it;
- a fiber's committed view records provider identity, not only value equality;
- an unloading provider stops satisfying new targets before its stored binding is removed;
- a dependent reads from its committed view during cleanup, so the old provider must remain available until the dependent drains.

The Loader's managed realm reassignment is described on pages 63-64. Audit both binding movement and notification selection; a realm symbol shared by several fibers does not by itself identify which fiber owns a binding.

## DSH/Cordis evidence map

At the audited DSH base commit `47f943859bef60e4160492346772ded9b24f765a`, the fork and configured research upstream had no commit or tree difference. Re-check the current worktree.

- `vendor/cordis/src/registry.ts`: injection normalization, runtime registration, and provider uniqueness rules.
- `vendor/cordis/src/reflect.ts`: provision, service store, provider ownership, and dependency notification.
- `vendor/cordis/src/context.ts`: `get`, isolation, interception, and context derivation.
- `vendor/cordis/src/fiber.ts`: target computation, committed views, lifecycle inertia, dependent draining, parent ownership, and service access guards.
- `vendor/loader/src/config/isolate.ts`: managed realm reassignment.
- `vendor/loader/src/config/tree.ts`: configuration tree and lifecycle settlement.
- `packages/extensions/tool-cordis/tests/cordis-lifecycle.spec.ts`: direct lifecycle and ownership probes.
- `packages/boot/app-boot/tests/config-reload.spec.ts`: loader reconciliation, realm movement, and rollback behavior.

Useful discovery command:

```bash
rg -n "inject|provide\(|committed|target|isolate\(|intercept\(|notify|inertia" \
  vendor/cordis/src vendor/loader/src packages/extensions/tool-cordis/tests \
  packages/boot/app-boot/tests
```

## Validated relocation hazards

The following checks were validated against the DSH Loader and belong in every future realm audit:

- A provider implementation's cleanup key must move with its reflection-store binding. A disposer that retains the source symbol leaves a ghost binding owned by a disposed fiber and prevents clean reactivation.
- An occupied destination has two different meanings. A provider-owned binding move must reject before context mutation, while a consumer-only move between populated realms is valid because it transfers no implementation.
- Starting notified fibers is not transaction settlement. Await every selected fiber, preserve a single failure or aggregate several failures, and route failure through the entry update's compensation boundary.
- A committed binding compares provider identity, not exposed value. Equal-valued replacement by a different provider fiber must still refresh the consumer.
- A mutual dependency cycle may settle as inactive `PENDING` fibers. It is evidence that the acyclic-precedence premise is absent, not evidence of unconditional runtime progress.

The executable ownership probes live in `packages/extensions/tool-cordis/tests/cordis-lifecycle.spec.ts`; Loader movement, collision, and rollback probes live in `packages/boot/app-boot/tests/config-reload.spec.ts`. The corresponding implementation points are `vendor/cordis/src/reflect.ts` and `vendor/loader/src/config/isolate.ts`.

## Additional limits

- Section 6.3, pages 69-70: proxy-mediated declarations resemble capability checks, but malicious code with direct host access still requires an external sandbox.
- Section 6.5, page 71: a mutual dependency cycle normally leaves participants inactive. Decomposition can remove the cycle but increases component granularity.
- Section 6.6, pages 72-73: key identity alone does not establish interface or version compatibility and does not prevent independent key collisions. Namespacing, peer constraints, or structural checks are separate mechanisms.

Keep these limits visible in every architecture recommendation.
