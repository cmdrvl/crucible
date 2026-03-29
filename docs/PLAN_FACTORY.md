# Crucible — Phase 1

## One-line promise
**Turn a legacy estate into a deterministic metadata catalog plus derived
archaeology claims, with enough coverage and provenance to support one bounded
replacement outcome.**

This repo is no longer carrying the whole end-state Crucible vision as an
implementation plan. Phase 1 is intentionally smaller:

- ship `crucible scan`
- hydrate the metadata catalog
- emit stable `claim.v0` files only where direct observation is insufficient
- support one real legacy outcome slice
- feed `decoding` only where convergence is actually needed

Everything else is deferred until we learn from real slices.

---

## What Phase 1 is

Phase 1 is the archaeology layer for bounded modernization work.

For Hyperion-style knockouts, the first job is not:

- decompose the full estate
- generate the full target architecture
- build twins
- run tournaments

The first job is:

1. inventory the relevant legacy surfaces
2. hydrate a deterministic metadata catalog from direct observations
3. derive claims only for ambiguous, inferential, or multi-source propositions
4. preserve provenance tightly enough that humans can resolve ambiguity
5. connect observations into evidence neighborhoods with per-subject readiness
6. hand only those derived claims to `decoding`

If Phase 1 does that well, one outcome slice can be scanned, understood, and
rebuilt with a mostly manual replacement loop.

---

## What Phase 1 is not

Phase 1 is NOT:

- `crucible decompose`
- `crucible transform`
- `crucible replay`
- `crucible tournament`
- `twinning` integration
- a general orchestration framework
- a message bus
- a full migration factory

Those may become later phases. They are not part of the current implementation
wedge.

---

## Target user story

For one bounded legacy outcome:

- a governed Hyperion export
- one close report family
- one reconciliation output
- one recurring management package

an operator should be able to run:

```bash
crucible scan db ...
crucible scan repo ...
crucible scan files ...
```

and receive:

- catalog records for applications, tables, files, jobs, reports, feeds,
  consumers, and lineage edges
- content-addressed claim files only for contested or inferential propositions
- an inventory of what was scanned
- a scan report showing coverage and gaps
- evidence neighborhoods linking catalog, claims, and inventory into a
  navigable graph with per-subject readiness classification
- enough evidence for `decoding` to converge only the parts of the slice that
  observation alone cannot settle

That is the Phase 1 bar.

---

## Required surfaces

Phase 1 must cover the four surfaces already identified in the Hyperion use
case:

1. configured surface
2. executed surface
3. human surface
4. consumption surface

Not every source inside those surfaces has to be parsed on day one. But the
scan layer must be able to ingest artifacts from all four classes and emit
catalog records, derived claims, or explicit "unparsed but inventoried" entries
with provenance.

---

## Catalog-first architecture

`crucible scan` should primarily behave like a metadata hydration engine, not a
claims-only feeder for `decoding`.

The default path is:

1. observe a thing directly
2. normalize it into the metadata/catalog model
3. attach provenance and lineage
4. only emit `claim.v0` if the proposition is inferred, ambiguous, or in
   tension with other evidence

That means `decoding` is not the hot path for plain scan facts.

Examples that should go straight into the catalog:

- table and column existence
- file and artifact inventory
- directly observed applications, jobs, reports, feeds, mappings, and
  consumers
- mechanically extractable lineage and dependency edges
- linked-node relationships recoverable without ambiguity

Examples that should produce `claim.v0` for `decoding`:

- valid-value sets inferred from multiple repos and exports
- liveness or staleness assessments
- authoritative-output claims
- semantic labels or business meaning hints
- dependency edges that are only weakly inferred or contradicted elsewhere

### Two-channel scanner contract

Every Phase 1 scanner should have exactly two semantic output channels:

1. **Catalog channel**
   Directly observed metadata normalized into the catalog model.
2. **Claim channel**
   Derived `claim.v0` records for propositions that need convergence.

The distinction is adapter-owned, not decoder-owned.

Scanner rule:

- if the adapter can recover the fact directly and mechanically from a source
  of record, write the catalog channel
- if the adapter must infer, merge, rank, reconcile, or interpret competing
  evidence, write the claim channel

This boundary should be visible in code. A scanner should not emit everything
as claims and leave the distinction to later tooling.

---

## Relationship to `cmdrvl-cli metadata`

Phase 1 should explicitly leverage the metadata model already exposed by
`cmdrvl-cli metadata`, rather than inventing a parallel catalog abstraction.

The existing metadata surfaces already cover the core scan targets:

- tables and columns
- files
- custom resources
- custom links
- linked-node traversal
- lineage graph nodes and edges

So `crucible scan` should treat `cmdrvl-cli metadata` as the primary catalog
shape for observed facts. In practice that means:

- DB objects hydrate table-oriented metadata records
- file and export inventories hydrate file metadata records
- applications, jobs, reports, feeds, mappings, and consumers hydrate custom
  resource records
- direct dependencies and lineage edges hydrate metadata links
- cross-surface neighborhoods and execution paths should be expressible through
  linked-node / lineage graph structures

`crucible` may stage local artifacts first, but the model it targets should
remain aligned with `cmdrvl-cli metadata`.

### Use the metadata substrate directly

`crucible scan` should not treat the `cmdrvl-cli metadata` shell commands as
its primary implementation strategy.

Preferred implementation order:

1. target the same metadata models
2. call the metadata API directly or reuse the underlying metadata client where
   practical
3. use CLI command wrapping only as a bootstrap or manual fallback

The important part is not whether the code lives in the same repo or process.
The important part is that `crucible scan` writes into the existing metadata
shape instead of inventing a second catalog.

---

## Phase 1 commands

Phase 1 has exactly one top-level command family:

```text
crucible scan <SUBCOMMAND> [OPTIONS]
```

Subcommands:

```text
crucible scan db
crucible scan repo
crucible scan files
```

### `crucible scan db`

Purpose: inspect a legacy database or exported database metadata and emit
catalog records plus derived claims about structure, usage hints, and liveness
evidence.

Minimum inputs:

- live connection string OR exported metadata files
- optional include/exclude filters for schemas and tables

Minimum outputs:

- metadata records for tables, columns, and direct lineage/dependency edges
- custom resource / link hydration where the DB scan can observe it directly
- `claim.v0` only for ambiguous usage/liveness propositions
- scan report summarizing covered objects and blind spots

### `crucible scan repo`

Purpose: inspect codebases and adjacent application assets that touch the
legacy system.

Minimum Phase 1 coverage:

- SQL string extraction
- connection/config discovery
- ORM/model metadata where obvious
- report definition files where parseable
- tests and enum/constants files where they express semantic or valid-value
  hints

Minimum outputs:

- metadata records for directly observed applications, reports, feeds,
  consumers, and dependencies
- `claim.v0` for inferred reads, writes, valid values, semantic labels, and
  consumer dependencies that require interpretation
- file-level provenance for every emitted claim

### `crucible scan files`

Purpose: inventory and parse local artifact directories around the legacy
outcome slice.

Typical inputs:

- metadata exports
- load rules
- calc scripts
- scheduler exports
- logs
- output extracts
- downstream deliverables
- spreadsheets and templates

Minimum outputs:

- metadata records where adapters exist
- `claim.v0` where the adapter yields only inferential or contested
  propositions
- inventory records plus provenance when adapters do not exist yet
- scan report listing parsed vs inventoried-only artifacts

---

## Phase 1 output model

Phase 1 has two output classes:

1. **Catalog hydration artifacts**
   Direct observed metadata normalized toward the `cmdrvl-cli metadata` model.
2. **Derived claim artifacts**
   `claim.v0` records for propositions that need convergence.

Observed metadata does not need `decoding`.

### Catalog hydration targets

The Phase 1 scan layer should be able to populate or stage data equivalent to:

- table metadata
- file metadata
- custom resources for applications, jobs, reports, feeds, mappings, consumers
- custom links between those nodes
- lineage graph nodes and edges

The exact write path may be:

- direct metadata API upserts
- deterministic staged JSON artifacts that `cmdrvl-cli metadata` can ingest
- a local catalog snapshot file that mirrors the metadata shapes

But the target model should remain aligned with `cmdrvl-cli metadata`.

### Catalog vs claim examples

Examples of catalog channel outputs:

- `Table` / `TableInput`-shaped records for DB objects
- `FileResponse` / `FileInput`-shaped records for files and exports
- custom `ResourceInput`-shaped records for applications, jobs, reports,
  feeds, mappings, and consumers
- `LinkInput`-shaped records for direct dependencies
- lineage graph node/edge structures for observed paths

Examples of claim channel outputs:

- `claim.v0` for inferred `valid_values`
- `claim.v0` for `liveness`
- `claim.v0` for `semantic_label`
- `claim.v0` for `authoritative_for`
- `claim.v0` for weak or conflicting dependency edges

### `claim.v0`

Phase 1 must emit one stable JSONL contract that `decoding` can consume
directly.

Required top-level fields:

```json
{
  "event": "claim.v0",
  "claim_id": "sha256:...",
  "source": {
    "kind": "db_scan",
    "scanner": "crucible.scan.db@0.1.0",
    "artifact_id": "sha256:...",
    "locator": {
      "kind": "db_object",
      "value": "fin.gl_actuals.amount"
    }
  },
  "subject": {
    "kind": "column",
    "id": "fin.gl_actuals.amount"
  },
  "property_type": "valid_values",
  "value": ["posted", "unposted"],
  "confidence": 0.92
}
```

Rules:

- `claim_id` is content-addressed from normalized payload
- every claim carries a concrete provenance pointer
- confidence is scanner-owned, not model-inferred
- no hidden enrichment

`claim.v0` is a secondary output of scan, not the only output of scan.

### Frozen Phase 1 field contract

Phase 1 should treat the following fields as frozen:

| Field | Type | Rules |
|-------|------|-------|
| `event` | string | exactly `claim.v0` |
| `claim_id` | string | `sha256:<64 lowercase hex>` |
| `source.kind` | enum | `db_scan`, `repo_scan`, `file_scan` |
| `source.scanner` | string | `<scanner-name>@<version>` |
| `source.artifact_id` | string | `sha256:<64 lowercase hex>` for the scanned artifact or exported metadata unit |
| `source.locator.kind` | enum | `db_object`, `file_range`, `export_row`, `log_line`, `config_key`, `manual_note` |
| `source.locator.value` | string | scanner-specific stable locator |
| `subject.kind` | enum | frozen Phase 1 subject vocabulary |
| `subject.id` | string | normalized stable identifier for the subject |
| `property_type` | enum | frozen Phase 1 archaeology vocabulary |
| `value` | JSON | normalized by `property_type` |
| `confidence` | number | `0.0` to `1.0`, inclusive |

Phase 1 should not add optional top-level fields. If a scanner has extra local
detail, it belongs in the evidence artifact or inventory output, not in the
shared claim wire format.

### Subject identifiers

`subject.kind` is fixed to:

```text
table | column | view | procedure | job | report | feed | artifact | consumer | mapping
```

`subject.id` should follow these conventions:

| Kind | ID convention | Example |
|------|---------------|---------|
| `table` | `<schema>.<table>` | `fin.gl_actuals` |
| `column` | `<schema>.<table>.<column>` | `fin.gl_actuals.amount` |
| `view` | `<schema>.<view>` | `fin.v_close_pack` |
| `procedure` | `<schema>.<procedure>` | `fin.refresh_close_pack` |
| `job` | `<system>.<job_name>` | `autosys.fdmee_load_actuals` |
| `report` | `<system>.<report_name>` | `hyperion.close_pack_ebitda` |
| `feed` | `<system>.<feed_name>` | `fdmee.actuals_load` |
| `artifact` | normalized relative path or export key | `exports/hfm/rules/account_map.csv` |
| `consumer` | `<system>.<consumer_name>` | `finance.board_pack_mailer` |
| `mapping` | `<domain>.<mapping_name>` | `accounts.ebitda_adjustments` |

DB-like identifiers should be lowercased. Paths should use `/` separators. If
the original system is case-sensitive or quoted, preserve the original spelling
in the evidence artifact, not the normalized `subject.id`.

### `value` normalization by property type

Phase 1 freezes the following `value` shapes:

| Property type | `value` shape |
|---------------|---------------|
| `exists` | `true` |
| `schema` | normalized object containing only scalar/array metadata relevant to the subject |
| `constraint` | object: `{"kind":"not_null\|pk\|fk\|check\|unique","detail":{...}}` |
| `reads` | subject ref object: `{"kind":"table","id":"fin.gl_actuals"}` |
| `writes` | subject ref object |
| `depends_on` | subject ref object |
| `used_by` | subject ref object |
| `schedule` | object: `{"kind":"cron\|event\|manual\|unknown","value":"...","timezone":"..."}` |
| `valid_values` | sorted array of strings |
| `semantic_label` | normalized string |
| `liveness` | one of `alive`, `stale`, `unknown`, `dead` |
| `authoritative_for` | subject ref object |

A Phase 1 subject ref object is always:

```json
{
  "kind": "report",
  "id": "hyperion.close_pack_ebitda"
}
```

Raw SQL text, report definitions, stack traces, and other bulky evidence stay
behind the `artifact_id` + `locator` boundary. The claim record carries only
the normalized proposition.

### Claim normalization rules

The claim contract should be reproducible byte-for-byte:

- no timestamps, hostnames, usernames, or run-local counters inside
  `claim.v0`
- object keys serialized in sorted order before hashing
- arrays sorted and de-duplicated when order is not semantically meaningful
- strings trimmed; repeated internal whitespace collapsed only where scanners
  declare the value to be free text
- `claim_id` computed from canonical JSON of every field except `claim_id`
- any proposition outside the frozen subject/property vocabulary must be
  inventoried, not emitted as `claim.v0`

### Output layout

```text
scan-results/
├── catalog/
│   ├── tables.json
│   ├── files.json
│   ├── resources.json
│   ├── links.json
│   └── lineage.json
├── claims/
│   ├── db.claims.jsonl
│   ├── repo.claims.jsonl
│   └── files.claims.jsonl
├── inventory/
│   ├── db.inventory.json
│   ├── repo.inventory.json
│   └── files.inventory.json
├── neighborhoods/
│   ├── nodes.json
│   └── edges.json
└── scan-report.json
```

The exact filenames can vary, but this five-part shape may not:

- catalog
- claims
- inventory
- neighborhoods
- report

---

## Evidence neighborhoods

A scan produces flat outputs — catalog records, claims, inventory entries. But
the value of scanning comes from the connections between outputs from different
scanners.

Every subject discovered by scan has an **evidence neighborhood**: the connected
subgraph of all catalog records, claims, inventory entries, and lineage edges
that relate to it across all scan surfaces.

### Why neighborhoods are a Phase 1 concern

The evidence neighborhood of the target outcome IS the replacement scope.

Walking outward from `hyperion.close_pack_ebitda`:

- the report references views → `v_close_pack`
- the view joins tables → `gl_actuals`, `mapping_table`
- repo scan found code reading those tables → `close_report.java`
- that code references a load job → `fdmee.actuals_load`
- file scan found scheduler exports for that job

That connected subgraph is the complete archaeological context for the outcome
slice. Every node reachable from the target is in-scope for replacement.
Everything unreachable is out.

### Per-node readiness classification

Every node in an evidence neighborhood carries a readiness signal derived from
its evidence quality:

| Classification | Meaning | Derived from |
|---------------|---------|--------------|
| `observed` | Fully in catalog, no contested claims | Catalog records only |
| `contested` | In catalog but has open claims needing convergence | Catalog + claims |
| `inventoried` | Scanned but not parsed | Inventory records only |
| `referenced` | Mentioned by other nodes but never directly scanned | Inferred from edges |

The readiness classification is computed, not scanner-assigned. It falls out of
which output channels contain records for a given subject.

### Neighborhood as composition primitive

The three scan subcommands produce independent outputs. The evidence
neighborhood is how they compose:

- `scan db` nodes connect to `scan repo` nodes through shared table/view
  references
- `scan repo` nodes connect to `scan files` nodes through config paths and
  artifact references
- `scan files` nodes connect to `scan db` nodes through export/load
  relationships

No manifest file needed. The graph is the composition.

### Neighborhood scope for `decoding`

`decoding` should receive claims in the context of their evidence neighborhood,
not as flat JSONL. A liveness claim about `fin.gl_actuals` is dramatically
easier to converge when it arrives alongside the catalog records showing which
code reads it, which jobs load it, and which reports consume it.

### Neighborhood output

The neighborhood graph is a computed artifact built from the combined outputs of
all scanners. It should be deterministic and content-addressed like every other
scan output.

The graph should be emittable as:

- a node list with readiness classification per node
- an edge list with edge types (dependency, lineage, reference)
- per-neighborhood summary (node count by readiness, edge count by type)

The scan report should surface neighborhood topology alongside flat counts.

---

## Phase 1 adapters

The scan layer needs a small adapter model, not a giant plugin framework.

Day-one adapters should be hand-written and explicit. Good Phase 1 candidates:

- Oracle / SQL Server / Postgres metadata adapter
- SQL literal extractor for Java and Python
- config / connection string extractor
- filesystem inventory adapter
- simple log / scheduler export adapter

Anything more elaborate should wait until repeated need is proven.

---

## Phase 1 exit criteria

Crucible Phase 1 is implementation-ready when the plan supports shipping:

1. `crucible scan db`
2. `crucible scan repo`
3. `crucible scan files`
4. stable catalog hydration output aligned with `cmdrvl-cli metadata`
5. stable `claim.v0`
6. scan report with coverage / blind spots
7. evidence neighborhoods with per-subject readiness classification
8. deterministic reruns for identical inputs

Crucible Phase 1 is functionally successful when one real legacy outcome slice
can be scanned across all relevant surfaces, the directly observed metadata can
hydrate the catalog cleanly, the evidence neighborhood of the target outcome
is navigable with clear readiness classification, and the resulting derived
claims are good enough for `decoding` to produce a useful first canonical map
where needed.

---

## Phase 1 implementation checklist

Crucible is Phase 1 implementation-ready when this checklist is concrete enough
to code without reopening design questions:

1. **CLI shell**
   - `crucible scan db`
   - `crucible scan repo`
   - `crucible scan files`
   - shared output directory and report writer wiring

2. **Shared claim contract module**
   - frozen enums for `source.kind`, `subject.kind`, and `property_type`
   - canonical JSON serializer
   - `claim_id` builder
   - subject-id normalizer
   - subject-ref/value helpers

3. **Catalog writer**
   - deterministic catalog records aligned with `cmdrvl-cli metadata`
   - resource/link staging for non-table scan entities
   - lineage graph staging
   - provenance attachment for observed facts
   - direct metadata API or shared-client path preferred over CLI shell-out

4. **Inventory writer**
   - deterministic inventory records for scanned inputs
   - `artifact_id` generation
   - parsed-vs-inventoried-only classification

5. **`scan db` v0**
   - table / column / constraint extraction
   - view / procedure inventory
   - accessible job metadata extraction
   - direct catalog hydration for observed DB objects
   - provenance locators for every emitted claim

6. **`scan repo` v0**
   - SQL literal extraction for Java and Python
   - config / connection discovery
   - catalog hydration for directly observed apps / reports / feeds / consumers
   - enum / constant scanning for `valid_values` and `semantic_label`
   - file-range provenance

7. **`scan files` v0**
   - recursive inventory
   - high-value adapters for scheduler exports, logs, and structured extracts
   - catalog hydration for parseable file/report/export artifacts
   - inventory-only fallback when parsing is not implemented

8. **Evidence neighborhood builder**
   - node list aggregated from catalog records, claims, and inventory entries
   - edge list derived from lineage edges, dependency links, and subject
     references across scan surfaces
   - per-node readiness classification (observed / contested / inventoried /
     referenced)
   - neighborhood traversal from a seed subject outward
   - deterministic, content-addressed graph output

9. **Scan report**
   - catalog counts by node / edge type
   - claim counts by source kind and property type
   - inventoried-only counts
   - neighborhood topology: node count by readiness classification, edge count
     by type, reachable subgraph from target outcome
   - blind-spot summary
   - subject coverage summary

10. **Determinism tests**
    - identical input rerun produces identical catalog, claims, neighborhoods,
      and report
    - mixed-source fixture test for one bounded legacy slice
    - normalization tests for subject IDs and claim hashing
    - classification tests proving observed facts stay on catalog channel and
      inferred facts become claims
    - neighborhood stability: same inputs produce identical graph structure and
      readiness classifications

Phase 1 coding should start only after the catalog target model, claim
boundary, and neighborhood graph schema are frozen.

---

## Relationship to `decoding`

Crucible Phase 1 stops at catalog hydration, evidence neighborhoods, and
derived claims.

```text
legacy estate
  -> crucible scan
  -> metadata catalog / lineage / inventory
  -> evidence neighborhoods with per-subject readiness
  -> optional claim.v0 JSONL for ambiguous or inferential propositions
  -> decoding archaeology mode only where needed
```

Crucible does not decide truth in Phase 1. It discovers and records evidence.
Most directly observed metadata should bypass `decoding` and land in the
catalog. Evidence neighborhoods connect observations across scan surfaces and
classify readiness per subject. `decoding` converges only the parts of the
evidence stream that are actually claim-resolution problems, receiving claims
in the context of their surrounding neighborhood.

---

## Hyperion slice #1 readiness

We should attempt the first Hyperion slice once all of the following are true:

- one bounded outcome is chosen
- the relevant database surface can be scanned
- the relevant repos can be scanned
- surrounding artifacts can at least be inventoried, even if not all are
  parsed yet
- scan outputs can hydrate the metadata catalog without ad hoc translation
- evidence neighborhoods are navigable from the target outcome to its full
  dependency boundary
- claim output is stable and deterministic
- `decoding` can consume the derived claim files without translation glue

That is enough to start learning. It is not enough to automate the whole
program, and it does not need to be.
