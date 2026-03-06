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
5. **Infer** — scanned parsers and ingestors reveal fingerprints and profiles of input documents. Apply those to the document corpus to identify gaps and holes in what Oracle actually ingested.
6. **Reconcile** — compare what Oracle contains against what the documents say it should contain. The spine tools (compare, rvl, verify) prove where Oracle is right and where it has holes.
7. **Twin** the targets — each data product/mart gets a twin with its own schema and verify rules.
8. **Assemble** — build candidate datasets from reconciled sources (Oracle extraction + document re-parsing where needed). Load into twins. Score.
9. **Tournament** — multiple assembly strategies compete. Score against Oracle as ground truth, against document evidence, against the gold set. Best assembly wins per target.
10. **Seal** — evidence packs prove each data product is correct before cutover.

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

## Phase 3: Twin + Tournament

### Twinning the targets

The Oracle monolith doesn't get replaced by one system. It fragments into data products and marts — each purpose-built for a specific use case:

- Loan performance mart (for risk reporting)
- Deal structure mart (for portfolio analytics)
- Compliance data product (for regulatory reporting)
- Surveillance mart (for monitoring)
- ... dozens more

Each target gets a **twin pair**: Twin A (legacy schema for replay) and Twin B (target schema for scoring). Schemas are derived from the decoded claims about what that slice of Oracle actually does. See `twinning` plan for the two-twin design specification.

```bash
# Twin A: Oracle schema for behavioral replay
twinning postgres --schema oracle-loan-tables.sql --port 5433

# Twin B: New data product schema for target scoring
twinning postgres --schema loan-perf-schema.sql --rules loan-perf-rules.json --port 5434
```

### Tournament scoring

For each target data product, the tournament asks: can we assemble correct data from the available sources?

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

**The loop:**
1. Assemble candidate data for target mart
2. Load into twin
3. Score (benchmark + verify + compare + assess)
4. If PROCEED: seal evidence, harvest gold set, move to next target
5. If not: iterate on assembly strategy, re-score
6. The gold set grows with each approved target. Next target has a higher bar.

### Coverage dashboard

At any point, the dashboard shows:

```
Legacy Oracle: 1000 tables

Scanned:           1000 / 1000  (100%)
Classified:         847 / 1000  ( 85%)  — alive/dead/uncertain
Mapped to target:   623 / 847   ( 74%)  — assigned to a data product
Proven equivalent:  412 / 623   ( 66%)  — evidence pack sealed
Remaining:          211 / 623   ( 34%)  — in progress or blocked

Dead tables:        153          — not queried in 2+ years, no active consumers
Uncertain:          153          — conflicting claims, needs human review

Data products:      24 targets
  Completed:        14 / 24     — all tables proven, evidence sealed
  In progress:       7 / 24     — tournament running
  Blocked:           3 / 24     — waiting on human resolution of conflicts
```

This is what the previous 5 retirement attempts didn't have: a precise, continuously-updated map of what's done, what's left, and what's blocking.

---

## Phase 4: Replay (Behavioral Equivalence)

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

`factory scan` captured Oracle's historical query patterns from `v$sql` and `dba_hist_sqlstat` — every query every application ran, with frequency and recency. `factory replay` turns those into an executable behavioral test suite:

```bash
factory replay --queries scan-results/queries/risk-app.sql \
  --oracle oracle://... \
  --twin localhost:5433 \
  --output replay-results/
```

For each historical query:
1. Run against Oracle (live, or cached result sets from scan)
2. Run the same SQL verbatim against Twin A
3. Compare result sets (row-for-row, column-for-column)
4. Classify: MATCH, MISMATCH, or SKIP (query uses unsupported SQL features)

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

## Phase 5: Prove + Seal

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

**Wave 2: Tournament (10-15 agents)**
Each agent works one data product. Boots its own twin pair (Twin A + Twin B), assembles candidate data, loads, scores, iterates.

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
┌──────────────────────────────────────────────────────────┐
│                    FACTORY                                │
│                                                          │
│   scan → claims → decode → twin → tournament → seal      │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                    SPINE                                  │
│                                                          │
│   vacuum hash fingerprint profile lock shape rvl         │
│   verify compare benchmark assess canon pack             │
│                                                          │
│   (works anywhere there is data)                         │
└──────────────────────────────────────────────────────────┘
```

When data flows from Oracle to a data product, it passes through the spine. When documents get re-parsed, the output goes through the spine. When a twin scores a candidate, it uses the spine. The factory orchestrates; the spine proves.

---

## Implementation order

1. **factory scan --db** — Database introspection. Highest confidence claims. Immediately useful.
2. **factory scan --repo** — Codebase scanning. Start with SQL string extraction (easy, high value), add ORM/model parsing iteratively.
3. **Claim format + storage** — JSONL files, content-addressed. No message bus.
4. **Decode v0** — Resolve structural claims (DDL + codebase). Surface conflicts. Build the map.
5. **Coverage dashboard** — Alive/dead/mapped/proven counts. The project management layer.
6. **Two twins per target** — Twin A (Oracle schema) + Twin B (new schema). Schemas from decoded claims. Rules from scan.
7. **Tournament loop** — Assemble, load, score, iterate. Spine tools do the scoring.
8. **factory replay** — Behavioral equivalence. Replay historical queries against Twin A. Compare result sets.
9. **Evidence sealing** — Pack per data product. Static proof + behavioral proof + assess decision.

Each step is independently useful. You don't need step 9 to get value from step 1.
