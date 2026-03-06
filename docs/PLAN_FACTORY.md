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

### Decoding

Decoding resolves competing claims into a canonical understanding:

- Two apps define different valid values for the same column → surface the conflict, let humans resolve
- DDL says nullable, app validator says required → the app-level rule is the real constraint (higher confidence than schema absence)
- Audit logs say table is dead, but a quarterly job still reads it → it's alive (quarterly)
- Three different apps JOIN through the same tables differently → all three usage patterns are real, map them all

The decoder doesn't pick one truth when there are legitimately multiple truths. A table used differently by three apps has three usage profiles. All three are claims. All three feed the migration plan.

**The output is a canonical map:** every table classified (alive/dead/uncertain), every column annotated (used by whom, constrained how, means what), every relationship traced (FK, JOIN pattern, proc flow), every gap flagged.

---

## Phase 3: Twin + Tournament

### Twinning the targets

The Oracle monolith doesn't get replaced by one system. It fragments into data products and marts — each purpose-built for a specific use case:

- Loan performance mart (for risk reporting)
- Deal structure mart (for portfolio analytics)
- Compliance data product (for regulatory reporting)
- Surveillance mart (for monitoring)
- ... dozens more

Each target gets a twin: an in-memory instance with its own schema and verify rules, derived from the decoded claims about what that slice of Oracle actually does.

```bash
# Boot a twin for the loan performance mart
twinning postgres --schema loan-perf-schema.sql --rules loan-perf-rules.json --port 5433

# Boot a twin for the compliance data product
twinning postgres --schema compliance-schema.sql --rules compliance-rules.json --port 5434
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

## Phase 4: Prove + Seal

### Evidence per data product

Each completed data product migration produces:

```
evidence/loan-performance-mart/
├── scan-claims.pack           # All claims about the source tables
├── decode-map.pack            # Canonical understanding of what these tables do
├── assembly-strategy.json     # How the data was assembled (Oracle + re-parsed docs)
├── benchmark.report.json      # Gold set accuracy
├── verify.report.json         # Rule compliance
├── compare.report.json        # Exhaustive diff vs Oracle source
├── rvl.report.json            # Materiality analysis
├── assess.report.json         # Policy decision: PROCEED
├── data.lock.json             # Content-addressed lockfile for the data
└── evidence.pack              # Sealed evidence bundle
```

### What the auditors see

Not "we migrated the database." Instead: "Here are 24 evidence packs, one per data product. Each one proves: what Oracle contained, what we assembled, how they compare, which rules passed, which risks were accepted and why, and the policy decision. Every number traces to a source. Every decision is reproducible."

That's why the previous attempts failed and this one won't. Not better technology — better proof.

---

## What we killed from the old plan

The original PLAN_FACTORY.md was 2000 lines. This is what we cut and why:

| Cut | Why |
|-----|-----|
| Agent swarm coordination | Premature. Agents iterate independently. Coordination is a scheduling problem, not an architecture. |
| Fountain code / RaptorQ analogies | Elegant metaphor, zero engineering value. Claims + scoring is the real pattern. |
| Event stream architecture (NATS, Redis Streams) | Premature infrastructure. Claims can be JSONL files for v0. |
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
6. **Twin per target** — One twin per data product. Schema from decoded claims. Rules from scan.
7. **Tournament loop** — Assemble, load, score, iterate. Spine tools do the scoring.
8. **Evidence sealing** — Pack per data product. The auditor deliverable.

Each step is independently useful. You don't need step 8 to get value from step 1.
