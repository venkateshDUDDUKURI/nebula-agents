# Dead-Code Review Guide

The knowledge graph answers "what's covered" through canonical nodes, feature
mappings, code-index bindings, and the symbol layer. This guide covers the
inverse question: **what has drifted, or is dead?** Two complementary checks
close that loop:

- **Ontology orphans** — canonical nodes that nothing else references and that
  have no code-index binding. Surfaced by
  `python3 {PRODUCT_ROOT}/scripts/kg/validate.py --check-orphans`.
- **Code dead-code candidates** — bound symbols not reachable from any
  declared entry point. Surfaced by
  `python3 {PRODUCT_ROOT}/scripts/kg/dead-code.py`.

Both are routing aids for release-readiness cleanup. Per
`solution-ontology.yaml.authority.precedence`, raw source files remain
authoritative — these reports nominate cleanup candidates; humans confirm
removal.

---

## When to run

| Cadence | Owner | Command |
|---|---|---|
| Release-readiness checkpoint | Architect | `validate.py --check-orphans` + `dead-code.py --safe-only` |
| Feature close | Backend / frontend / ai engineer | `dead-code.py --node <touched-node>` to confirm no new orphans introduced |
| ADR creation | Architect | `validate.py --check-orphans` — confirm the new ADR has at least one rationale link, otherwise it ships as an orphan |
| Before deleting code | Anyone | `dead-code.py --min-confidence 0.85 --node <node>` to confirm the deletion candidate is unreached |

`kg_orphan_check` in `agents/templates/lifecycle-stage-template.yaml` wires
the orphan command into the release-readiness gate.

---

## Ontology orphans

A canonical node is an orphan when **none** of the following hold:

1. A feature or story references it via any `REF_FIELDS` edge (`affects`,
   `governed_by`, `uses_schema`, `uses_api_contract`, `depends_on`,
   `restricted_to_role`, `enforced_by_policy`, `workflow_states`,
   `validated_by`, `supersedes`) or via the `feature` parent on a story.
2. Another canonical node references it via `related_nodes`, `allowed_roles`,
   `rationale.adr`, or workflow `transitions_to`.
3. A `code-index.yaml` binding exists with the node's id.

### Default exemptions

| Kind | Why exempt |
|---|---|
| `workflow_state` | Rolls up to its parent workflow; validator already checks state cohesion |
| `glossary_term` | Vocabulary anchor — value is being referenced from prose, not from edges |

Extend exemptions per product via `--orphan-exempt-kind <kind>` (repeatable).
Record any product-specific exemption in a product ADR so future architects
can see the reasoning.

### Severity

By default orphan findings are warnings; the validator still passes overall.
Promote them to errors with `--orphans-as-errors` when wiring a hard gate at
release readiness. The framework's
[lifecycle-stage-template.yaml](../../templates/lifecycle-stage-template.yaml)
ships `kg_orphan_check` as a release-readiness gate, not an
implementation-stage gate, so day-to-day development is not blocked.

### Resolution paths

For each orphan, pick one:

| Action | When |
|---|---|
| Add a feature mapping | Node represents real domain semantics that some feature uses, but the mapping has not been backfilled |
| Add a code-index binding | Node has implementation but `code-index.yaml` does not point to it yet |
| Remove the node from `canonical-nodes.yaml` | Node was added speculatively and no feature picked it up; remove and let the next feature reintroduce it if needed |
| Add a rationale link (ADRs only) | ADR exists but no canonical node cites it; the ADR is the orphan — add a `rationale:` entry on the node it actually governs |
| Record an exemption | The orphan is intentional (e.g., placeholder for a not-yet-built feature). Document why in a product ADR and add the kind to the gate's `--orphan-exempt-kind` list |

---

## Code dead-code candidates

`dead-code.py` walks the call graph in `symbol-index.yaml` starting from
**declared entry points**:

- **Bound entry-point nodes**: any symbol whose canonical node has `_kind` in
  `{endpoint, ui_route}` (override with `--entry-kind`).
- **Framework name suffixes**: `*Handler`, `*Listener`, `*Subscriber`,
  `*Consumer`, `*Plugin`, `*Adapter`, `*Middleware`, `*Filter`,
  `*Interceptor`. These are invoked by DI containers / pipelines / message
  buses; no caller appears in the symbol index.
- **Framework file patterns**: hosted services (`*HostedService.cs`,
  `*BackgroundService.cs`, `*Worker.cs`), endpoint registrations
  (`*Endpoints.cs`, `*Controller.cs`), bootstrappers (`Program.cs`,
  `Startup.cs`, `DependencyInjection.cs`), EF Core configurations
  (`*Configuration.cs`, `Configurations/`), DI extensions
  (`*Extensions.cs`, `*Module.cs`, `*Registration.cs`), seeders
  (`*SeedData.cs`, `*Seeder.cs`), migrations (`Migrations/`), and tests
  (`*Tests.cs`, `*.test.{t,j}sx?`, `*.spec.{t,j}sx?`, `test_*.py`,
  `*_test.py`, `tests/**`).

Reachability is BFS over the `callees` graph from those seeds.

### Confidence model

Each unreached symbol gets a confidence score in `[0, 1]`:

```
baseline                                              0.6   (unreached)
+0.2  no callers anywhere in the symbol index
+0.1  visibility is public
−0.2  visibility is private/internal/protected
−0.2  node has no feature-mapping refs (ontology orphan — already covered)
```

Score clamps to `[0, 1]`.

| Band | Score | Meaning |
|---|---|---|
| `weak` | 0.5–0.69 | Likely an artifact of same-node-only call resolution. Inspect, but do not remove without source-level confirmation |
| `default` | 0.7–0.84 | Plausible candidate. Default `--min-confidence`. Triage at feature close |
| `safe` | 0.85–1.0 | High-confidence candidate. `--safe-only` threshold. Default architect gate at release readiness |

The model biases toward false negatives (missing a dead symbol) rather than
false positives (proposing a removal that breaks code). When in doubt, the
score is lower.

### Skipped symbol kinds

`class`, `record`, `struct`, `interface`, `type`, `enum`, `delegate`,
`property`, and `constructor` are never flagged — they are declarations
rather than callable bodies. Method/function symbols on the same type carry
the real reachability signal.

### Known limitations

Call edges in `symbol-index.yaml` are resolved by name within the **same
canonical node** (see [symbol-index-guide.md](symbol-index-guide.md) §"How
`code-index.yaml` and `symbol-index.yaml` compose"). Cross-node calls (an
endpoint on `entity:order` invoking a service method on `entity:customer`)
are invisible to the reachability walk. The confidence dampers compensate by
lowering the score when a symbol's node has feature-mapping refs that could
carry an untracked cross-node flow, but some genuine false positives remain.
The triage rubric below explains how to recognize them.

---

## Triage rubric

For each candidate, ask in order:

1. **Is the symbol name a registration helper or framework callback the
   pattern list missed?** (e.g., `RegisterRoutes`, `OnModelCreating`,
   `BuildContainer`). If yes, extend `FRAMEWORK_ENTRY_NAME_SUFFIXES` or
   `FRAMEWORK_ENTRY_FILE_PATTERNS` in `scripts/kg/symbols.py` and re-run.
2. **Does an endpoint or test in a different canonical node call this
   symbol?** (Grep the name; verify by file path.) If yes, this is a
   same-node-resolution false positive — record the symbol in a per-product
   ignore list referenced from a product ADR.
3. **Is the symbol genuinely unused?** Delete it. Re-run
   `python3 {PRODUCT_ROOT}/scripts/kg/symbols.py` + `dead-code.py` to confirm
   the report shrinks.

---

## Examples (customers / orders)

| Finding | Action |
|---|---|
| `policy_rule:order-export` is an orphan: no feature `enforced_by_policy` references it, and no code-index binding exists | Remove the node — premature. The next feature that needs export policy can re-add it. |
| `adr:042` is an orphan: no canonical node lists it under `rationale.adr` | Add a `rationale:` entry on the node the ADR actually governs (e.g., `entity:order`). The ADR is the source of truth; the rationale link is what makes it discoverable. |
| `entity:order-attachment` is an orphan but a planned feature folder under `planning-mds/features/F0044-order-attachments/` exists | Add the feature to `feature-mappings.yaml.coverage.excluded_features` until the feature is implemented; the orphan disappears. |
| `dead-code.py` reports `CustomerService.GetByExternalIdAsync` at confidence 0.9; grep shows it called from `OrderImportEndpoints.cs` (different node) | Same-node-resolution false positive. Add a per-product ignore entry; consider whether the cross-node link warrants a code-index binding on `entity:customer` that also covers `OrderImportEndpoints.cs`. |
| `dead-code.py` reports `LegacyOrderReceiptFormatter.Format` at confidence 0.9; grep shows zero references | Genuine dead code. Delete the file. Re-run `symbols.py`. |

---

## Telemetry

Both CLIs emit JSONL telemetry events via `kg_common.emit_telemetry`. Key
fields:

| Field | `validate.py --check-orphans` | `dead-code.py` |
|---|---|---|
| `tool` | `validate` | `dead-code` |
| `orphan_count` / `candidates_count` | yes | yes |
| `nodes_returned` / `nodes_count` | orphaned node ids | nodes hosting candidates |
| `confidence_band` | `high` if any orphans, else `low` | same |
| `tokens_estimated` | yes | yes |

`eval.py` scores these against scenario fixtures the same way it scores
`lookup.py` and `hint.py`.

---

## Relationship to other gates

| Gate | Relationship |
|---|---|
| `symbol_index_sync` | Must pass before `kg_orphan_check` is meaningful — orphan and dead-code reports assume the symbol layer is in sync |
| `inline_decisions_check` | Independent; a decision marker on an orphan node is its own signal that the node was intended to be reached |
| `boundary_genericness` | Independent; orphan detection is framework-generic and never names solution concepts |
| Risk scoring (`risk.py`) | Independent; a node with high `kg.risk` and an orphan child symbol is a stronger signal than either in isolation, but no automated rule combines them |
