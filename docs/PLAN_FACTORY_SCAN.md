# factory — Factory Orchestration & Scan

## One-line promise
**Bootstrap the factory by extracting what "correct" looks like from the customer's existing system — schema, verify rules, coverage queries, and validation expectations — automatically.**

---

## Problem

The factory needs to know what "right" looks like before extracting a single value. The target schema, verify rules, coverage queries, and validation expectations. Defining these manually takes weeks of domain expert time.

But if the customer already has a web app hitting their database at 85% quality, everything the factory needs is already encoded in their codebase and schema. `factory scan` extracts it automatically — DDL constraints from the database, validation rules from ORM models, coverage targets from app queries, expected data shapes from API handlers.

The output is a `factory-config/` directory that feeds `twinning`, `verify`, and `assess`. The customer reviews gaps and fills in domain rules that only exist in people's heads. Everything else is derived with provenance.

---

## Non-goals

`factory scan` is NOT:
- An extractor (it doesn't process document corpora)
- A migration tool (it doesn't modify the customer's database)
- A complete replacement for domain expertise (gaps.json captures what it can't find)
- A runtime service (it runs once during onboarding, then as-needed for drift detection)

It does not produce data. It produces the rules and schema that define what correct data looks like.

---

## CLI

```
factory scan [OPTIONS]

Options:
  --repo <PATH>          Path to customer's application codebase
  --db <URL>             Database connection string (postgresql://...)
  --output <DIR>         Output directory for factory configuration (default: factory-config/)
  --lang <LANG>          Force language detection (rust, python, java). Default: auto-detect
  --json                 JSON output for scan report
```

At least one of `--repo` or `--db` is required. Both can be provided for maximum extraction coverage.

### Exit codes

`0` scan complete | `1` scan complete with gaps | `2` refusal

### Usage modes

```bash
# Scan a codebase + live database (maximum coverage)
factory scan --repo /path/to/customer-app --db postgresql://... --output factory-config/

# Scan repo only (no DB access)
factory scan --repo /path/to/customer-app --output factory-config/

# Scan DB only (no source code)
factory scan --db postgresql://... --output factory-config/
```

---

## What it extracts

### From the database

| Source | Extracts | Used for |
|--------|----------|----------|
| `pg_dump --schema-only` | Tables, columns, types, PKs, FKs, UNIQUE, CHECK, NOT NULL | Twin schema (`--schema schema.sql`) |
| `pg_catalog` / information_schema | Row counts, null rates, value distributions per column | Coverage anchors (`expected` in twin scoring) |
| Declared constraints | CHECK expressions, FK relationships | Verify rules (constraint violations = data quality issues) |

### From the codebase

Rust codebases are more statically analyzable than Python/JS — struct definitions, type signatures, and trait impls are explicit in the source. `factory scan` on a Rust repo gets higher-confidence extractions than on a dynamic language.

| Source | Extracts | Used for |
|--------|----------|----------|
| Struct definitions + `sqlx`/`diesel` models | Field types, nullability (`Option<T>`), serde attributes, validation impls | Verify rules beyond DDL constraints |
| SQL queries (`sqlx::query!`, raw SQL strings, query builders) | SELECT patterns, JOIN paths, WHERE clauses | Coverage queries — "the queries the app actually runs" |
| API route handlers (Actix, Axum, Rocket) | Endpoint -> query mapping, response structs | Expected data shapes |
| Frontend chart configs / API response types | Axis labels, expected ranges, series definitions | Chart validation assertions |
| `#[test]` functions + integration tests | Assertions about data values, edge cases | Verify rules (tests encode domain knowledge) |
| SQL migration files (sqlx migrations, refinery, etc.) | Schema evolution over time | Understanding renamed/deprecated columns |
| Computed/derived fields | `impl` blocks with calculation methods, SQL views | Verify rules for calculated values (NOI = Revenue - Expenses) |
| `enum` types + `FromStr` / `Deserialize` impls | Valid value sets for categorical fields | CHECK-equivalent constraints |

### Python/Django/SQLAlchemy support

| Source | Extracts | Used for |
|--------|----------|----------|
| Django models + validators | Field types, nullability, choices, custom validators | Verify rules |
| SQLAlchemy models + column definitions | Types, constraints, relationships | Twin schema + verify rules |
| Pydantic models + validators | Expected shapes, value constraints | API coverage targets |
| `pytest` / `unittest` assertions | Domain knowledge encoded as test expectations | Verify rules |

---

## Output

A factory configuration directory:

```
factory-config/
├── schema.sql              # DDL for twinning --schema
├── rules.json              # Verify rules for twinning --rules
├── coverage-queries.sql    # Queries the app runs (coverage targets)
├── anchors.json            # Row counts, expected ranges, declared totals
├── chart-assertions.json   # Expected visual properties for answer engine
├── scan-report.json        # What was found, confidence per extraction
└── gaps.json               # What couldn't be extracted (needs manual input)
```

### Confidence scoring

Every extracted rule carries a confidence score based on its provenance:

| Source | Confidence | Rationale |
|--------|-----------|-----------|
| DDL constraint (NOT NULL, CHECK, FK) | 1.0 | Database enforces it — it's a hard fact |
| Explicit ORM validator | 0.95 | Code enforces it — high confidence |
| Test assertion | 0.9 | Developer encoded this expectation |
| Query pattern inference | 0.8 | App queries this column — implies it matters |
| Computed field inference | 0.7 | Calculation exists — verify rule is inferred |
| Statistical inference (null rates, distributions) | 0.5 | Observed pattern — may not be intentional |

Rules below a configurable confidence threshold are placed in `gaps.json` for human review rather than `rules.json`.

### Scan report

```json
{
  "version": "factory_scan.v0",
  "sources": {
    "database": { "url": "postgresql://...", "tables": 12, "constraints": 47 },
    "codebase": { "path": "/path/to/app", "language": "rust", "files_analyzed": 234 }
  },
  "extractions": {
    "schema_rules": 47,
    "codebase_rules": 23,
    "coverage_queries": 15,
    "anchors": 8,
    "chart_assertions": 3
  },
  "confidence_distribution": {
    "1.0": 47,
    "0.9-0.99": 12,
    "0.7-0.89": 11,
    "below_0.7": 3
  },
  "gaps": {
    "total": 8,
    "categories": {
      "missing_validation": 3,
      "ambiguous_calculation": 2,
      "undocumented_constraint": 3
    }
  }
}
```

---

## Lineage

Every extraction `factory scan` makes is recorded as lineage in the data-fabric catalog — not just dumped into config files and forgotten.

```
rule: NOI_POSITIVE
  derived_from:
    - source: "pg_catalog.pg_constraint"                    # DDL CHECK
      confidence: 1.0
    - source: "customer-app/src/models/financial.rs:47"      # Rust struct validator
      confidence: 0.95
    - source: "customer-app/src/api/handlers/deals.rs:112"  # API query pattern
      confidence: 0.8
  used_by:
    - twinning --rules rules.json                           # twin enforcement
    - verify --rules rules.json                             # spine verification
    - coverage-queries.sql:23                                # coverage scoring
```

When a verify rule fires during factory processing — "NOI is null for property P-123" — you can trace the rule back through the catalog to why it exists. The rule isn't arbitrary. It's derived from the customer's own system, and the derivation chain is evidence-grade.

The catalog also tracks **gaps** — rules the customer fills in manually get a `source: "manual"` provenance marker. Over time, the ratio of auto-derived to manual rules is a quality metric: 95% auto-derived means a well-encoded domain model; 50% means tribal knowledge that should be in code.

---

## Gaps

`gaps.json` captures what scan couldn't extract — domain rules that only exist in people's heads:

```json
{
  "gaps": [
    {
      "category": "missing_validation",
      "description": "No validator found for DSCR reasonableness (expected range 0.5-3.0)",
      "suggestion": "Add verify rule: DSCR BETWEEN 0.5 AND 3.0",
      "tables_affected": ["financials"],
      "confidence": 0.0
    },
    {
      "category": "ambiguous_calculation",
      "description": "NOI calculation found in two places with different formulas",
      "sources": [
        "src/models/financial.rs:47 — revenue - expenses",
        "src/api/handlers/deals.rs:89 — revenue - expenses - reserves"
      ],
      "suggestion": "Clarify which NOI definition is canonical",
      "confidence": 0.0
    }
  ]
}
```

The customer reviews `gaps.json` and fills in the blanks. These manual additions get `source: "manual"` provenance so the system tracks what was human-provided vs auto-derived.

---

## Refusal codes

| Code | Trigger | Next step |
|------|---------|-----------|
| `E_NO_SOURCE` | Neither `--repo` nor `--db` provided | Provide at least one |
| `E_DB_CONNECT` | Can't connect to database | Check connection string |
| `E_DB_PERMISSION` | Insufficient DB permissions for schema introspection | Grant read access to pg_catalog |
| `E_REPO_NOT_FOUND` | Repository path doesn't exist | Check path |
| `E_LANG_DETECT` | Can't detect codebase language | Use `--lang` flag |
| `E_NO_SCHEMA` | Database has no tables or repo has no models | Check target system |

---

## Usage examples

```bash
# Full scan: codebase + database
factory scan --repo /path/to/customer-app --db postgresql://... --output factory-config/

# Review what was found
cat factory-config/scan-report.json | jq '.extractions'

# Review gaps that need human input
cat factory-config/gaps.json | jq '.gaps[] | .description'

# Boot a twin from scan results
twinning postgres --schema factory-config/schema.sql \
  --rules factory-config/rules.json --port 5433

# Run verify against scan-derived rules
verify data.csv --rules factory-config/rules.json --json
```

---

## Relationship to other tools

| Tool | Relationship |
|------|-------------|
| **twinning** | Consumes `schema.sql` and `rules.json` from scan output |
| **verify** | Consumes `rules.json` — scan-derived rules become data contracts |
| **assess** | Coverage anchors from scan feed assess policies |
| **benchmark** | Scan-derived expected values can seed assertion files |
| **data-fabric** | Scan lineage recorded in catalog for traceability |
| **decoding** | Scan-derived schema + rules are precode constraints for decode resolution |

---

## Implementation notes

### Candidate crates

| Need | Crate | Notes |
|------|-------|-------|
| SQL parsing | `sqlparser-rs` | Parse DDL, extract constraints |
| Rust AST parsing | `syn` + `quote` | Parse struct definitions, impls, attributes |
| Python AST parsing | `tree-sitter-python` | Parse Django/SQLAlchemy models |
| Database introspection | `sqlx` or `tokio-postgres` | Query pg_catalog |
| YAML/JSON output | `serde_json`, `serde_yaml` | Config file generation |

### Implementation scope

| Component | LOC estimate |
|-----------|-------------|
| Database introspector (pg_catalog -> schema.sql + rules) | ~1-2K |
| Rust codebase analyzer (syn AST -> rules + coverage queries) | ~2-3K |
| Python codebase analyzer (tree-sitter -> rules) | ~1-2K |
| Confidence scoring + gap detection | ~500 |
| Output generation (config directory + scan report) | ~500 |
| Lineage emission (data-fabric integration) | ~500 |
| **Total** | **~6-10K lines of Rust** |

---

## Determinism

Same codebase + same database state = same scan output. No randomness. Confidence scores are deterministic based on source type. The scan report is content-addressed for reproducibility.
