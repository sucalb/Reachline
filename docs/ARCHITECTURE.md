# Architecture

## 1. SBOM & vulnerability aggregation

Every scanner (Syft, Grype, Trivy, OSV-Scanner, Snyk...) outputs a different
format. Rather than force everything into one industry format, each scanner
gets its own **adapter** that parses its native output into a shared
internal (canonical) model. The canonical model uses two existing standards
as its "interchange language":

- **CycloneDX JSON** for SBOM/component data
- **OpenVEX** for vulnerability status/justification

The join key across every tool is the **PURL** (`pkg:npm/lodash@4.17.15`) —
it's how findings from two different scanners about the same package get
correlated instead of duplicated.

### Core entities

- `Component` — name, version, PURL, ecosystem
- `Vulnerability` — CVE/GHSA id, severity, CVSS, source
- `Finding` — a `Component` × `Vulnerability` × scan, with per-scanner
  provenance kept even after dedup (two scanners can disagree on severity —
  don't silently overwrite, keep both, surface the conflict)
- `Scan` — run id, target, commit, timestamp, status

### Correlation

When two+ scanners report the same CVE on the same component, findings are
merged into one but the raw source records are retained for audit. Severity
conflicts are shown, not resolved silently.

## 2. Reachability-based auto-VEX (the differentiator)

```
SBOM + vulnerability findings
        │
        ▼
1. Vulnerable Function Extractor
   — pull the fix commit from OSV (most reliable signal) to know exactly
     which function/file changed; fall back to LLM-parsed CVE description
        │
        ▼
2. Static Call Graph Builder
   — per-language (JS/TS via ts-morph is the first target; Python/Java/Go
     later), built from declared entrypoints (HTTP handlers, CLI main, queue
     consumers)
        │
        ▼
3. Reachability Engine
   — BFS/DFS from entrypoints to the vulnerable function; returns
     reachable: true/false + the concrete call path as evidence
        │
        ├── not reachable ──► LLM drafts VEX: not_affected
        └── reachable     ──► LLM drafts VEX: affected, with risk context
        │
        ▼
4. VEX Document Generator → OpenVEX JSON, with the draft's reasoning and
   evidence kept as an audit trail separate from the final signed-off
   statement
        │
        ▼
5. Human Review Queue — approve / reject / edit. Nothing is auto-applied
   without a person signing off — static reachability has false negatives
   (dynamic dispatch, reflection, eval), and the LLM can be wrong.
```

Data model additions: `CallGraphSnapshot` (cached per commit — building it
is expensive), `ReachabilityResult`, `VexDraft` (status, justification,
evidence, confidence, human review state).

## 3. Scanner orchestration (running scans from the app)

Rather than requiring users to install and run scanner CLIs by hand, the app
runs each scanner as an ephemeral **Docker container** — no host installs
needed, and untrusted target code/images stay isolated:

```ts
interface ScannerPlugin {
  id: string;                              // "syft" | "grype" | "trivy" | "osv-scanner"
  supports: ("path" | "git" | "image")[];
  dockerImage: string;                     // e.g. "anchore/syft:latest"
  buildArgs(target: ResolvedTarget): string[];
  parse(rawOutput: Buffer): CanonicalRecord[];
}
```

Scan targets: a local path, a git URL (+ ref, cloned into a temp workspace),
or a container image reference. A scan job fans out — one queue job per
selected plugin, run in parallel — then fans back in through the
correlation engine, the reachability engine, and the LLM drafting step.

## 4. Backend & API

**Stack**: Node.js/TypeScript (Fastify) + PostgreSQL + Redis/BullMQ for the
async job queue — chosen mainly so the call-graph builder (which needs a JS
runtime anyway) can run in the same process as the API without a
cross-language hop.

### Schema (Postgres)

```
targets            (id, name, repo_url, default_branch)
scans              (id, target_id, commit_sha, status, started_at, finished_at)
components         (id, purl, name, version, ecosystem)
vulnerabilities    (id, cve_id, title, description, severity, cvss_score, source)
findings           (id, scan_id, component_id, vulnerability_id, severity, ...)
reachability_results (id, finding_id, reachable, confidence, path_json, ...)
vex_drafts         (id, finding_id, status, justification, evidence_file,
                     evidence_line, llm_model, confidence, ...)
vex_reviews        (id, vex_draft_id, decision, reviewer_id, reviewed_at, ...)
users              (id, email, name, role)   -- admin | reviewer | viewer
```

### REST API (`/api/v1`)

```
POST /scans                         { target: {type, value, ref?}, plugins: [...] }
GET  /scans/:id/status               per-plugin progress

POST /ingest/sbom                    { target_id, commit_sha, sbom }
POST /ingest/scan-results            { target_id, commit_sha, scanner, raw }

GET  /findings?target_id=&status=&severity=&reachable=&q=
GET  /findings/:id                   full detail: component, vuln, trace, VEX draft
GET  /findings/summary?target_id=    stat-tile aggregates

GET   /findings/:id/vex-draft
PATCH /findings/:id/vex-draft         edit justification before approval
POST  /findings/:id/vex-draft/approve { reviewer_id }
POST  /findings/:id/vex-draft/reject  { reviewer_id, reason }
GET   /findings/:id/vex-draft/export  OpenVEX JSON download
```

### Auth

- CI ingestion endpoints: per-target API key.
- Dashboard users: session/JWT with role `admin | reviewer | viewer` —
  approving/rejecting a VEX draft requires `reviewer` or above, since it
  directly affects the risk shown to the whole team.

## Open questions / next steps

- Which language gets call-graph support first beyond JS/TS?
- Where does the federated/shared VEX idea (see below) fit in, if at all?
- Real backend implementation — everything above is design only so far.

## Ideas parked for later

- **Federated VEX sharing** — let orgs publish anonymized "not affected"
  conclusions for a given CVE + package version so others don't re-triage
  the same finding from scratch (builds on OpenVEX + something like GUAC).
- **Business-impact-weighted risk score** — tie SBOM findings to a service
  catalog (which service, does it touch PII/payments, is it internet-facing)
  instead of scoring purely on CVSS/EPSS.
- **Git-blame-linked SBOM diffs** — for a newly flagged CVE in a long-present
  dependency, auto-surface which commit/PR introduced it.
