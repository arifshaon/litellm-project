# Delivery proposal and fixed-price milestones

**Proposed by:** Arif Shaon (`@arifshaon`)  
**Prepared:** 24 October 2026  
**Status:** Preliminary, public, pre-NDA bid. This is a proposed engagement, **not a claim that the gateway has been built**. The customer's full brief and confidential data are not included.

## Executive proposal

Build a **pilot-ready, privacy-first, multi-tenant AI cost optimisation gateway**, using LiteLLM as the **single AI gateway** in front of OpenAI, Anthropic and Google. Add only the custom features essential to prove savings transparently: UK PII detection/restoration, strongly partitioned masked caching, batch processing, scoped budgets, tenant-safe reporting and AWS London two-zone operations.

The engineering priority is not merely routing AI calls; it is to **demonstrate privacy controls and defensible finance figures** with automated evidence. Every milestone ends in a demo plus passed acceptance tests, with code created in the customer's GitHub repository.

Full architecture, limitations and acceptance matrix: [Implementation plan](implementation-plan.md).

## Comparable work, role and skills

**Personally developed middleware/proxy example:** NCAP Secure Image Proxy, a FastAPI Python HTTP proxy used to serve controlled image paths from Azure-mounted storage. Its implementation contains URL-to-storage-path routing, traversal protections, MIME/content-disposition handling, GET and HEAD endpoints, and configuration for cloud deployment. Its source is in a **private** GitHub repository (`arifshaon/ncap-proxy`) and can be demonstrated under an NDA, subject to any applicable rights.

The relevant transfer skills are: writing Python HTTP services, safeguarding an externally accessible route, containerised/cloud system design, backend integrations, data processing, operational debugging, and designing verifiable tests.

**Experience boundary:** Previous production **LiteLLM + Presidio + AWS ECS/Fargate** work is **not claimed** in this bid. Production **NestJS/Next.js** delivery is not claimed either. The TypeScript layer is deliberately kept as a thin control plane and dashboard, and all privacy-critical logic stays in Python, where the proposer has 15+ years of experience. The paid UK-detector trial and first LiteLLM integration are explicit technical qualification checkpoints. The plan depends on demonstrated results from those gates, not an unsupported claim of extensive experience with all three products.

## Proposed commercial offer

| Milestone | Window (2026) | Fixed price (USD) | Payment condition |
| --- | --- | ---: | --- |
| **M1 Foundation** | 26 Oct–6 Nov | $2,000 | Keys, providers, budgets, Docker, cloud monitoring and failure-mode design demonstrated |
| **M2 Privacy and reliability** | 9–20 Nov | $3,000 | UK recognisers, labelled >=50-prompt test corpus, mandatory masking, tenant isolation, provider fallback checks |
| **M3 Caching and ledger** | 23 Nov–4 Dec | $3,000 | Masked exact/semantic cache pilot, versioned price schedule, shadow baseline and tested savings reconciliation |
| **M4 Pilot differentiators** | 7–18 Dec | $3,500 | OpenAI-first Batch Autopilot, Cache Doctor report, Runaway Agent Guard, dashboards/statements |
| **M5 Pilot readiness** | 21–31 Dec | $2,500 | AWS London two-AZ deployment, monitoring, rollback, onboarding guide, documentation, handover and final demo |
| **TOTAL** | **26 Oct–31 Dec 2026** | **$14,000** | Five client-approved Freelancer Milestone Payments |

**Availability proposed:** 35–40 focused engineering hours/week, with a proposed start of **Monday, 26 October 2026**, and pilot handover **Thursday, 31 December 2026**. These are **proposed bid commitments and require confirmation of actual capacity before contracting**.

The fixed price excludes cloud consumption, LLM provider charges, any paid tooling/licence fees, Freelancer platform fees (if separately applicable), and third-party compliance certification. Payments are requested only after milestone acceptance tests. Scope changes are quoted/approved before implementation.

**Delivery feasibility:** The window provides roughly nine and a half weeks. That is tight for the original complete specification. The December pilot assumes **OpenAI-first batch integration**, a **report-based Cache Doctor**, agreed basic dashboards, and **fail-closed on complete gateway outage** unless a separate client-side PII-protected fallback is agreed. This is a controlled pilot scope, **not an implicit promise that all possible Batch APIs and client SDKs are complete**.

## Milestone details / definition of done

### M1 — Foundation (26 Oct–6 Nov; $2,000)

**Deliver:** client-owned repo structure, CI skeleton and tests, local Docker Compose, reproducible LiteLLM config, OpenAI/Anthropic/Google test connectivity, scoped virtual keys and per-client budgets, metadata-only basic CloudWatch health metrics. Record service boundaries and approve fail-open/fail-closed semantics.

**Accept when:** tenant A cannot use tenant B's keys; configured calls to all three providers pass; hard budget triggers can be demonstrated; recovery flow and threat model are reviewed; audit logging plan contains no raw prompts. A minimal benchmark methodology is agreed.

### M2 — Privacy and reliability (9–20 Nov; $3,000)

**Deliver:** tested Presidio-based synchronous privacy hook plus UK NI/NHS/check-digit/postcode recognisers, request-bound placeholder mapping and return restoration, metadata-only errors/logs, negative tenant tests, retry/back-up model routing, streaming handling or documented pilot exclusion.

**Accept when:** >=50 representative labelled prompts meet **>=95% instance-level removal/recall** on upstream-bound data under an agreed counting method; graders confirm responses restore correctly to an agreed threshold; checksum-positive/negative fixtures pass; forced sanitiser errors block all provider egress; cross-tenant isolation tests pass. Do not imply that 95% test recall guarantees privacy in every real-world case.

### M3 — Caching and finance (23 Nov–4 Dec; $3,000)

**Deliver:** masked tenant-scoped exact cache; semantic cache proof-of-concept with tenant partition/hard filter and supported AWS vector strategy; immutable per-request rate versions; dry-run/shadow reference model; metadata ledger and monthly aggregation; provider-native prompt caching discount excluded from *gateway-attributable* savings.

**Accept when:** cache data remains masked, cross-tenant cache probes fail, price-fixture calculations reconcile including rate changes/retries/discounts/negative savings, cache metrics are visible, and defined added-latency benchmarks are reported. **<50 ms p95 exact hits** is a target; semantic cache p95 is established at spike, not falsely promised equal to exact.

### M4 — Pilot differentiators/reporting (7–18 Dec; $3,500)

**Deliver:** explicit asynchronous `can_wait` jobs, masking before JSONL upload and OpenAI Batch API integration; batch result polling and controlled restoration; initial Cache Doctor diagnostics/report; spend hard limits and spike notifications; engineer dashboard and downloadable monthly finance statement.

**Accept when:** provider-uploaded batch fixtures have no raw PII; jobs and results are isolated across tenants; cached/prompt-discount/batch savings are not double-counted; Runaway Agent Guard tests show hard caps and spike alerts; finance report reconciles against gold test cases. Anthropic/Google batch APIs require separate feasibility/acceptance agreement.

### M5 — Pilot deployment and handover (21–31 Dec; $2,500)

**Deliver:** deploy to AWS London `eu-west-2` across 2 availability zones with appropriately separated LiteLLM, NestJS/UI and worker tasks; managed database/Redis controls, restricted admin, Secrets Manager, baseline alerts, secure CI/CD, deployment rollback and restore runbooks, onboarding checklist and final handover demonstration.

**Accept when:** one-AZ failure/degraded-dependency drills complete, mandatory PII policy still blocks unsafe egress, known-clean pinned LiteLLM build and licence inventory are recorded, client can onboard a test organisation, all smoke tests pass, and documentation allows another engineer to operate the pilot.

## Early paid trial — UK detector pack

A small, individually priced **paid trial (credited against M2 if awarded, subject to agreement)** can de-risk the highest-value security requirement before the full contract:

1. Build and test NI, UK postcode and NHS Modulus 11 recognisers in an isolated Python package using Presidio.
2. Supply labelled positive, negative and adversarial test samples; include checksum failure cases, repeated numbers and common false positives.
3. Return an evaluation report with precision/recall, edge cases, runtime cost, installation instructions and short demonstration.

The trial should not include real customer data; use synthetic identifiers. Its separate price and acceptance window need agreeing before work begins.

## Issues requiring a signed decision in M1

- **Gateway-down fallbacks.** Does “fail-open” mean continuing when *optional dependencies* fail, or surviving complete LiteLLM outage? The latter requires an independent PII-protected route (e.g., locally sanitising client SDK with directly controlled provider credentials) and cannot be achieved solely through a downed gateway. No bypass with raw PII will be offered.
- **Streaming contract.** Is a nonstreaming pilot acceptable for requests containing placeholders if real-time restoration cannot meet correctness tests?
- **Semantic-cache performance.** Is the semantic-hit p95 to be set after testing with a local embedding option? Exact hit and miss performance should be measured independently.
- **Batch provider scope.** Is OpenAI-first batching sufficient for December, with Anthropic/Google as a post-pilot phase?
- **Holiday acceptance window.** M5 spans 25–26 December UK bank holidays. Agree in M1 who signs off the final demo and when, for example a pre-recorded drill on 23 December and a live sign-off in the week of 28 December.
- **Provider key model.** Are credentials shared by platform with tenant cost allocation, or customer-supplied per organisation? This affects deployment, secrets and data boundaries.

## Security and licensing commitment

No second inference gateway. No GPL/AGPL or Enterprise-only code without prior approval. Do not store prompt text in audit/ledger. Use restrictive AWS IAM and private administration. Confirm a **safe, pinned immutable** LiteLLM image with signed/provenance checks and incident review (including the **24 March 2026 `1.82.7`/`1.82.8`** and **21 October 2026 `1.107.0`** compromises and CVE-2026-42208); avoid floating image tags and rotate secrets on suspected compromise.

## Communication and ownership

- Weekly written status and blocker/dependency tracking.
- Each milestone: issue checklist, code review, tests, live demo, concise acceptance report.
- Company owns and controls the delivery repository from the first day; keys shared only through approved secret management, not GitHub commits.
- End-of-pilot handover includes operational runbooks, environment setup, API examples, tenant onboarding, baseline validation and limits/known issues.

**Next step:** NDA, 30-minute requirements/acceptance call, and agreement on a small paid UK detector trial or M1 kick-off. Price/schedule/weekly capacity and the deliberately narrowed pilot boundaries should be confirmed **before** taking a fixed-price commitment.

---

**Reference:** [Full technical plan](implementation-plan.md). **This is a proposed solution, not a pre-existing deployed gateway.**
