---
name: audit-cordis-dependencies
description: Audit Cordis and Cordis-like dependency declarations, providers, committed views, isolation realms, interception metadata, cycles, and provider-consumer lifecycle ordering. Use when reviewing inject/provide/get, ctx.isolate or ctx.intercept, service replacement, realm movement, dependency deadlocks, progress or confluence claims, and multi-plugin topology.
---

# Audit Cordis Dependencies

Build an evidence-backed graph of what components may provide, what they declare, what they actually install, and what each fiber resolved. Read [dependency-realm-contracts.md](references/dependency-realm-contracts.md) before evaluating progress, confluence, or sandbox claims.

## Workflow

1. Read repository instructions, declarations, lifecycle code, loader configuration, and focused tests. Treat prose and papers as claims to verify against the current implementation.
2. Build the graph with separate node types for fibers/components, keys, realm identities, providers, and parents. Do not collapse a key name into one global binding when isolation exists.
3. Add separate edges for:
   - declared requirement: component or fiber to key;
   - possible provision: component to key;
   - installed provision: active fiber to key in a realm;
   - committed resolution: consumer fiber to exact provider fiber;
   - parent ownership: parent fiber to registered child;
   - interception: context and key to merged access metadata.
4. Compare declarations with behavior. Record conditional provisions, optional lookups, undeclared proxy access, duplicate providers in one realm, and provider identity changes that keep equal values.
5. Trace one activation and one withdrawal through lifecycle states. Verify the provider stops satisfying new targets before cleanup, dependents drain before the provider removes bindings, and committed dependencies remain readable during dependent cleanup.
6. Audit cycles across both provider precedence and parent registration. Report a cycle as an unsatisfied topology unless the runtime demonstrates a separate cycle-breaking mechanism.
7. Evaluate theoretical claims only after checking their prerequisites. Add deterministic tests for every violated or unproven ordering edge when the user requests code changes.

## Graph output

Use a compact table for small graphs. Use Mermaid for three or more components or when realms change resolution. Label every provider edge with its realm and distinguish `possible`, `installed`, and `committed`.

Always include:

- providers per `(key, realm)`;
- consumers and whether the dependency is hard, optional, or accessed without declaration;
- the exact provider identity committed by each active consumer;
- parent-child registration edges;
- cycles and ambiguous or duplicate provisions;
- provisions declared but not installed on every successful activation.

## Claim matrix

Report a condition matrix rather than a narrative chain-of-thought:

| Claim | Required premises | Observed premises | Evidence | Result |
| --- | --- | --- | --- | --- |

Use `demonstrated`, `conditionally demonstrated`, `violated`, or `not demonstrated`. Name every missing premise. In particular, do not claim:

- progress without an acyclic precedence graph, finite fiber population, and bounded activation work;
- confluence without pairwise-independent steps, total provision, quiescence, and absence of failed fibers;
- access control for direct host-language references that bypass context mediation;
- type or version compatibility merely because two parties use the same key;
- global uniqueness when uniqueness exists only within a realm.

## Failure probes

Prefer state-driven probes with explicit barriers:

- provider appears, disappears, and is replaced by an equal-valued provider;
- provider enters unloading while a dependent cleanup is blocked;
- consumer moves between local and shared realms;
- provider moves realm while consumers remain in old and new scopes;
- declared provision is omitted under one successful configuration;
- a cycle is introduced after one participant is already active;
- provider registration fails after store publication;
- interception metadata changes without changing satisfaction;
- child registration or retirement changes the support graph.

If the user requested only analysis, stop at findings and missing tests. If they requested a fix, preserve realm identity and committed-view semantics rather than patching around the observed symptom.
