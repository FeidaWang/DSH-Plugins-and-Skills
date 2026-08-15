# DSH Plugins and Skills

Evidence-driven plugins and reusable skills for auditing Cordis lifecycle behavior.

## Included plugin

### `dsh-spatiotemporal-audit`

A Codex plugin containing three focused skills:

- `audit-cordis-effects`: audit effect ownership, cleanup boundaries, and rollback ordering.
- `audit-cordis-dependencies`: map providers, injectors, realms, and dependency constraints.
- `verify-cordis-dynamic-change`: verify reload, configuration change, and migration paths as lifecycle transactions.

The workflows distinguish conditional guarantees from universal guarantees and require source evidence plus deterministic tests for high-confidence conclusions.

## Use

Add the repository as a Codex marketplace, then install the plugin:

```bash
codex plugin marketplace add FeidaWang/DSH-Plugins-and-Skills
codex plugin add dsh-spatiotemporal-audit@dsh-plugins-and-skills
```

The plugin lives at `plugins/dsh-spatiotemporal-audit/`. Its skills can also be reused independently from the nested `skills/` directory.

## License

Apache-2.0. See [LICENSE](LICENSE).
