# Symbol-Index Guide

`{PRODUCT_ROOT}/planning-mds/knowledge-graph/symbol-index.yaml` is the
symbol-level layer of the knowledge graph. It binds individual methods,
classes, functions, and properties to the canonical nodes already declared in
`code-index.yaml`, so retrieval can jump straight to a definition instead of
loading a whole file.

Symbol-index is **not** authoritative. Raw source files remain the source of
truth (per `solution-ontology.yaml.authority.precedence`). The symbol layer is
a retrieval aid that turns "open the file and read everything" into "look up
the symbol, get its callers/callees, edit narrowly."

---

## When to use it

- Before editing a bound method body — call
  `python3 {PRODUCT_ROOT}/scripts/kg/lookup.py --symbol <name>` to get the
  symbol record plus its caller/callee neighbourhood.
- When triaging a regression — `blast.py --symbol <name>` or
  `blast.py symbol:<id>` walks one hop of call edges and reports the
  canonical nodes and files reached.
- When reviewing a diff that adds, renames, or removes a class, function,
  method, or property in a bound file — re-run `validate.py
  --regenerate-symbols --check-symbols` so the layer stays in sync.

---

## Schema

```yaml
version: 0
generated_at: '2026-05-10T00:00:00+00:00'
summary:
  total_symbols: 1234
  by_language:
    csharp: { files: 100, parsed: 100, cached: 0 }
    typescript: { files: 40, parsed: 40, cached: 0 }
    python: { files: 0, parsed: 0, cached: 0 }
  disambiguated_ids: 4
symbols:
  - id: symbol:entity-customer:customer-service.cancel-async
    node: entity:customer
    kind: method
    name: CancelAsync
    file: backend/src/Customers/Services/CustomerService.cs
    line: 142
    signature: 'public async Task<Result> CancelAsync(Guid id, CancellationToken ct)'
    visibility: public
    language: csharp
    container: CustomerService
    callers:
      - symbol:entity-customer:customer-endpoints.cancel
    callees:
      - symbol:entity-customer:customer-repository.update-async
      - symbol:entity-customer:i-customer-repository.update-async

  - id: symbol:entity-order:order-form.submit
    node: entity:order
    kind: function
    name: submit
    file: frontend/src/features/orders/OrderForm.tsx
    line: 84
    signature: 'function submit(values: OrderValues): Promise<void>'
    visibility: local
    language: typescript
    container: null
    callers: []
    callees:
      - symbol:entity-order:order-form.validate
```

### Field reference

| Field | Type | Description |
|---|---|---|
| `id` | string | Stable symbol id: `symbol:<node-slug>:<container-or-file-stem-slug>.<name-slug>`. Collisions are disambiguated with `-2`, `-3`, … suffixes. |
| `node` | string | Canonical node id from `canonical-nodes.yaml` or `mapping_nodes`. |
| `kind` | enum | `class \| record \| struct \| interface \| enum \| delegate \| method \| function \| property \| constructor \| type` |
| `name` | string | Source-level identifier (e.g. `CancelAsync`, `submit`). |
| `container` | string \| null | Owning type for members (e.g. `CustomerService`); `null` for top-level declarations. |
| `file` | string | Repo-relative file path. |
| `line` | integer | 1-based line number of the declaration. |
| `signature` | string | First line of the declaration, attributes stripped. |
| `visibility` | enum | `public \| internal \| protected \| private` (C#); `export \| local \| public` (TS); `public \| private` (Python). |
| `language` | enum | `csharp \| typescript \| python` |
| `callers` | string[] | Symbol ids that invoke this symbol (best-effort, name-matched within node). |
| `callees` | string[] | Symbol ids this symbol invokes. |

Callers/callees are resolved by matching invoked names against other symbols
on the **same canonical node**. Over-linking is acceptable — the layer is a
routing aid, not a static-analysis report.

---

## Regeneration

```bash
# Full regeneration (uses .kg-state/symbols-cache.json for incremental parsing)
python3 {PRODUCT_ROOT}/scripts/kg/symbols.py

# Force a full re-parse (ignore cache)
python3 {PRODUCT_ROOT}/scripts/kg/symbols.py --force

# Restrict to a single canonical node
python3 {PRODUCT_ROOT}/scripts/kg/symbols.py --node entity:customer

# Restrict to a single language
python3 {PRODUCT_ROOT}/scripts/kg/symbols.py --language typescript

# Regenerate as part of validation (delegates to symbols.py)
python3 {PRODUCT_ROOT}/scripts/kg/validate.py --regenerate-symbols --check-symbols
```

### Cadence

- After any design session that adds aggregate methods, service operations,
  endpoints, or React components on a bound canonical node.
- As part of the build action closeout (Step 6 KG validation).
- Before opening a code review on a feature branch.

---

## Supported languages

| Extension | Extractor | Where |
|---|---|---|
| `.py` | Python stdlib `ast` | inline in `scripts/kg/symbols.py` |
| `.ts`, `.tsx` | ts-morph (TypeScript compiler API) | `scripts/kg/ts-symbols/` (Node subprocess) |
| `.cs` | Roslyn (`Microsoft.CodeAnalysis.CSharp`) | `scripts/kg/csharp-symbols/` (.NET subprocess) |

The Node and .NET extractors must be installed before they will produce
symbols:

```bash
# Once per checkout (or after dependency changes)
(cd {PRODUCT_ROOT}/scripts/kg/ts-symbols && npm install)
(cd {PRODUCT_ROOT}/scripts/kg/csharp-symbols && dotnet build --configuration Release)
```

`symbols.py` detects missing extractors and skips those languages with a
warning to stderr — the rest of the pipeline still works.

Adding a new language is a matter of writing a new `BaseExtractor` subclass
in `symbols.py` (or a parallel subprocess tool) and mapping its file
extensions in `LANGUAGE_BY_EXT`.

---

## How `code-index.yaml` and `symbol-index.yaml` compose

| Layer | Granularity | Generated by | Authoritative? |
|---|---|---|---|
| `canonical-nodes.yaml` | Domain concept (entity, workflow, endpoint…) | hand-curated | No (raw docs win) |
| `code-index.yaml` | File / glob | hand-curated | No (raw source wins) |
| `symbol-index.yaml` | Method / class / function / property | `symbols.py` from `code-index.yaml` | No (raw source wins) |

`symbol-index.yaml` walks **only** files declared in `code-index.yaml`. The
product owns curation of `code-index.yaml`; the framework cannot inject new
file paths. To bring a new directory under symbol-layer coverage, add a
binding (or extend an existing one) in `code-index.yaml`, then regenerate.

---

## Telemetry

`symbols.py` and `lookup.py --symbol` emit JSONL telemetry events with the
same shape as the other KG tools (see `eval.py`). Key fields:

- `tool` — `symbols`, `lookup`, `hint`, or `blast`
- `nodes_returned` / `nodes_count` — canonical nodes touched
- `symbols_returned` / `symbols_count` — symbol ids in the response
- `confidence_band` — `high` (clean match) / `low` (empty) / `ambiguous`
  (collisions / disambiguated ids)
- `tokens_estimated` — best-effort estimate so cost is comparable across
  tools.

Use `eval.py` to score retrieval quality against scenario fixtures.
