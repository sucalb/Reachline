# Reachline

**A reachability-verified VEX triage dashboard for software supply-chain security.**

Most SBOM/SCA tools flood teams with vulnerability alerts that are technically
true but practically irrelevant — a CVE flagged in a dependency that your code
never actually calls. Reachline cuts that noise by tracing whether a
vulnerable function is *reachable* from your application's real entrypoints,
then uses an LLM to draft a [VEX](https://www.cisa.gov/resources-tools/resources/minimum-requirements-vulnerability-exploitability-exchange-vex)
(Vulnerability Exploitability eXchange) statement with the evidence attached
— so a human only has to review and sign off, not investigate from scratch.

## The problem

- Every scanner (Syft, Grype, Trivy, OSV-Scanner...) speaks a different
  output format — there's no single source of truth for "what's actually
  vulnerable in my stack."
- CVSS severity alone doesn't tell you if a vulnerability is *exploitable in
  your specific codebase*. Security teams spend most of their time manually
  triaging findings that turn out to be unreachable dead code or
  test-only dependencies.
- That triage work is repeated, by hand, across thousands of companies using
  the same open-source packages.

## The idea

1. **Normalize** — every scanner's output is converted to a canonical model
   keyed on [PURL](https://github.com/package-url/purl-spec), so findings
   from different tools can be deduplicated and correlated.
2. **Trace reachability** — build a static call graph of the target project
   and check whether any entrypoint can actually reach the vulnerable
   function, not just whether the package is present.
3. **Draft the VEX automatically** — feed the call path and the relevant
   source code to an LLM, which drafts a VEX statement (`affected` /
   `not_affected` / `fixed`) with a cited justification and a confidence
   score.
4. **Human review, not automation** — every draft goes into a review queue.
   A person approves or disputes it before it affects the reported risk.

## What's here

| | |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | SBOM ingestion, backend schema, REST API, scanner-plugin orchestration, async worker pipeline |
| [`ui/reachline-mockup.html`](ui/reachline-mockup.html) | Working UI prototype — findings ledger, reachability trace view, VEX review drawer, "File New Scan" flow |

The UI mockup is a self-contained HTML file (open it directly in a browser,
no build step) with mock data standing in for real scan results — it's the
interaction design, not a connected app yet.

## Status

Early-stage design. Architecture and UI are sketched out; backend
implementation (ingestion pipeline, reachability engine, LLM drafting) is
the next step. Not yet running against real projects.

## Why "Reachline"

The differentiator isn't another SBOM dashboard — it's tracing a *line* of
reachability from your code's real entrypoints to the vulnerable function,
so triage time goes to the vulnerabilities that can actually hurt you.
