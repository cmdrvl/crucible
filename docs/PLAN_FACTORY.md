# The Data Factory — System Decomposition Engine

## One-line promise
**Decompose complex data systems — document corpora or legacy databases — into provably correct data products, using tournament-style competition scored by the spine.**

---

## Problem

Two versions of the same problem:

**Document corpus decomposition.** A million files from trustee sites, data rooms, and servicer portals. Excel, PDF, CSV. Messy, inconsistent, spanning years of format changes. The goal: verified, structured data with every value traceable to a source cell. This is the structured finance use case — CMBS, CLO, ABS.

**Legacy system decomposition.** A 20-year-old 1000-table Oracle database inside a bank. Five failed retirement attempts over 12 years. Stored procedures calling stored procedures. Views on views. Jobs nobody remembers. Applications connecting from unknown sources. The goal: decompose the monolith into purpose-built data products and marts, with provable migration at every step.

Both problems share the same structure:
1. You don't fully understand what you have
2. Multiple sources tell competing stories about the same data
3. You need to prove the output is correct, not just believe it
4. Quality compounds — each round of evidence makes the next round's bar higher

The factory solves both. The spine provides the proof infrastructure. The factory adds: scanning, claims, decoding, twinning, and tournament scoring.

---

## The spine underneath

The spine tools are the universal proof layer. They work anywhere there is data:

| Tool | Role in the factory |
|------|-------------------|
| `vacuum` + `hash` + `fingerprint` | Inventory and identify every file |
| `profile` + `shape` | Understand structure, detect compatibility |
| `lock` | Content-address and seal inputs |
| `verify` | Check data contracts (rules the data must satisfy) |
| `compare` | Exhaustive diff (every cell that changed) |
| `rvl` | Materiality analysis (what changes matter) |
| `benchmark` | Score extraction quality against gold set |
| `assess` | Policy-based go/no-go decisions |
| `pack` | Seal evidence (tamper-evident, content-addressed) |
| `canon` | Canonical identity registries |

The factory doesn't replace the spine. It orchestrates it — scanning to discover what needs proving, claims to capture competing evidence, decoding to resolve conflicts, twins to test against, and tournaments to pick winners.

---

## Two modes

### Mode 1: Document corpus → structured data

**Input:** Messy document corpus (PDFs, Excel, CSV from trustees, servicers, data rooms).

**Output:** Verified structured data in a target database. Every record traceable to a source file through a deterministic chain.

**The loop:**
1. **Scan** the corpus — `vacuum` | `hash` | `fingerprint` to inventory and classify
2. **Extract** — parsers emit claims (value + evidence + confidence) for each recognized document
3. **Decode** — resolve competing claims into canonical rows using declared policy
4. **Score** — `benchmark` against gold set, `verify` against rules, `assess` for go/no-go
5. **Seal** — `pack` the evidence
6. **Harvest** — `benchmark harvest` grows the gold set from approved results
7. Go to 2. The bar is higher every round.

The key insight: extraction is a tournament. Run N parser configurations against the same documents. Score them all against the same gold set. Best one wins. The gold set grows. Next round the bar is higher.

### Mode 2: Legacy database → data products

**Input:** Monolithic legacy database (Oracle, SQL Server, DB2) + every application, script, report, and job that touches it.

**Output:** Purpose-built data products and marts, each provably equivalent to the subset of the legacy system it replaces. Sealed evidence for auditors and regulators.

**The loop:**
1. **Scan** the legacy system — schema, stored procs, views, jobs, audit logs, connection patterns
2. **Scan** every application codebase — Java, Python, COBOL, Excel macros, Crystal Reports, whatever touches Oracle
3. **Claim** — each scan produces claims about what the database is, what it does, who uses it, how. Claims are messy and contradictory. That's expected.
4. **Decode** — resolve competing usage claims into a canonical understanding. Surface conflicts for human review. Build the map.
5. **Decompose** — derive optimal data product boundaries from the co-access graph, FK structure, lineage, and business function labels. Generate target schemas and verify rules. Compete multiple decomposition strategies.
6. **Infer** — scanned parsers and ingestors reveal fingerprints and profiles of input documents. Apply those to the document corpus to identify gaps and holes in what Oracle actually ingested.
7. **Reconcile** — compare what Oracle contains against what the documents say it should contain. The spine tools (compare, rvl, verify) prove where Oracle is right and where it has holes.
8. **Twin** the targets — each data product/mart gets a twin pair (Twin A + Twin B) with schemas generated by decompose.
9. **Assemble** — build candidate datasets from reconciled sources (Oracle extraction + document re-parsing where needed). Load into twins. Score.
10. **Tournament** — multiple assembly strategies compete. Score against Oracle as ground truth, against document evidence, against the gold set. Cross-boundary failures feed back to decompose. Best assembly wins per target.
11. **Seal** — evidence packs prove each data product is correct before cutover.

The data flowing from Oracle to each data product passes through the spine: shape for structure, verify for rules, compare/rvl for equivalence, benchmark for accuracy, assess for go/no-go, pack for evidence. The spine is everywhere there is data.

---

## Phase 1: Scan (Archaeology)

### What scan does

Scan is reconnaissance. It doesn't transform data — it produces claims about what exists.

### Database scan

```bash
factory scan --db oracle://... --output scan-results/
```

| Source | Claims produced | Confidence |
|--------|----------------|------------|
| DDL (tables, columns, types, PKs, FKs, CHECK, NOT NULL) | Schema structure, hard constraints | 1.0 |
| Stored procedures + triggers | Data flow, transformation logic, implicit rules | 0.6-0.8 |
| Views + materialized views | Derived data relationships, aggregation logic | 0.7-0.9 |
| Scheduled jobs | What runs, when, what it touches | 0.7 |
| `v$sql` / `dba_hist_sqlstat` | Which tables are actually queried, how often, by whom | 0.8 |
| `dba_audit_trail` | Who connects, what they do | 0.7 |
| Row counts, null rates, value distributions | What's alive, what's dead, what's sparse | 0.5-0.8 |
| FK relationships + JOIN patterns | Lineage between tables | 0.9 |

### Codebase scan

```bash
factory scan --repo /path/to/app --output scan-results/
```

Scan every application that touches the database. Expect mess — 20 years of Java, Python, shell scripts, Excel macros with ODBC, Crystal Reports, maybe COBOL. Each produces claims:

| Source | Claims produced | Confidence |
|--------|----------------|------------|
| SQL queries in application code | Which tables/columns matter, JOIN paths, WHERE clauses | 0.7-0.8 |
| ORM models (Hibernate, SQLAlchemy, Django) | Field types, validators, relationships | 0.8-0.9 |
| Report definitions (Crystal, SSRS, Jasper) | Expected data shapes, aggregations | 0.6-0.7 |
| API endpoints / route handlers | Which data is served to users, expected response shapes | 0.7 |
| Test files | Domain assertions, edge cases, expected values | 0.8 |
| Config files / connection strings | Which apps connect to which schemas | 0.9 |
| Enum definitions / constant tables | Valid value sets | 0.8 |
| COBOL copybooks | Exact byte-level field layouts (PIC clauses), record structure, data types | 1.0 |
| JCL job definitions | Program-to-dataset lineage, step dependencies, execution schedules, condition codes | 0.8-0.9 |

### Copybook codec layer

When scan encounters COBOL copybooks, it produces three artifacts from a single parse:

1. **Structural claims** (confidence 1.0) — field names, types, offsets, lengths. `PIC S9(7)V99 COMP-3` declares a signed 7.2 packed decimal at an exact byte offset. No ambiguity.
2. **Conversion codec** — deterministic EBCDIC-to-ASCII, packed decimal-to-numeric, zone decimal-to-numeric conversion for every field in the record. This codec is how mainframe data enters the spine's text/numeric world.
3. **Shape definition** — structural compatibility contract for the spine's `shape` tool, generated directly from PIC clauses.

```bash
factory scan --copybook LOAN-RECORD.cpy --output scan-results/
# Produces: claims/copybook-loan-record.claims.jsonl
#           codecs/LOAN-RECORD.codec.json
#           shapes/LOAN-RECORD.shape.json
```

### Inferred fingerprints and profiles

When scan encounters parsers and ingestors in the application code — the code that *feeds* Oracle — it infers fingerprints and profiles for the input documents those parsers consumed. These fingerprints can then be applied to the actual document corpus to:

- Identify documents that were parsed and loaded into Oracle
- Identify documents that should have been parsed but weren't (gaps)
- Identify documents that were parsed incorrectly (holes — where Oracle's data doesn't match the source)
- Decide whether re-parsing from original documents produces higher quality than extracting from Oracle

### Scan output

```
scan-results/
├── claims/                  # All claims as JSONL (one file per source)
│   ├── ddl.claims.jsonl
│   ├── stored-procs.claims.jsonl
│   ├── app-java-risk.claims.jsonl
│   ├── app-python-reporting.claims.jsonl
│   └── crystal-reports.claims.jsonl
├── inferred/
│   ├── fingerprints/        # Inferred document fingerprints from parser code
│   └── profiles/            # Inferred data profiles from application models
├── lineage/
│   └── table-lineage.json   # Table-to-table, app-to-table relationships
├── liveness/
│   └── activity-report.json # What's alive, what's dead, usage frequency
└── scan-report.json         # Summary: what was scanned, what was found, gaps
```

### What scan doesn't do

It doesn't understand intent. If the bank's analysts know "DSCR below 1.0 means distressed" but that rule isn't encoded anywhere, scan won't find it. The gap report flags what's missing. Humans fill in the tribal knowledge.

---

## Phase 2: Claims + Decoding

### Claims

Every piece of evidence about what the legacy system is and does gets expressed as a claim:

```json
{
  "event": "claim.v0",
  "claim_id": "sha256:...",
  "source": {
    "kind": "codebase_scan",
    "repo": "risk-reporting-app",
    "file": "src/main/java/com/bank/risk/LoanService.java:142",
    "evidence": "SELECT balance, status FROM loan_master WHERE deal_id = ?"
  },
  "proposed": {
    "subject": "table:loan_master",
    "assertion": "queried_by",
    "detail": {
      "application": "risk-reporting-app",
      "columns_used": ["balance", "status", "deal_id"],
      "join_path": "loan_master.deal_id → deal_header.deal_id",
      "frequency": "daily"
    }
  },
  "confidence": 0.75
}
```

Claims can be about:
- **Structure** — "table X has FK to table Y" (from DDL)
- **Usage** — "the risk app queries columns A, B, C daily" (from codebase)
- **Rules** — "column balance is never negative" (from CHECK constraint or app validator)
- **Liveness** — "table old_backup hasn't been read since 2021" (from audit logs)
- **Semantics** — "column 'status' means loan status with values active/closed/default" (from Java enum)
- **Lineage** — "stored proc P reads from table A, writes to table B" (from proc analysis)

Claims are expected to be messy, overlapping, and contradictory. That's the point.

### Convergence (the fountain model)

Each property of each table is a **bucket**: `(table, column, property_type)`. Claims pour into buckets from multiple independent sources. The `decoding` tool tracks the state of every bucket using its convergence engine:

| State | Meaning | Action |
|-------|---------|--------|
| **Empty** | No source has said anything about this | Unknown — may be dead, may be undiscovered |
| **Single-source** | One source, one claim | Low confidence — need corroboration |
| **Converging** | Multiple sources, consistent | Confidence rises with each independent confirmation |
| **Converged** | Enough independent sources agree | High confidence — move on |
| **Conflicted** | Sources actively disagree | Needs human resolution |

The key insight from fountain codes: **you don't need every source to be complete. You need enough independent sources agreeing to converge.** A Java app that only touches 30 of 1000 tables still fills 30 high-value buckets. A COBOL batch job nobody understands fills 12 more. Crystal Reports fills 8. No single source covers everything. Together they converge.

**Convergence thresholds are per-property-type:**

| Property | Converges when | Rationale |
|----------|---------------|-----------|
| Column exists + type | DDL alone (confidence 1.0) | Schema is fact |
| NOT NULL / FK / CHECK | DDL alone (confidence 1.0) | Hard constraint |
| Valid value set | 2+ independent sources agree | App enums may be stale; DDL CHECK is definitive |
| Usage (which apps query it) | All scanned apps processed | Usage is additive, not convergent |
| Liveness (alive/dead) | Audit logs + 1+ app source agree | Both behavioral and structural evidence |
| Semantics (what it means) | 2+ sources with consistent interpretation | Column names are ambiguous; usage reveals intent |

**Convergence tells you when to stop scanning.** If you've scanned 3 of 5 apps and 90% of buckets have converged, the marginal value of reverse-engineering the COBOL is measurable: it would fill at most N remaining empty/single-source buckets. If N is small, skip it. If N is large, it's worth the effort. The decision is data-driven, not gut-driven.

**Convergence also tells you where to focus humans.** Conflicted buckets are where expert attention has the highest leverage — two sources disagree, a human resolves it in 30 seconds, the bucket converges. The gap dashboard ranks conflicts by impact (how many downstream data products depend on this bucket?) so humans work the highest-leverage conflicts first.

See `decoding` plan for full convergence model details — bucket state machine, claim shapes, thresholds, and dashboard specification.

### Convergence on the dashboard

```
Buckets: 14,200 total (1000 tables x ~14.2 properties avg)

  Converged:      11,340  (79.9%)  --------------------------------  done
  Converging:      1,420  (10.0%)  ----                              need 1 more source
  Single-source:     890  ( 6.3%)  ---                               low confidence
  Conflicted:        210  ( 1.5%)  -                                 needs human
  Empty:             340  ( 2.4%)  -                                 undiscovered

Sources scanned: 5 of 7 known applications
  Marginal value of next source (python-etl): ~180 buckets (fills 120 single-source, 60 empty)
  Marginal value of next source (cobol-batch): ~40 buckets (fills 30 single-source, 10 empty)
  -> Recommendation: scan python-etl next, defer cobol-batch
```

### Decoding

`decoding` resolves converged and conflicted buckets into a canonical understanding (see `decoding` plan for full specification):

- **Converged buckets** resolve automatically — the decoder records the canonical value, the contributing claims, and the convergence path.
- **Conflicted buckets** surface for human review with full context: all competing claims, their sources, their confidence scores, and the specific disagreement. Two apps define different valid values for the same column → the human picks the canonical set and the decision is recorded.
- **Single-source buckets** stay provisional — flagged with their confidence and marked as candidates for corroboration from the next scanned source.

The decoder also handles legitimate multiplicity: a table used differently by three apps doesn't have one usage profile — it has three. All three are claims. All three are valid. All three feed the migration plan. The decoder doesn't force false convergence; it surfaces the actual complexity.

**The output is a canonical map:** every table classified (alive/dead/uncertain), every column annotated (used by whom, constrained how, means what), every relationship traced (FK, JOIN pattern, proc flow), every gap flagged — and every classification backed by a convergence record showing which sources contributed and how they agreed.

---

## Phase 3: Decompose (Derive Target Architecture from Evidence)

### The problem nobody talks about

Every database migration starts the same way: architects spend weeks drawing boxes on a whiteboard. "These tables go in the loan performance mart. Those go in compliance. That one goes... somewhere." It takes weeks. It's wrong. It changes. It's the single biggest reason migrations stall — not the data movement, but the *target design*.

The factory already has everything it needs to derive the target architecture. After scan + decode, the canonical map contains:

- **Table co-access patterns** — from `v$sql` / codebase scan: which tables are always queried together
- **FK graph** — from DDL: structural coupling between tables
- **Application ownership** — from codebase scan: which apps touch which tables
- **Lineage flows** — from stored procs, views, JCL: data flowing between tables
- **Liveness classification** — from audit logs + decode: what's alive, what's dead
- **Business function labels** — from application scan: "risk-reporting-app" maps to the risk domain

That's a weighted graph. Tables are nodes. JOINs, FK relationships, co-access, lineage are edges. The question "how should this decompose into data products" is a graph partitioning problem: find clusters of tables that are tightly coupled internally and loosely coupled externally.

### `factory decompose`

```bash
factory decompose --claims scan-results/ --decode-map decode-map.json --output decomposition/
```

The decompose step:

1. **Build the co-access graph.** `weight(A, B)` = how often tables A and B appear in the same query, join path, stored proc, or application. Edges from FK relationships get structural weight. Edges from lineage (proc reads A, writes B) get flow weight. Edges from co-access in the same application get usage weight.

2. **Run community detection.** Louvain or Leiden algorithm — O(n log n), well-understood, deterministic. Finds natural clusters of tables that maximize internal cohesion and minimize external coupling. The modularity score quantifies how clean the boundaries are.

3. **Overlay business function labels.** The risk-reporting-app touches tables in cluster 1. The compliance reports touch tables in cluster 3. The algorithm finds structural clusters; the application labels give them business names. Cluster 1 becomes "loan-performance." Cluster 3 becomes "compliance."

4. **Identify shared dimensions.** Some tables appear in multiple clusters above a threshold — `counterparty_master` is queried by risk, compliance, and portfolio analytics. These are shared dimensions that need replication or a common service. The decomposition flags them explicitly.

5. **Flag orphans.** Alive tables that don't cluster cleanly with anything. Dead tables that can be archived. Tables with uncertain liveness that need human investigation.

6. **Generate target schemas.** Each cluster's tables, columns, constraints (from DDL), verify rules (from decoded application validators), and expected query patterns (from `v$sql`) are assembled into a Twin B schema definition. The architect doesn't write DDL. The factory writes DDL. The architect reviews and adjusts.

### Decomposition output

```json
{
  "version": "factory_decompose.v0",
  "source": {
    "total_tables": 1000,
    "alive_tables": 847,
    "dead_tables": 153
  },
  "decomposition": {
    "strategy": "co-access-community",
    "modularity_score": 0.73,
    "products_proposed": 24,
    "tables_assigned": 623,
    "shared_dimensions": 18,
    "orphans": 53,
    "cross_cluster_edges": 34
  },
  "products": [
    {
      "name": "loan-performance",
      "tables": 47,
      "primary_consumers": ["risk-reporting-app", "surveillance-app"],
      "cohesion": 0.91,
      "coupling": 0.12,
      "verify_rules_derived": 23,
      "query_patterns_captured": 912,
      "schema_generated": "schemas/loan-performance.sql",
      "confidence": "converged"
    },
    {
      "name": "deal-structure",
      "tables": 23,
      "primary_consumers": ["portfolio-analytics", "investor-reporting"],
      "cohesion": 0.87,
      "coupling": 0.18,
      "verify_rules_derived": 14,
      "query_patterns_captured": 340,
      "schema_generated": "schemas/deal-structure.sql",
      "confidence": "converged"
    }
  ],
  "shared_dimensions": [
    {
      "table": "counterparty_master",
      "appears_in": ["loan-performance", "deal-structure", "compliance"],
      "recommendation": "replicate_or_service",
      "access_pattern": "read-only from all consumers",
      "rows": 12400
    }
  ],
  "orphans": [
    {
      "table": "tmp_batch_2019",
      "liveness": "dead",
      "last_accessed": "2019-04-12",
      "recommendation": "archive"
    },
    {
      "table": "custom_analytics_staging",
      "liveness": "alive",
      "consumers": ["unknown — ad-hoc SQL from 2 connection pools"],
      "recommendation": "investigate"
    }
  ]
}
```

### Boundaries are claims

The proposed boundaries pour into the convergence model like everything else. "Table `loan_master` belongs to cluster `loan-performance`" is a claim. If an agent working on deal structure also needs `loan_master`, that's a conflicting claim — `loan_master` appears in two clusters. The decoder surfaces it. A human decides: replicate it, assign it to one cluster and expose a service, or merge the two clusters. The decision is recorded. The boundary converges.

**The decomposition itself is a convergent process.** Early boundaries are provisional. As the tournament runs and some boundaries fail (query replay can't complete because a JOIN crosses a boundary), the failed tests produce counter-evidence. The factory refines the boundaries and re-decomposes. The loop:

```
decompose → twin → tournament
    ^                   |
    |   (boundary wrong) |
    +-------------------+
```

### Competing decompositions

Multiple decomposition strategies can compete in the same tournament:

```bash
# Three strategies: tight clusters, loose clusters, domain-aligned
factory decompose --claims scan-results/ --strategy tight --output decomp-tight/
factory decompose --claims scan-results/ --strategy loose --output decomp-loose/
factory decompose --claims scan-results/ --strategy domain --output decomp-domain/

# Tournament scores each decomposition end-to-end
for strategy in tight loose domain; do
  factory tournament --decomposition decomp-${strategy}/ \
    --claims scan-results/ --output tournament-${strategy}/
done

# Winner: the decomposition where the most data products pass at the highest score
```

The "tight" strategy produces many small data products (high cohesion, low coupling, but more shared dimensions to manage). The "loose" strategy produces fewer larger products (simpler to manage, but lower cohesion). The "domain" strategy aligns boundaries with business functions (may not match structural access patterns). The tournament proves which decomposition actually works — not which one looks best on a whiteboard.

### What this replaces

Every database migration in history starts with architects spending weeks drawing boxes. The design is based on tribal knowledge, meetings, and gut feel. It takes weeks. It's wrong. It changes mid-migration. Nobody can prove the boundaries are right until the migration is half-done and something breaks.

`factory decompose` replaces weeks of architecture workshops with a command that runs in minutes and produces falsifiable boundaries backed by convergence records. The boundaries come from evidence — co-access patterns, FK relationships, lineage, liveness — not from humans guessing. And the tournament proves whether they work before any data moves.

---

## Phase 4: Twin + Tournament

### Twinning the targets

The Oracle monolith fragments into data products and marts — each proposed by `factory decompose` and validated by the tournament. Each target gets a **twin pair**: Twin A (legacy schema for replay) and Twin B (target schema for scoring). Twin B schemas are generated by the decompose step from decoded claims. See `twinning` plan for the two-twin design specification.

```bash
# Twin A: Oracle schema for behavioral replay (tables in this cluster)
twinning postgres --schema decomp/schemas/oracle-loan-tables.sql --port 5433

# Twin B: Generated target schema with derived verify rules
twinning postgres --schema decomp/schemas/loan-performance.sql \
  --rules decomp/rules/loan-performance.json --port 5434
```

### Tournament scoring

For each target data product, the tournament asks: can we assemble correct data from the available sources, AND are the boundaries right?

**Candidates:**
- Direct Oracle extraction (SELECT from the relevant tables)
- Document re-parsing (re-extract from original source documents where gaps were found)
- Hybrid (Oracle extraction + document re-parsing for specific gaps)
- Multiple transformation strategies for the same source data

**Scoring:**
- `benchmark` against gold set (human-validated reference values)
- `verify` against rules (data contracts derived from scan + human review)
- `compare` against Oracle (exhaustive diff — does the data product match what Oracle has?)
- `rvl` for materiality (which differences matter?)
- `assess` for go/no-go (policy decision: PROCEED, PROCEED_WITH_RISK, ESCALATE, BLOCK)

**Boundary validation (new):**
- Query replay that crosses a boundary fails → signal that the boundary is wrong
- Verify rules referencing tables outside the boundary → signal that a table is missing
- FK violations because a referenced table is in a different product → shared dimension or boundary error

**The loop:**
1. Assemble candidate data for target mart
2. Load into twin
3. Score (benchmark + verify + compare + assess)
4. If PROCEED: seal evidence, harvest gold set, move to next target
5. If BOUNDARY_ERROR: feed back to decompose, refine, re-twin, re-score
6. If ASSEMBLY_ERROR: iterate on assembly strategy, re-score
7. The gold set grows with each approved target. Next target has a higher bar.

### Coverage dashboard

At any point, the dashboard shows:

```
Legacy Oracle: 1000 tables

Scanned:           1000 / 1000  (100%)
Classified:         847 / 1000  ( 85%)  — alive/dead/uncertain
Decomposed:         847 / 847   (100%)  — proposed by factory decompose
  Products:         24 proposed (modularity 0.73)
  Shared dims:      18 tables (flagged for replication/service)
  Orphans:          53 tables (39 dead → archive, 14 alive → investigate)
Mapped to target:   623 / 847   ( 74%)  — assigned and accepted by human review
Proven equivalent:  412 / 623   ( 66%)  — evidence pack sealed
Remaining:          211 / 623   ( 34%)  — in progress or blocked

Boundary health:
  Clean:            21 / 24     — no cross-boundary failures
  Needs refinement:  2 / 24     — query replay flagged missing tables
  Blocked:           1 / 24     — shared dimension conflict unresolved

Dead tables:        153          — not queried in 2+ years, no active consumers
Uncertain:          153          — conflicting claims, needs human review

Data products:      24 targets
  Completed:        14 / 24     — all tables proven, evidence sealed
  In progress:       7 / 24     — tournament running
  Blocked:           3 / 24     — waiting on human resolution of conflicts
```

This is what the previous 5 retirement attempts didn't have: a precise, continuously-updated map of what's done, what's left, and what's blocking. And for the first time in any migration ever: the target architecture itself was derived from evidence, not drawn on a whiteboard.

---

## Phase 5: Replay (Behavioral Equivalence)

### The two-twin design

Each data product gets two twins:

**Twin A — Oracle schema.** Same tables, same columns, same relationships as the legacy database. Load the migrated data into Oracle's own schema structure. This twin speaks Oracle's language.

**Twin B — Target schema.** The clean, purpose-built schema for the data product. Load the transformed data. This twin speaks the new model's language.

```bash
# Twin A: Oracle schema for the loan performance slice
twinning postgres --schema oracle-loan-tables.sql --port 5433

# Twin B: New data product schema
twinning postgres --schema loan-perf-schema.sql --rules loan-perf-rules.json --port 5434
```

### Why two twins

The query translation problem disappears. You don't need to rewrite `SELECT balance, status FROM loan_master WHERE deal_id = ?` into new-schema SQL. You replay it verbatim against Twin A, which has the same schema as Oracle. If the result sets match, the data migration didn't lose or corrupt anything.

Twin B proves a different thing: the new data product is correct on its own terms. Verify rules pass, benchmark scores meet the bar, assess says PROCEED.

Two independent proofs:
1. **Behavioral equivalence** — Twin A + replay: "the migrated data answers the same questions Oracle did"
2. **Target correctness** — Twin B + spine scoring: "the new data product satisfies its own contracts"

The transformation logic between Twin A and Twin B is itself testable — load the same source data into both, export, run `compare`. Any difference is either an intentional schema change (documented) or a bug.

### Replay

`factory scan` captured historical query and execution patterns. `factory replay` turns those into an executable behavioral test suite.

**SQL replay** (Oracle, DB2, SQL Server): Captured from `v$sql`, `dba_hist_sqlstat`, or equivalent. Each historical query replayed verbatim against Twin A.

```bash
factory replay --queries scan-results/queries/risk-app.sql \
  --oracle oracle://... \
  --twin localhost:5433 \
  --output replay-results/
```

For each historical query:
1. Run against the legacy system (live, or cached result sets from scan)
2. Run the same SQL verbatim against Twin A
3. Compare result sets (row-for-row, column-for-column)
4. Classify: MATCH, MISMATCH, or SKIP (query uses unsupported SQL features)

**Program execution replay** (COBOL batch jobs): Captured from JCL job streams. Each JCL step maps to one program execution. The unit of replay is a compiled program, not a SQL query. Requires non-SQL twin types (VSAM, flat file) — see `twinning` plan for the generalized interface model.

```bash
factory replay --jcl scan-results/jobs/NIGHTLY.jcl \
  --vsam-twin localhost:6000 \
  --db2-twin localhost:5433 \
  --mainframe-outputs expected/ \
  --output replay-results/
```

For each JCL step:
1. Set up input datasets (load into appropriate twins — VSAM, DB2, flat file)
2. Run the COBOL program (compiled via GnuCOBOL) against the twins
3. Capture output datasets and return codes
4. Compare outputs against known-good mainframe outputs
5. Classify: MATCH, MISMATCH, or SKIP (program uses unsupported system services)

### Replay output

```json
{
  "version": "factory_replay.v0",
  "data_product": "loan-performance-mart",
  "queries_total": 912,
  "queries_matched": 905,
  "queries_mismatched": 4,
  "queries_skipped": 3,
  "match_rate": 0.9923,
  "mismatches": [
    {
      "query_id": "sha256:...",
      "source_app": "risk-reporting-app",
      "sql": "SELECT SUM(balance) FROM loan_master WHERE status = 'delinquent'",
      "oracle_result": "45230000.00",
      "twin_result": "45229500.00",
      "delta": "500.00",
      "classification": "rounding"
    }
  ],
  "skipped": [
    {
      "query_id": "sha256:...",
      "source_app": "legacy-cobol-batch",
      "sql": "SELECT ... CONNECT BY PRIOR ...",
      "reason": "hierarchical_query_unsupported"
    }
  ]
}
```

### Replay coverage on the dashboard

The coverage dashboard gains a new axis:

```
Data product: loan-performance-mart

  Static proof:
    Tables compared:    47 / 47    (100%)
    Rows matched:       99.98%
    Verify rules:       PASS (12/12)
    Benchmark accuracy: 0.997

  Behavioral proof:
    Query patterns:     912 captured from 5 applications
    Replayed:           909 / 912  (99.7%)
    Matched:            905 / 909  (99.6%)
    Mismatched:         4          (all classified as rounding)
    Skipped:            3          (hierarchical queries — manual equivalents pending)

  Assess decision:      PROCEED_WITH_RISK
  Risk factors:         4 rounding mismatches (bounded, < $0.01 per row)
```

### What the bank couldn't say before

Previous attempts: "We migrated the data. It looks right. We think it works."

With replay: "We replayed 12 months of production queries from 5 applications against the migrated data. 99.6% returned identical results. 4 queries had sub-penny rounding differences. 3 Oracle-specific hierarchical queries need manual equivalents. Here's the evidence pack."

That's not a belief. That's a proof.

---

## Phase 6: Prove + Seal

### Evidence per data product

Each completed data product migration produces:

```
evidence/loan-performance-mart/
├── scan-claims.pack           # All claims about the source tables
├── decode-map.pack            # Canonical understanding of what these tables do
├── assembly-strategy.json     # How the data was assembled (Oracle + re-parsed docs)
├── replay.report.json         # Behavioral equivalence (query replay results)
├── benchmark.report.json      # Gold set accuracy
├── verify.report.json         # Rule compliance
├── compare.report.json        # Exhaustive diff vs Oracle source
├── rvl.report.json            # Materiality analysis
├── assess.report.json         # Policy decision (covers both static + behavioral)
├── data.lock.json             # Content-addressed lockfile for the data
└── evidence.pack              # Sealed evidence bundle
```

### What the auditors see

Not "we migrated the database." Instead: "Here are 24 evidence packs, one per data product. Each one proves two things independently: the data matches (static equivalence via compare + verify + benchmark) AND the system behaves the same (behavioral equivalence via query replay). Every number traces to a source. Every query is reproducible. Every decision is sealed."

That's why the previous attempts failed and this one won't. Not better technology — better proof.

---

## Agent swarm (the parallel factory)

### The operational model

The factory runs as 15-20 parallel `ntm` sessions, each an independent agent with its own working directory and (when needed) its own twin instance. There is no custom orchestration framework. Coordination happens through three mechanisms that already exist:

1. **Convergence dashboard** — every agent can read the current bucket state. When agent 7 finishes scanning the Java risk app, it commits 340 claims. The dashboard updates. Every other agent sees 340 buckets fill. No message passing required.
2. **Git** — claims are JSONL files in a shared repo. Agents commit and push. No locks, no coordination service. Content-addressed claim IDs mean duplicate claims from different agents are idempotent.
3. **Work allocation by phase** — agents self-select from the highest-value available work, guided by the dashboard.

### Swarm topology

The factory runs in waves. Each wave saturates a phase before moving on:

**Wave 1: Scan (5-7 agents)**
Each agent scans one application codebase or one slice of Oracle. They run in parallel with zero coordination — each produces independent claims.

```
ntm spawn scan-java-risk        # Agent scans the Java risk app
ntm spawn scan-python-reporting  # Agent scans the Python reporting scripts
ntm spawn scan-crystal-reports   # Agent scans Crystal Reports definitions
ntm spawn scan-oracle-procs      # Agent scans stored procedures
ntm spawn scan-oracle-jobs       # Agent scans scheduled jobs + audit logs
```

Each agent: scan → produce claims (JSONL) → commit → push. When all scan agents finish, the convergence dashboard shows the state of all 14,200 buckets. Humans review conflicted buckets.

**Wave 1.5: Decompose (1-3 agents)**
After scan converges and humans resolve conflicts, decompose agents propose and compete target architectures.

```
ntm spawn decompose-tight          # Tight clustering (many small products)
ntm spawn decompose-domain         # Domain-aligned clustering
ntm spawn decompose-hybrid         # Hybrid: structural clusters refined by domain
```

Each agent: build co-access graph → run community detection with strategy-specific parameters → generate target schemas → commit. A meta-tournament compares the decomposition strategies before the main tournament begins.

**Wave 2: Tournament (10-15 agents)**
Each agent works one data product from the winning decomposition. Boots its own twin pair (Twin A + Twin B), assembles candidate data, loads, scores, iterates.

```
ntm spawn dp-loan-performance   # Agent works the loan performance mart
ntm spawn dp-deal-structure     # Agent works the deal structure mart
ntm spawn dp-compliance         # Agent works the compliance data product
ntm spawn dp-surveillance       # Agent works the surveillance mart
...                             # 10-15 data products in parallel
```

Each agent: assemble → load twin → score (benchmark + verify + compare + assess) → iterate → seal evidence pack. Agents don't interact. Each twin is isolated. Each evidence pack is independent.

**Wave 3: Replay (5-10 agents)**
Each agent replays historical queries for one data product against its Twin A.

```
ntm spawn replay-loan-perf      # Replay risk app queries against loan perf Twin A
ntm spawn replay-compliance     # Replay compliance queries against compliance Twin A
...
```

### Why this works without a framework

The convergence model is the coordination mechanism. Agents are the fountain encoders — they spray claims from independent sources. The convergence dashboard is the decoder — it tracks when enough claims have landed to justify confidence. No agent needs to know what any other agent is doing. They just:

1. Read the dashboard to find the highest-value work
2. Do the work (scan, assemble, score, replay)
3. Commit results (claims, reports, evidence packs)
4. The dashboard updates

This is the agent swarm from the old plan, minus the custom orchestration framework. `ntm` handles session management. Git handles state sharing. Convergence handles coordination. Content-addressing handles deduplication.

### Agent isolation guarantees

- Each agent has its own working directory (ntm provides this)
- Each agent boots its own twin instances (no shared database state)
- Claims are content-addressed (duplicate claims from different agents are harmless)
- Evidence packs are per-data-product (no cross-agent output conflicts)
- The only shared mutable state is the git repo, and JSONL is append-friendly

### Scaling

| Phase | Agents | Bottleneck | Duration |
|-------|--------|-----------|----------|
| Scan | 5-7 | Number of applications + Oracle slices | Hours to days |
| Human review | 1-3 | Conflicted buckets | Days (front-loaded) |
| Tournament | 10-15 | Number of data products | Days to weeks |
| Replay | 5-10 | Number of data products × query volume | Hours to days |
| Evidence sealing | 1 per product | Sequential per product (fast) | Minutes each |

20 agents working for 2 weeks can scan, assemble, tournament-score, replay, and seal evidence for 24 data products. That's a 1000-table Oracle database decomposed with proof in 2 weeks. The previous attempts took years.

---

## What we killed from the old plan

The original PLAN_FACTORY.md was 2000 lines. This is what we cut and why:

| Cut | Why |
|-----|-----|
| Agent swarm custom framework | Replaced with ntm + git + convergence. The swarm concept survived — the custom orchestration layer didn't. |
| Fountain code / RaptorQ implementation | The event stream / reactive projector implementation. The convergence *model* survived in `decoding` — it's how claims from multiple shitty sources justify confidence. |
| Event stream architecture (NATS, Redis Streams) | Premature infrastructure. Claims are JSONL files. `decoding` reads them directly. |
| Passive projector pattern | Over-abstraction. Decoding reads claims, emits mutations. A function, not a reactive stream. |
| Governance framework | Important but premature. Ship the factory, then figure out how it decays. |
| Answer engine (charts, visual artifacts) | Downstream of the core problem. Build it when the data products exist. |
| Cross-asset-class vision | Aspiration, not a plan. Prove CMBS first. |
| Commercial model / pricing | Not an engineering artifact. |
| 5-phase factory process (rigid) | Replaced with a tournament loop that works for both modes. |

What survived: scanning, claims, decoding, twinning, tournament scoring, gold set flywheel, coverage tracking, evidence sealing. The core ideas are right. The 2000-line wrapper was wrong.

---

## Relationship to the spine

The spine is the proof layer. The factory is the orchestration layer.

```
+----------------------------------------------------------+
|                    FACTORY                                 |
|                                                           |
|   scan → claims → decode → decompose → twin → tournament  |
|                               ^                    |      |
|                               |  (boundary wrong)  |      |
|                               +--------------------+      |
|                                          ↓                |
|                                  replay → seal            |
|                                                           |
+-----------------------------------------------------------+
|                    SPINE                                   |
|                                                           |
|   vacuum hash fingerprint profile lock shape rvl          |
|   verify compare benchmark assess canon pack              |
|                                                           |
|   (works anywhere there is data)                          |
+-----------------------------------------------------------+
```

When data flows from Oracle to a data product, it passes through the spine. When documents get re-parsed, the output goes through the spine. When a twin scores a candidate, it uses the spine. The factory orchestrates; the spine proves.

---

## Implementation order

1. **factory scan --db** — Database introspection. Highest confidence claims. Immediately useful.
2. **factory scan --repo** — Codebase scanning. Start with SQL string extraction (easy, high value), add ORM/model parsing iteratively.
3. **Claim format + storage** — JSONL files, content-addressed. No message bus.
4. **Decode v0** — Resolve structural claims (DDL + codebase). Surface conflicts. Build the map.
5. **factory decompose** — Co-access graph + community detection + boundary proposal. Generates target schemas and verify rules from decoded claims. The architecture step that replaces weeks of whiteboard sessions.
6. **Coverage dashboard** — Alive/dead/decomposed/mapped/proven counts. Includes boundary health and decomposition metrics.
7. **Two twins per target** — Twin A (Oracle schema) + Twin B (generated target schema). Schemas from decompose step. Rules from decoded claims.
8. **Tournament loop** — Assemble, load, score, iterate. Boundary validation alongside data quality scoring. Spine tools do the scoring.
9. **factory replay** — Behavioral equivalence. Replay historical queries against Twin A. Compare result sets. Cross-boundary failures feed back to decompose.
10. **Evidence sealing** — Pack per data product. Static proof + behavioral proof + boundary proof + assess decision.

Each step is independently useful. You don't need step 10 to get value from step 1.

---

## On COBOL and mainframes

The factory architecture was designed around two modes: document corpus decomposition and legacy SQL database decomposition. But the hardest legacy systems aren't SQL databases — they're COBOL mainframes. A 40-year-old COBOL/VSAM/IMS/CICS system with DB2 is the ultimate test of whether this architecture generalizes. This section analyzes what works, what breaks, and how to solve the gaps without changing the architecture.

### What transfers cleanly

**The convergence model works perfectly.** COBOL mainframes actually have *more* independent sources than Oracle. COBOL programs, JCL job streams, copybooks, DB2 DDL, CICS transaction definitions, batch schedules, operator manuals — each emits claims about what the system does. The fountain model doesn't care that the sources are in COBOL. It cares that they're independent.

**Tournament scoring works identically.** Data is data. `benchmark`, `verify`, `compare`, `assess`, `pack` — none of them know or care that the source was a mainframe. Assemble candidate data, score it, best one wins.

**Evidence sealing works identically.** The spine doesn't touch the source system.

### What works but gets harder

**Scanning COBOL is surprisingly tractable.** COBOL is in some ways *easier* to scan than Java. The `DATA DIVISION` declares every field with exact byte layout — `PIC S9(7)V99 COMP-3` tells you it's a signed 7.2 packed decimal. Copybooks are the schema. You get incredibly precise structural claims at confidence 1.0.

The hard part: COBOL programs are 10,000-50,000 lines of spaghetti maintained for 40 years by copy-paste. You find 12 programs that do almost the same thing with slight variations. The convergence model sees 12 sources "agreeing" — but they're copies, not independent corroboration. The derivation graph in `decoding` handles this via clone detection: diff every program pair, cluster by similarity, each clone cluster votes as one source with weight 1, not as N independent sources. COBOL is the easiest language to do this for — no metaprogramming, no dynamic dispatch, no monkey-patching.

**JCL is the richest claim source on the mainframe.** A single JCL job declares which programs run, in what order, what files each step reads (DD statements map to VSAM datasets, flat files, DB2 tables), what files each step writes, and condition codes for branching. JCL IS the lineage graph. A JCL parser producing claims covers program-to-dataset relationships across the entire mainframe in one pass. JCL is a structured format with well-documented syntax — parsing is tractable.

**DB2 on the mainframe works almost like Oracle.** DB2 has SQL traces, DDL introspection, and the SQL is close enough to Postgres that the twin handles it. Maybe 40-60% of a typical mainframe's data lives in DB2. For that slice, the stack works today.

### The key reframe: the twin is an interface emulator

The twin's core abstraction is "speak the protocol the client expects." That's the right idea — it's just too narrow if we only implement SQL wire protocols. The generalization (specified in the `twinning` plan):

| Interface | Protocol | Twin implementation | LOC estimate |
|-----------|----------|-------------------|-------------|
| SQL (DB2/Postgres) | SQL wire protocol | Current twin (pgwire) | ~10-15K (exists) |
| VSAM | COBOL file I/O (OPEN/READ/WRITE/CLOSE on keyed datasets) | In-memory keyed byte-array store | ~3-4K |
| IMS/DL/I | Hierarchical navigation (GU/GN/GNP/ISRT/REPL/DLET) | In-memory tree store | ~5-8K |
| Flat files | Sequential I/O with copybook layout | In-memory byte stream | ~1-2K |
| CICS | Transaction dispatch (EXEC CICS commands) | Transaction router + API surface mock | Commercial (Micro Focus) or mock top 50 |

The VSAM twin is the highest-value addition. It's simpler than the SQL twin (no query parsing, no planning — just keyed byte-array storage), covers ~30% of mainframe workloads, and combined with GnuCOBOL enables off-mainframe batch job replay.

### The copybook is the schema

A COBOL copybook declares every field with surgical precision:

```cobol
01 LOAN-RECORD.
   05 LOAN-ID         PIC X(10).
   05 BALANCE         PIC S9(9)V99 COMP-3.
   05 STATUS          PIC X(1).
   05 RATE            PIC 9V9(5) COMP-3.
   05 ORIGINATION-DT  PIC 9(8).
   05 FILLER          PIC X(43).
```

`PIC S9(9)V99 COMP-3` — signed, 9 integer digits, 2 decimal digits, packed decimal format. Exactly 6 bytes. No ambiguity. No inference needed. The copybook simultaneously produces: structural claims (confidence 1.0), a conversion codec (deterministic EBCDIC/packed decimal to native types), and a `shape` definition. One artifact, three outputs.

### Program execution replay

COBOL batch programs don't issue capturable SQL queries. They run. A nightly batch job reads VSAM datasets, processes records in a `PERFORM` loop, updates DB2 tables, and writes report files. The behavioral equivalence test is: **same inputs, same outputs.**

GnuCOBOL (open source, production-grade) compiles COBOL to native executables. Combined with twins for VSAM/DB2/flat files, you get an off-mainframe execution environment:

1. Capture input state from the mainframe (export VSAM datasets, export DB2 tables, collect flat files)
2. Load inputs into twins (VSAM twin for VSAM inputs, Postgres twin for DB2 inputs)
3. Compile and run the COBOL program against the twins
4. Capture output state (updated datasets, DB2 changes, report files, return codes)
5. `compare` output vs known-good output from the mainframe

JCL job streams become the replay script. Each JCL step maps to one program execution. Step dependencies map to sequential execution. DD statements map to twin connections. Condition codes map to assertions.

### The 80/20

| Category | % of typical mainframe | Stack coverage |
|----------|----------------------|---------------|
| DB2 programs | ~40% | **Works today** — Postgres twin handles DB2 SQL |
| VSAM batch programs | ~30% | **Solvable** — VSAM twin (~3-4K LOC) + GnuCOBOL |
| IMS programs | ~10% | **Hard** — IMS twin (~5-8K LOC), subtle navigation semantics |
| CICS online transactions | ~15% | **Commercial** — Micro Focus, or mock top 50 transactions |
| Assembler / vendor-specific | ~5% | **Manual** — factory identifies and classifies as SKIP |

The stack as designed handles 40%. With a VSAM twin and GnuCOBOL integration, it handles 70%. Adding IMS gets to 80%. The last 20% is either commercial tooling or manual work — but the factory's contribution is still real: it *finds and classifies* what needs manual handling instead of discovering it mid-migration.

### What's genuinely hard

**IMS.** Hierarchical databases with segment types, PCBs (Program Communication Blocks), SSAs (Segment Search Arguments), and navigational access. `GN` walks depth-first, `GNP` scopes to a subtree, `GU` does keyed lookup with concatenated keys. Implementable but subtle — fewer mainframes use IMS these days (DB2 has been eating its lunch for 20 years), but the ones that do are the hardest migrations.

**CICS.** Full emulation is hundreds of commands — screen management (BMS), inter-program communication, temporary storage queues, transient data, file control, interval control. The pragmatic move: extract CICS transaction logic into testable units, mock the CICS API surface for the top 50 transactions, accept that the long tail gets manual testing. Or use Micro Focus Enterprise Server commercially.

**Vendor extensions.** Programs that call assembler subroutines, use vendor-specific COBOL extensions (IBM vs Micro Focus vs Fujitsu), or depend on mainframe system services (RACF for security, SMF for auditing, HSM for storage management). These don't migrate — they get replaced. The factory's job is to identify them, classify them as SKIP, and surface them for human handling.

### What the architecture needs

Three additions to the existing stack, none of which change the architecture:

1. **Generalized twin interface** — The twin becomes an interface emulator with pluggable protocol backends: Postgres, VSAM, IMS, flat files. Same in-memory storage and constraint enforcement layer, different protocols. (Specified in the `twinning` plan.)

2. **Copybook codec layer** — Copybook parsing produces structural claims + conversion codec + shape definition from a single artifact. This bridges mainframe byte-level data into the spine's text/numeric world. (Specified in factory scan above.)

3. **Program execution replay** — Extend replay from "SQL query replay" to "program execution replay." JCL step maps to compile COBOL + run against twins + capture outputs + compare. Same MATCH/MISMATCH/SKIP reporting, different execution unit. (Specified in factory replay above.)

The convergence model, tournament scoring, evidence sealing, gold set flywheel, agent swarm — all unchanged. The factory scans different sources, the decoder resolves different claim shapes, the twins speak different protocols. The proof infrastructure is the same.
