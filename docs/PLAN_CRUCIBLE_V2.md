# Crucible v2 — Architecture Plan

> Supersedes the Phase 1 scanner-centric design in PLAN_FACTORY.md.
> This plan reframes crucible as a skill-first archaeology system with a Rust
> substrate, structured concurrency via asupersync, and a clean boundary to
> decoding.

## Core thesis

Crucible is the archaeology brain. Its job is to discover what exists in an
environment, collect evidence from multiple sources, and produce structured
claims about what it found. It does not resolve conflicting claims (that's
decoding) and it does not own the metadata catalog (that's cmdrvl-cli).

## Architecture layers

```
┌─────────────────────────────────────────────────────────┐
│  crucible skill                                         │
│  (orchestration, routing, LLM interpretation, loops)    │
│  invokes sub-skills: code-archaeology, doc-parse, etc.  │
├─────────────────────────────────────────────────────────┤
│  crucible rust crate                                    │
│  (asupersync regions, claim contract, convergence       │
│   engine, subject-identity normalization, evidence      │
│   neighborhoods, assessment output)                     │
├─────────────────────────────────────────────────────────┤
│  asupersync                                             │
│  (structured concurrency, cancel-correct drain,         │
│   capability security, deterministic testing)           │
└─────────────────────────────────────────────────────────┘

Optional integrations (not dependencies):
  → cmdrvl-cli metadata  (catalog read/write via shell-out)
  → decoding             (claim resolution when conflicts exist)
```

### Layer responsibilities

**Crucible skill** (top):
- Decides which detector skills to invoke for a given target environment
- Orchestrates loops, multi-pass investigation, swarm coordination
- Handles LLM-assisted interpretation where needed
- Produces the final assessment report
- This is the user-facing entry point

**Crucible Rust crate** (middle):
- Asupersync-based execution engine for parallel detector regions
- `claim.v0` contract: content-addressed claims with confidence and provenance
- Subject-identity normalization: merge signals about the same entity
- Corroboration scoring: multiple independent signals agreeing raises confidence
- Evidence neighborhood graph: subjects, edges, readiness classification
- Assessment output: structured report of findings, agreements, and conflicts
- Deterministic: same inputs, same detector outputs → same assessment

**Asupersync** (bottom):
- Structured concurrency: detector regions own their tasks, close to quiescence
- Cancel-correct drain: when one path finishes or is superseded, others drain
  with bounded cleanup — the drain produces evidence, not silence
- Two-phase effects: no data loss on cancellation
- Capability security: detectors only have access to what they're given
- Lab runtime: deterministic testing of concurrent detector behavior

## LLM boundary: the spine-tool contract

No spine tool calls an LLM. Vacuum, hash, fingerprint, decoding — all
deterministic, offline-capable, no API key required. Crucible's Rust crate
preserves this property.

**Rule: The Rust crate never calls an LLM. The LLM lives in the skill layer.**

Where LLM intelligence is needed, the skill does the thinking and passes the
result to the crate as an explicit, deterministic input:

| Intelligence needed | Who does it | What the crate receives |
|--------------------|-------------|------------------------|
| Subject-identity merge ("these are the same endpoint") | Skill (LLM) | Merge proposal: `merge(subject_a, subject_b, reason, confidence)` |
| Interpret ambiguous evidence | Skill (LLM) | A claim with `source.kind: llm_interpretation` and provenance |
| Route decision ("gap found, spin up another detector") | Skill (LLM) | A new detector invocation (crate just spawns the region) |
| Classify conflict severity | Skill (LLM) | Annotation on the conflict record |
| Generate human-readable summary | Skill (LLM) | Post-processing of the assessment output |

The crate's contract:
- Receives claims (from detectors or from the skill's LLM interpretations)
- Executes merge proposals (deterministically — same proposal → same result)
- Scores corroboration (counting, not judging)
- Detects conflicts (structural comparison, not semantic interpretation)
- Produces assessment output (structured data, not prose)

This means:
- `crucible` the crate can run without network access or API keys
- Same detector outputs + same merge proposals = same assessment (reproducible)
- The skill layer is optional — you can feed claims manually for testing
- No precedent broken: spine tools remain deterministic substrates

The skill layer is already LLM-powered by definition (skills are agent-
orchestrated). The LLM intelligence that crucible needs is not a new dependency
— it's the skill runtime that was always going to be there.

## Three jobs, three systems

| Job | System | Question it answers |
|-----|--------|-------------------|
| Run parallel detectors with lifecycle discipline | asupersync | "How do we execute concurrent investigation safely?" |
| Collect, normalize, and corroborate claims | crucible | "What exists, and what do we believe about it?" |
| Resolve conflicting claims via policy | decoding | "When claims disagree, which one wins?" |

These compose but don't overlap. Crucible does not resolve conflicts — it
identifies them. Decoding does not discover subjects — it receives claims.
Asupersync does not understand claims — it manages task lifecycle.

## System boundaries

Crucible doesn't just discover individual subjects — it discovers **where the
boundaries are**. A boundary is a group of subjects that belong together as a
coherent system: they deploy together, fail together, serve the same business
function, or share an ownership/trust perimeter.

### Why boundaries matter

In any sufficiently complex environment, there are multiple systems running.
Crucible needs to produce an opinion on what's inside each boundary:

> "This system has one web app, two APIs, three queues, a database, and a
> cron job. They deploy from the same repo, live in the same namespace, and
> call each other. Here's the map."

Without boundaries, crucible produces a flat list of subjects. With boundaries,
it produces a **system map** — which is what operators actually need to reason
about replacement scope, failure domains, and ownership.

### Boundary detection signals

Boundaries are themselves a convergence problem. Signals that subjects belong
together:

| Signal | Source | Strength |
|--------|--------|----------|
| Same deploy pipeline / same repo | git-scan, CI/CD configs | Strong |
| Same namespace / VPC / cluster | kubernetes, cloud provider | Strong |
| Call each other (dependency edges) | code-archaeology, network-observe | Strong |
| Same team owns them | CODEOWNERS, Jira/Linear, org configs | Medium |
| Same domain/subdomain family | dns-scan, certificate-scan | Medium |
| Same credentials / auth boundary | config-scan, secret manager | Medium |
| Correlated failures | datadog, sentry, logs | Medium |
| Co-mentioned in documentation | doc-parse, confluence, notion | Weak |
| Same naming convention / prefix | heuristic | Weak |

### Boundary hierarchy

Boundaries nest:

```
estate (everything we can see)
  └── system A (web app + API + database + queue)
  │     └── subsystem A1 (the API cluster specifically)
  └── system B (batch pipeline + scheduler + data warehouse)
  └── system C (third-party integration — DUS gateway + client adapter)
```

A scan might start scoped to one boundary ("tell me about system A") or start
broad ("scan the estate and tell me what systems you find"). Both are valid.

### Boundary representation

A boundary is a first-class object in the assessment output:

```yaml
boundary:
  id: "boundary:client-loan-submission"
  name: "Loan Submission System"
  confidence: 0.85
  detection_signals:
    - {signal: same_repo, source: git-scan, subjects: [api, worker, schema]}
    - {signal: same_namespace, source: kubernetes, subjects: [api, worker]}
    - {signal: dependency_edge, source: code-archaeology, pairs: [[api, queue], [worker, db]]}
  subjects:
    - {id: "app:loan-api", role: "API gateway"}
    - {id: "app:loan-worker", role: "async processor"}
    - {id: "db:loan-postgres", role: "primary datastore"}
    - {id: "queue:loan-submissions", role: "async dispatch"}
    - {id: "ext:dus-gateway", role: "external dependency (Fannie Mae)"}
  edges:
    - {from: loan-api, to: loan-submissions, type: publishes}
    - {from: loan-worker, to: loan-submissions, type: consumes}
    - {from: loan-worker, to: loan-postgres, type: reads_writes}
    - {from: loan-api, to: dus-gateway, type: calls}
  parent_boundary: "estate:client-prod"
```

### Boundaries and evidence neighborhoods

- An **evidence neighborhood** is per-subject: all evidence about one thing
- A **boundary** is per-system: which subjects form a coherent unit
- Boundaries are built FROM neighborhoods — if two subjects share enough edges
  and co-occurrence signals, they're likely in the same boundary

The relationship: neighborhoods are the raw graph, boundaries are the
clustering of that graph into meaningful system units. The clustering is itself
a claim — "I believe these subjects form a system" — with confidence and
provenance from the signals that suggested it.

### Boundary detection as a convergence problem

Like subject detection, boundary detection uses the asupersync pattern:
- Multiple detectors independently suggest groupings
- Crucible merges proposals and scores confidence
- High-confidence boundaries are promoted to the system map
- Low-confidence groupings are noted as tentative
- The skill layer can propose boundary merges (LLM-assisted) when
  deterministic signals are ambiguous

### Implications for profiles and scope

Profiles can declare boundary expectations:

```yaml
# "I know I'm looking at one system, tell me what's in it"
scope:
  boundary_mode: single_system
  entry_points: ["app:loan-api"]  # start here, follow edges

# "Scan broadly and tell me what systems you find"
scope:
  boundary_mode: discover
  entry_points: []  # look everywhere within scope
```

## Forensics properties

Crucible is a forensics tool. These properties are non-negotiable in the design.

### Evidence integrity

- Every finding traces to raw source: which detector, what it observed, when
- Claims are content-addressed (SHA256) — tamper-evident by construction
- The assessment itself is hashable: same inputs → same hash
- Evidence is append-only after collection — never modified, only annotated

### Non-destructive observation

- Crucible never mutates the system it examines. Read-only by default.
- Active probes (send a test request, run a query) require an explicit
  capability grant in the detector's manifest — not a default
- If a detector needs write/execute access, the profile must opt in and
  the assessment records that it was granted

### Chain of custody

- The tool's own execution is part of the record: what ran, what it accessed,
  what it produced, duration, exit status
- Scan initiation metadata: who started it, under what authority, with what
  scope, at what time
- Every merge proposal, corroboration decision, and conflict detection is
  logged with the reasoning inputs that produced it

### Scope control

- Explicit boundaries declared before scan: "examine THIS, not that"
- Crucible never wanders beyond declared scope
- The scope declaration is part of the assessment output — an auditor can
  verify what was in-bounds
- Out-of-scope signals are noted ("I saw evidence of X but it's outside my
  declared scope") but not pursued

### Temporal awareness

- Every observation carries a timestamp (when collected, not when processed)
- Assessment records scan window: start time, end time, per-detector timing
- Evidence freshness is explicit — "this claim is from 3 days ago" vs "just now"
- Re-scan support: delta detection between scan N and scan N+1

### Reporting layers

| Layer | Audience | Content |
|-------|----------|---------|
| Raw evidence | Machine / audit | Full-fidelity claim records, detector logs, drain evidence |
| Structured assessment | Developer / operator | Claims, convergence state, conflicts, neighborhoods |
| Human summary | Stakeholder / client | Prose report (skill-layer generated, optional) |
| Diff report | Operations | Delta vs catalog or vs prior scan |

### Doctor mode

Before running a scan, `crucible doctor` verifies:
- Required plugins installed and version-compatible
- Permissions sufficient for declared scope (can read target paths, ports, etc.)
- Dependencies available (asupersync runtime, cmdrvl-cli if catalog mode)
- Credential access confirmed for detectors that need it (cloud configs, etc.)
- Fail early and clearly — never produce a partial assessment silently

Doctor output is itself a structured record: what was checked, what passed,
what failed, what was skipped.

## Plugin architecture

Crucible is a platform for archaeology, not a fixed set of scanners. New
surfaces, new environments, new engagement types are all handled by adding
plugins — never by modifying the core.

### Plugin contract

Every detector plugin declares:

```yaml
# Example: detector-plugin.yaml
name: process-scan
version: 0.1.0
description: "Scan running processes and listening ports on the local machine"

# What the plugin needs to run
capabilities_required:
  - filesystem.read:/proc
  - command.execute:lsof
  - command.execute:ps

# What it produces
claim_types:
  - subject.kind: application
    property_types: [exists, listens_on, serves_domain]

# Scope compatibility — which profiles/targets can invoke this plugin
compatible_targets:
  - local_machine
  - remote_ssh

# Runtime constraints
estimated_duration: 5s–30s
idempotent: true
destructive: false
```

### Plugin categories

| Category | Examples | Notes |
|----------|----------|-------|
| Local system | process-scan, docker-scan, filesystem-scan | Runs on the machine being examined |
| Code analysis | git-scan, repo-structure, dependency-graph | Examines source code for evidence |
| Network/infra | dns-scan, port-scan, cloud-config, certificate-scan | Examines network and infrastructure surfaces |
| Document/artifact | doc-parse, pdf-extract, config-file-scan | Extracts claims from documentation and artifacts |
| History/logs | browser-scan, error-log-scan, audit-log-scan | Evidence from temporal records |
| Integration | api-probe, webhook-test, auth-flow-test | Active probes (require explicit capability grant) |

### Plugin lifecycle

```
discover → install → doctor-check → configure → invoke → collect → drain
```

- **Discover**: registry of available plugins (local, remote, bundled)
- **Install**: fetch/build plugin if not present
- **Doctor-check**: verify capabilities are satisfiable before scan starts
- **Configure**: profile passes target-specific config (paths, URLs, credentials)
- **Invoke**: crucible spawns the plugin in an asupersync region
- **Collect**: plugin emits claims into the region's channel
- **Drain**: plugin finishes or times out; drain evidence recorded

### Plugin isolation

- Each plugin runs in its own asupersync region — cannot affect other plugins
- Capability security: a plugin only has access to what its manifest declares
  and the profile grants
- A plugin crash/timeout is contained: other detectors continue, the failure
  is recorded as evidence ("this detector failed, here's why")
- No shared mutable state between plugins — communication is via claims only

### Extensibility paths

| Extension type | What you add | Core changes needed |
|---------------|--------------|-------------------|
| New detector | Plugin manifest + implementation | None — plugin registry picks it up |
| New output adapter | Output formatter (catalog, CSV, PDF, etc.) | None — adapter interface |
| New merge strategy | Subject-identity rule module | Register in normalizer |
| New profile | YAML profile selecting plugins + config | None — profile loader |
| New claim type | Extend claim.v0 schema | Schema version bump (additive only) |

### Profiles

Profiles are named configurations that select plugins, set scope, and tune
behavior for a specific engagement type:

```yaml
# Example: profiles/laptop-audit.yaml
name: laptop-audit
description: "Find applications and services on a developer workstation"
target: local_machine

plugins:
  - process-scan
  - git-scan
  - dns-scan
  - cloud-config-scan
  - browser-scan
  - docker-scan

scope:
  include:
    - filesystem: ["~", "/etc", "/opt"]
    - processes: all
    - git_repos: ["~/Source", "~/Developer"]
  exclude:
    - filesystem: ["~/.ssh", "~/.gnupg"]

options:
  catalog_diff: optional    # shell out to cmdrvl-cli if available
  report_format: structured # raw | structured | human
  max_duration: 120s
```

```yaml
# Example: profiles/api-archaeology.yaml
name: api-archaeology
description: "Reverse-engineer an undocumented API from available evidence"
target: remote_api

plugins:
  - code-archaeology    # skill-backed
  - doc-parse           # skill-backed
  - config-scan
  - error-log-scan
  - network-observation

scope:
  include:
    - codebase: "{provided_path}"
    - documentation: "{provided_urls}"
    - logs: "{provided_log_path}"
  exclude: []

options:
  active_probes: false  # no live API calls unless explicitly enabled
  llm_merge: true       # skill layer may propose subject merges
  max_duration: 600s
```

### Plugin development

A minimal plugin is:
1. A manifest (YAML declaring capabilities, claim types, constraints)
2. An executable (binary, script, or skill reference) that:
   - Reads its configuration from stdin or a config file
   - Emits claims as JSONL to stdout (or a named pipe/channel)
   - Exits with a status code
   - Respects cancellation signals (for asupersync drain protocol)

This keeps the bar low for new plugins while the manifest ensures the core
can reason about what the plugin does without running it.

## Flywheel Connectors integration

Crucible does not build its own connectors to external services. It layers on
top of Flywheel Connectors (`Source/flywheel/flywheel_connectors`), which
provides 150+ pre-built connectors with a uniform protocol (FCP v3).

### Why this matters

Any real archaeology will need to reach into services: read from a database,
pull configs from cloud providers, query an API, scan a git host, read from
object storage. Flywheel Connectors already solves this — with the same
asupersync-native execution model, capability-scoped authority, zone isolation,
and durable evidence chains that crucible needs.

### What flywheel_connectors provides

- **150+ connectors**: AWS, GCP, Azure, GitHub, GitLab, Salesforce, Snowflake,
  PostgreSQL, MySQL, MongoDB, Slack, Jira, Confluence, Google Drive, S3,
  Elasticsearch, Datadog, Sentry, Netlify, Vercel, and many more
- **FCP protocol (v3)**: capability-scoped, region-owned, budgeted execution
  that closes to quiescence — asupersync-native
- **Zone-first isolation**: every connector instance is bound to a zone with
  explicit authority
- **Durable evidence**: every action produces inspectable receipts and audit
  chains
- **No ambient authority**: credentials and permissions are explicit grants,
  not environment leakage

### How crucible uses it

Crucible detectors that need external service access don't implement their own
API clients. They declare a flywheel connector dependency in their manifest:

```yaml
# Example: detector that needs to read from PostgreSQL
name: database-schema-scan
capabilities_required:
  - connector: postgresql
    operations: [query, describe_tables, describe_columns]
    zone: "{target_zone}"
```

At runtime, crucible's plugin loader:
1. Checks that the required connector is installed and configured
2. Provisions a scoped connector instance (read-only, bounded, audited)
3. Passes the connector handle to the detector's asupersync region
4. The detector uses FCP operations to collect evidence
5. All access is logged as part of the chain of custody

### Design constraints

- **Crucible assumes a functioning flywheel_connectors installation** for any
  detector that needs external service access
- **Doctor mode checks connector availability** — if a profile requires the
  PostgreSQL connector but it's not installed/configured, doctor fails early
- **Crucible never talks to external services directly** — always through FCP
- **Local-only detectors** (process-scan, filesystem-scan) don't need
  connectors and work without flywheel_connectors installed
- **The connector layer is the credential boundary** — crucible never handles
  raw credentials; FCP manages credential injection and rotation

### Implication for profiles

Profiles declare connector dependencies alongside plugin dependencies:

```yaml
name: cloud-estate-audit
plugins:
  - aws-resource-scan
  - github-repo-scan
  - postgresql-schema-scan

connectors_required:
  - aws: {zone: client-prod, operations: [describe_*]}
  - github: {zone: client-org, operations: [list_repos, read_repo]}
  - postgresql: {zone: client-db, operations: [query, describe_*]}
```

This makes the engagement's external access requirements explicit, auditable,
and scope-controlled before anything runs.

## Two modes of value

### Standalone (no catalog)

Crucible produces an **archaeology assessment**: a structured, evidence-backed
report of what was found in an environment. This has independent value as a
deliverable — a consultant can hand it to a client and say "here's what we
found, here's what's uncertain, here's what conflicts."

Output: assessment report + claims + evidence neighborhoods.

### With catalog (cmdrvl-cli integration)

Crucible shells out to `cmdrvl-cli metadata` to:
1. **Read** what the catalog already knows (for diff/gap detection)
2. **Write** high-confidence findings that should be registered
3. **Detect drift** between what crucible finds and what the catalog says

The catalog is an optional output adapter, not a hard dependency.

## Claim lifecycle

```
detector emits raw signal
  → crucible normalizes subject identity
    → crucible groups claims by subject
      → claims agree? → high-confidence finding (catalog-ready)
      → claims conflict? → unresolved, available for decoding
      → single-source claim? → low-confidence, noted with provenance
```

Every claim carries:
- `claim_id`: content-addressed SHA256
- `source`: which detector, which skill, which run
- `subject`: normalized identity of the thing being described
- `property`: what's being asserted
- `confidence`: 0.0–1.0, detector-owned
- `corroboration`: list of independent signals that agree (populated by crucible)

## Asupersync in crucible

Even simple detection is a convergence problem. A single signal (port open,
git repo exists, DNS record found) is weak. Confidence comes from multiple
independent detectors corroborating the same subject.

The execution pattern:

1. Spawn detector regions in parallel (one per surface type)
2. Each detector emits claims as it finds evidence
3. Crucible's convergence engine continuously merges and scores
4. When a detector finishes or times out, it drains with evidence
5. Drain output is itself a claim: "I looked here and found nothing" is signal
6. Region closes to quiescence — all detectors complete, all evidence captured

The drain-as-evidence property is why asupersync matters from day one. A
detector that ran but found nothing is different from a detector that never ran.
Both are recorded.

---

## Use Case: Find Missing Applications (Easy)

**Scenario**: Run crucible on a laptop or workstation. Detect applications and
services that are in active use but not registered in the SOAR metadata catalog.

**Target environment**: Developer laptop (macOS)

**Why it's hard enough for asupersync**: No single signal reliably identifies
"this is a live application." A port listener could be ephemeral. A git repo
could be abandoned. A DNS record could be stale. Confidence requires
corroboration from independent surfaces.

### Detector regions (parallel via asupersync)

| Detector | What it scans | Claims it produces |
|----------|--------------|-------------------|
| process-scan | `lsof`, `ps`, listening ports | "something is serving on port X at domain Y" |
| git-scan | local repos with deploy configs (GitHub Actions, Netlify, Vercel) | "repo X deploys to URL Y" |
| dns-scan | `/etc/hosts`, local DNS, cloud DNS configs | "domain X resolves to infrastructure Y" |
| cloud-config-scan | `~/.config/gcloud`, `~/.netlify`, Vercel configs, Cloudflare | "service X is configured in provider Y" |
| browser-scan | bookmarks, frequent history entries | "user regularly visits URL X" |
| docker-scan | running containers, compose files | "service X is defined/running in container Y" |

### Convergence

Subject identity normalization: multiple signals about the same app converge on
a canonical subject. Example:

```
process-scan:  "port 443 serving tranchelist.com"
git-scan:      "cmdrvl/tranche-list deploys to tranchelist.com via Netlify"
dns-scan:      "tranchelist.com → Netlify edge"
cloud-config:  "Netlify site tranche-list exists in ~/.netlify"
browser-scan:  "tranchelist.com visited 40x this month"
```

All five signals → same subject: `app:tranchelist.com`. Confidence: very high.

### Catalog diff (optional)

Shell out to `cmdrvl-cli metadata` to check: is `app:tranchelist.com`
registered? If not → gap finding.

### Output (standalone)

```
Assessment: laptop-scan-2026-05-17

Findings (high confidence — 3+ corroborating signals):
  - app:tranchelist.com     [5 signals, not in catalog]
  - app:cmdrvl.com          [4 signals, not in catalog]
  - app:cairn@cmdrvl.com    [3 signals, registered]

Tentative (1-2 signals, low confidence):
  - app:localhost:3000      [1 signal: process-scan only, likely ephemeral]

Detectors that ran:
  - process-scan:      completed, 12 claims
  - git-scan:          completed, 8 claims
  - dns-scan:          completed, 5 claims
  - cloud-config-scan: completed, 6 claims
  - browser-scan:      completed, 14 claims
  - docker-scan:       completed, 0 claims (no Docker running)
```

### What the Rust crate does here

- Spawns detector regions via asupersync
- Receives claims from each detector
- Normalizes subject identity ("tranchelist.com" from five sources → one subject)
- Scores corroboration (count of independent signals per subject)
- Produces the structured assessment
- Records drain evidence (docker-scan ran, found nothing — that's a claim too)

### What the skill layer does here

- Decides which detectors to run based on the target (macOS laptop → these six)
- Invokes each detector (some are simple scripts, some are sub-skills)
- Optionally shells out to `cmdrvl-cli metadata` for the catalog diff
- Formats the final report for human consumption

---

## Use Case: Fannie Mae DUS Gateway Reverse Engineering (Medium)

**Scenario**: Client uses Fannie Mae's DUS (Delegated Underwriting and
Servicing) gateway. No public API spec exists. We need to produce an
evidence-backed model of how the gateway works — endpoints, auth, data formats,
integration patterns — from whatever sources we can find.

**Why it's medium**: Multiple heterogeneous evidence sources, some requiring
LLM interpretation. Claims will frequently conflict or be partial. Subject
identity is harder (what counts as "the same endpoint"?). Some detector paths
require deep analysis (code archaeology is a full skill, not a simple script).

### Detector regions (parallel via asupersync)

| Detector | What it scans | Claims it produces |
|----------|--------------|-------------------|
| code-archaeology skill | Client's codebase that calls DUS | "endpoint X exists, takes params Y, returns shape Z" |
| doc-parse skill | Fannie Mae integration guides, PDFs, portal pages | "docs say endpoint X accepts format Y" |
| network-observation | Traffic logs, proxy captures if available | "observed request to X with headers Y, got response Z" |
| config-scan | Connection configs, env vars, credential vaults | "DUS base URL is X, auth method is Y" |
| error-log-scan | Application error logs referencing DUS | "DUS returned error Z when sent payload W" |

### Convergence

Subject identity is harder here. Crucible needs to determine: "the endpoint
that code calls `/api/v2/loans/submit` and the one docs call 'Loan Submission
Service' — are these the same thing?"

This is where LLM-assisted interpretation (skill layer) feeds back into the
Rust crate's subject normalizer. The skill proposes a merge; the crate records
it as a claim with provenance.

Example convergence:

```
code-archaeology:  "POST /api/v2/loans/submit, body: JSON, auth: mTLS + X-DUS-Token header"
doc-parse:         "Loan Submission endpoint accepts XML or JSON, authenticates via OAuth2"
network-observe:   "saw POST to /api/v2/loans/submit with mTLS + custom header, JSON body"
config-scan:       "DUS_AUTH_METHOD=mtls_token in .env"
error-log-scan:    "DUS rejected request: invalid token format (expected base64)"
```

Subject: `endpoint:dus-gateway:/api/v2/loans/submit`

Agreements:
- Endpoint path: 3 sources agree on `/api/v2/loans/submit`
- Body format: code + network agree on JSON (docs say "XML or JSON" — not a conflict, just broader)
- Auth: code + network + config agree on mTLS + custom token header

Conflicts:
- Docs claim OAuth2 auth; code/network/config all show mTLS + token. This is a
  genuine conflict → available for decoding, or flagged in assessment as
  "docs appear outdated, operational evidence contradicts"

### Output (standalone — no catalog needed)

```
Assessment: dus-gateway-archaeology-2026-05-17

Subjects discovered: 12 endpoints, 3 auth mechanisms, 4 data formats

High-confidence model:
  - Base URL: https://dus.fanniemae.com/api/v2/
  - Auth: mTLS + X-DUS-Token (base64), NOT OAuth2 despite docs
  - Loan submission: POST /loans/submit (JSON)
  - Status check: GET /loans/{id}/status
  - ...

Conflicts requiring resolution:
  - Auth method: docs say OAuth2, all operational evidence says mTLS+token
  - Rate limiting: error logs suggest throttling, no other source confirms

Evidence gaps:
  - Webhook/callback mechanism: referenced in one error log, no other evidence
  - Batch submission: docs mention it, no code or network evidence

Detectors that ran:
  - code-archaeology:   completed, 34 claims (high quality — real integration code)
  - doc-parse:          completed, 22 claims (medium quality — docs may be stale)
  - network-observation: completed, 18 claims (high quality — observed traffic)
  - config-scan:        completed, 8 claims
  - error-log-scan:     completed, 11 claims
```

### What the Rust crate does here

- Spawns detector regions via asupersync (same as easy case)
- Subject-identity normalization is harder: needs merge proposals from skill layer
- Corroboration scoring across heterogeneous evidence quality
- Conflict detection: identifies where claims disagree on the same property
- Evidence neighborhood per subject: full provenance graph
- Assessment output with explicit confidence, conflicts, and gaps

### What the skill layer does here

- Decides which detectors to run (code archaeology needs a codebase path,
  doc-parse needs URLs/PDFs, network-observe needs log paths)
- Invokes sub-skills (code-archaeology is itself a complex skill with LLM)
- Proposes subject-identity merges (LLM-assisted: "these are the same endpoint")
- Interprets conflicts (LLM can say "docs are probably stale based on date")
- May run additional passes: "we have a gap on webhooks, let me grep for
  callback URLs in the codebase" → spawns a targeted follow-up detector

### What decoding does here (optional, downstream)

If the client wants a canonical API model, the conflicts get fed to decoding:
- "Auth method: docs say OAuth2, code/network/config say mTLS+token"
- Decoding policy: "operational evidence outweighs documentation when docs
  are undated and code is active" → resolves to mTLS+token
- Resolution becomes a canonical claim with full provenance trail

---

## Implementation sequence

### Phase 0: Contract freeze
- Finalize `claim.v0` wire format (already mostly done in PLAN_FACTORY)
- Define subject-identity normalization contract
- Define assessment output schema
- Define detector → crucible interface (how a detector emits claims)

### Phase 1: Rust crate core
- Asupersync integration: region management, detector lifecycle
- Claim ingestion and storage
- Subject-identity normalizer (deterministic merge rules)
- Corroboration scorer
- Conflict detector
- Assessment report generator
- Lab-runtime tests: deterministic detector simulation

### Phase 2: Easy use case — missing applications
- Detector scripts: process-scan, git-scan, dns-scan, cloud-config, browser, docker
- Crucible skill: orchestrates detectors for "scan this machine"
- Optional: cmdrvl-cli metadata diff adapter
- Ship as `crucible scan local` or similar

### Phase 3: Medium use case — DUS gateway archaeology
- Integration with code-archaeology skill
- Doc-parse detector (PDF/HTML → claims)
- Network/log observation detectors
- LLM-assisted subject-identity merge proposals
- Multi-pass investigation loops (find gap → spawn targeted detector)
- Ship as `crucible archaeology <target>` or similar

### Phase 4: Catalog integration
- Shell-out to `cmdrvl-cli metadata read` for diff
- Shell-out to `cmdrvl-cli metadata write` for hydration
- Drift detection mode: scheduled re-scan → compare → report changes

---

## Reality check

### What's strong

1. **Clean layering.** Skill / Rust crate / asupersync — each layer has one job
   and doesn't bleed into the others. Easy to reason about, easy to test.
2. **LLM boundary.** Preserving the spine-tool contract is good discipline and
   avoids a class of reproducibility and dependency problems.
3. **Forensics properties.** Evidence integrity, chain of custody, non-destructive
   observation — these give crucible professional-grade credibility.
4. **Two grounding use cases.** The laptop audit and DUS gateway prevent
   over-abstraction. Every design choice can be tested against "does this help
   me find missing apps?" or "does this help me reverse-engineer DUS?"
5. **Flywheel Connectors.** 150 connectors, same asupersync/capability model,
   no reinvention.
6. **System boundaries.** Elevating output above a flat subject list makes it
   operationally useful.

### What's risky or underspecified

1. **Asupersync maturity.** It's a library in active development, not a stable
   dependency. If it needs significant work before crucible can use it, that's a
   hidden Phase -1. Must assess readiness before committing Phase 1 timeline.

2. **Rust crate scope is large.** Subject-identity normalization, corroboration
   scoring, conflict detection, boundary clustering, evidence neighborhoods,
   assessment generation — that's a substantial system. Phase 1 alone could be
   months. Is there a proof-of-concept path that validates the idea before
   building the engine?

3. **Plugin model may be premature for Phase 2.** The laptop-audit detectors are
   probably shell scripts. The full plugin manifest / registry / doctor-check /
   FCP integration is a lot of machinery for six scripts that call `lsof` and
   parse git remotes.

4. **Flywheel Connectors coupling.** "Assumes a functioning installation" is a
   big assumption for standalone/external use. Local-only detectors must work
   without it. Need a clear fallback story.

5. **Boundary detection is a research problem.** Clustering heterogeneous
   signals into system boundaries is genuinely hard — closer to unsupervised
   learning than engineering. It might be the last thing that works well, not
   something to gate early phases on.

6. **No effort estimates.** Phase 0–4 are defined by deliverables, not size.
   Phase 0 could be days. Phase 1 could be months. Phase 2 could be weeks if
   you skip the plugin system or months if you build it first.

### Sequencing recommendation

The fastest path to proving crucible's value is to skip the Rust crate
initially and validate the concept with the skill layer alone:

```
Phase 0.5: Skill-only proof of concept (days, not months)
─────────────────────────────────────────────────────────
- Implement the laptop-audit as a Claude Code skill
- Six detector scripts (bash/python): process, git, dns, cloud-config, browser, docker
- Each emits claims as plain JSONL to a temp directory
- The skill reads all claims, does convergence (LLM-assisted), produces report
- Output: a structured assessment as markdown + JSONL claims
- No Rust crate, no asupersync, no plugin system, no FCP
- Validates: does multi-detector convergence actually find things the catalog missed?
```

If Phase 0.5 works, THEN build the engine:

```
Phase 0: Contract freeze (1–2 weeks)
────────────────────────────────────
- Freeze claim.v0 based on what the proof-of-concept actually needed
- Define subject-identity rules that were used in the PoC
- Define assessment schema based on actual PoC output
- These contracts are now grounded in reality, not speculation

Phase 1: Rust crate core (6–10 weeks)
─────────────────────────────────────
- Asupersync integration (requires maturity assessment first)
- Claim ingestion, subject normalizer, corroboration scorer
- Conflict detector, assessment generator
- Lab-runtime tests with simulated detectors
- At this point: same PoC but deterministic, fast, reproducible

Phase 2: Ship the easy use case (2–4 weeks)
───────────────────────────────────────────
- Port the six detector scripts to proper plugins (or keep as scripts
  with minimal manifest wrappers — don't over-engineer)
- Crucible skill wires up the Rust crate instead of doing convergence itself
- Optional: cmdrvl-cli metadata diff
- Ship as `crucible scan local`

Phase 3: Medium use case — DUS gateway (4–8 weeks)
─────────────────────────────────────────────────
- Code-archaeology skill integration
- Doc-parse and log-scan detectors (likely need FCP for remote access)
- LLM-assisted merge proposals
- Multi-pass investigation loops
- Boundary detection (first version — follow-edges, not unsupervised clustering)
- Ship as `crucible archaeology <target>`

Phase 4: Mature (ongoing)
────────────────────────
- Full plugin registry and discovery
- Catalog integration (read/write/drift)
- Boundary detection v2 (clustering)
- Profile marketplace
- Incremental/scheduled re-scans
```

### Critical path

The single most important thing to validate first: **does running multiple
detectors and converging their output actually discover things that a single
detector would miss?** If the answer is "yes, obviously, every time" — the
architecture is justified. If the answer is "not really, one good detector
finds everything" — the convergence engine is over-designed.

Phase 0.5 answers this question in days.

---

## Open questions

1. **Detector interface contract**: Do detectors emit claims directly into the
   asupersync channel, or do they produce a file/stream that crucible ingests
   after the region closes? Real-time vs batch affects the convergence engine.

2. ~~**Subject-identity as LLM-assisted**~~ **RESOLVED**: The crate uses
   deterministic merge rules only. When those are insufficient (medium+ cases),
   the skill layer proposes merges via LLM and passes them to the crate as
   explicit inputs. The crate never calls an LLM. See "LLM boundary" section.

3. **Detector discovery**: For the easy case, we know what detectors to run.
   For a general "scan this environment," how does the skill decide? Is there
   a meta-detector that looks at what surfaces are available and then picks?

4. **Incremental scans**: First scan produces an assessment. Second scan a week
   later — does crucible diff against its own prior assessment, or only against
   the catalog? Both seem valuable.

5. **Asupersync maturity**: Is the library ready to be a dependency, or does
   crucible need to abstract over it so we can swap in a simpler executor for
   Phase 2 (easy case) and bring asupersync in for Phase 3?
