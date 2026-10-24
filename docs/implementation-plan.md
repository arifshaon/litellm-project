# Implementation plan — LiteLLM AI Cost-Optimisation Gateway

**Version:** pre-NDA proposal, 24 October 2026  
**Purpose:** pilot-ready multi-tenant cost optimisation for UK organisations, using **LiteLLM as the only inference gateway**.  
**Not an existing product:** this document describes intended work; no implementation or benchmark result is claimed.

## 1. Design principles and boundaries

1. LiteLLM remains the sole externally exposed inference API. **No second gateway/proxy**. A public ALB routes requests to LiteLLM tasks; Next.js/NestJS serve administration/reporting rather than transparently proxying model calls.
2. Preserve a stringent privacy invariant: all allowed inference inputs are classified/masked **before** any upstream provider request; failures in mandatory masking **block** the request.
3. Derive tenant identity from **validated virtual keys / server-side authentication**, never a caller-supplied tenant header. Every query, cache entry, budget, job, audit event and ledger row must be tenant-scoped.
4. Keep audit and cost telemetry **metadata-only**. Never persist prompt/response bodies in logs or analytics. PII reconstruction mappings are ephemeral, with documented rules for async jobs.
5. Use permissively licensed components only. Check **licences per package, image and subdirectory**; obtain approval before GPL/AGPL or commercial-only code.
6. Treat financial savings as reproducible calculations with immutable rate-version references, not as marketing estimates.
7. Use the agreed stack: Python hooks + LiteLLM, NestJS/TypeScript, Next.js, PostgreSQL, Redis, Docker, ECS/Fargate in AWS eu-west-2 across two availability zones.

## 2. Logical architecture

```text
                        UK CLIENT APPLICATIONS
                                 |
                          TLS / public ALB
                                 |
                      [ LiteLLM, only AI gateway ]
                       |     |      |         |
                       |     |      |         +--> provider routing/retries
                       |     |      +------------> pre-call Python privacy policy
                       |     +-------------------> cost and spend hooks
                       +-------------------------> tenant-scoped masked cache
                                 |
                        OpenAI / Anthropic / Google

 Next.js engineer/finance UI  ---> NestJS control plane
                                        |
                           tenant authorization / APIs
                                        |
               +------------------------+----------------------+
               |                        |                      |
           PostgreSQL             Redis / vector          Batch worker
      tenants, ledger, audit      masked caches       privacy-first JSONL
      jobs, rate versions           rate counters        upload/poll/restore
```

- **Inference data path:** LiteLLM processes authentication/authorisation; synchronously executes a mandatory Python PII masking hook; optionally hits a tenant-isolated cache; routes to an approved provider; rehydrates eligible placeholders in the response only for the originating client.
- **Management path:** Next.js and NestJS implement onboarding, tenant-scoped policy/ledger APIs, batch queue management, dashboard and monthly statements. NestJS **does not** forward ordinary inference requests.
- **Batch path:** a background processor accepts eligible asynchronous work; it calls the **same Python sanitisation library** as the LiteLLM hook, builds masked JSONL, submits provider batch jobs, polls for results, and restores only to the originating tenant. Direct file/batch endpoints must not allow a user to bypass this policy.
- **Persistence:** PostgreSQL for accountable records and durable batch job state; Redis for exact cache, semantic index when supported, per-tenant counters and bounded transient state. Do not store plaintext prompts in either datastore.

## 3. Core workstreams

### 3.1 Foundation and isolation

- Company-owned GitHub repository from day one; documented folders for LiteLLM config/hooks, `services/control-plane`, `apps/dashboard`, infrastructure and tests.
- Docker Compose for local development and CI checks. Provider connection tests using test-only keys supplied securely.
- LiteLLM virtual keys, team/org allocation where supported in OSS, per-tenant limits and budgets; server-side mapping from authenticated key to immutable tenant ID.
- Database design with composite tenant-scoped keys, PostgreSQL row-level policies where applicable and explicit tenant filters in application code. Separate read/write credentials and least-privilege access.
- Avoid trusting caller-controlled `team_id`, cache key, metadata tags or callback parameters.
- Provider routing, timeouts, limited retries for safe/idempotent cases, failover between approved models with explicit semantic/cost limitations and circuit-breaker behaviour.
- Initial health checks and CloudWatch dashboards. Model and rate configuration must be versioned.

### 3.2 UK PII detection, anonymisation and restoration

**Approach:** Microsoft Presidio's analyzer/anonymizer libraries plus custom UK recognisers wrapped in a **synchronous, in-process Python policy hook**. Built-in LiteLLM Presidio uses HTTP services; in-process reuse is a deliberate custom engineering choice to benchmark. If the custom hook proves unsafe or incompatible, evaluate the supported `pre_call` Presidio service deployment and document any latency impact.

- Detectors: NI numbers with formatting/validity guards; NHS numbers (10 digits; NHS Modulus 11 check digit, reject invalid checksum); UK postcodes (canonical grammar plus normalisation); core entities (email, phone, name/address) from Presidio recognisers. Assess false positives and ambiguous identifiers.
- Per request, replace entities with typed indexed placeholders, e.g. `<PERSON_1>`, `<NHS_1>`. Bind placeholder maps to one authenticated request/response context, **never to a global cache**.
- Sanitise every supported text-bearing field including system/user/assistant messages, tool-call content, tool responses and relevant multimodal metadata. Unsupported binary/file paths are **blocked or explicitly excluded** for pilot.
- Cache stores only **masked content**. A hit may be restored with the **current** request's mapping only after validating type/count/schema; cache entries with incompatible or unexpected placeholders are invalidated, not blindly reused.
- Response restoration respects structured JSON and tool outputs. For streaming, use a bounded placeholder-aware buffer/reassembler, or initially disable streaming for requests requiring restoration until streaming tests pass. Do not promise arbitrary untested streaming fidelity.
- Never send raw prompts to observability, provider error logging, exceptions, tracing, analytics or fallback provider SDKs.
- Because detection is imperfect, PII acceptance measures **recall on a labelled corpus**, including negative/false-positive cases. Manual review checks that restored answers preserve intended meaning.

**Privacy test suite:** at least **50 representative prompts** (suggest 60–100 for coverage), seeded with both syntactically valid and invalid UK identifiers; include multi-entity overlaps, entities repeated in different order, JSON/tool messages, streamed placeholder boundaries, error outputs, duplicates, tricky postcode variants and deliberate bypass attempts. Acceptance: **>=95% recall of labelled PII instances not forwarded upstream** and no **known** raw-PII egress for supported test scenarios, plus agreed correctness grading after restoration. Precision and failure rates should be reported too; this is not a claim of perfect real-world detection.

### 3.3 Caching and latency

- **Exact cache** lookup on a canonical, already-masked representation with cache namespace `tenant:policy-version:model-and-parameters:...`. Include relevant message roles, tool definitions, model, temperature, output format and policy version in the key; exclude plaintext PII.
- **Semantic cache** optional per tenant. Validate Redis engine/version and vector-search capability on target AWS service; if unsupported, implement a scoped pgvector strategy only after client approval. Prefer a local embedding model if tests show benefit; this is not assumed to fit p95 targets.
- Exclude unsafe non-deterministic/high-risk requests and those with unresolved personalised placeholders from caching. Masking itself runs **before any lookup**.
- Test exact-hit p50/p95/p99, semantic-hit p50/p95/p99 and miss p50/p95/p99 using representative request lengths and warm containers.
- Proposed targets (subject to baseline/benchmark agreement): **exact hit <50 ms p95 added latency**, **miss <200 ms p95 added latency** measured against an equivalent direct call. Semantic-hit p95 is to be measured and agreed after the spike, rather than represented as <50 ms by default.
- Added latency should be measured inside a controlled region/network and **subtract provider inference time**, with definitions for cache-hit end-to-end time and cold-start exclusions.

### 3.4 Cost baseline and trustworthy savings ledger

- **Price catalogue:** versioned table with provider, model, input/output/cache read/write token tiers, batch pricing, effective-from time, currency, source link, and catalogue revision hash. Lock the price version on every ledger event.
- **Telemetry:** hashed/request identifier, tenant ID, routed provider/model, token quantities, whether hit/routed/batched, cost attribution inputs, rate version, event timestamp and reconciliation status. No prompt text.
- **Shadow mode:** capture what the request *would have cost* at the user-configured baseline model/provider and compare against the policy-selected execution. For true cache hits, preserve defensible token usage estimates and mark their estimation method.
- **Attribution:** distinguish provider-native prompt caching rebates from savings due to gateway mechanisms. An illustrative ledger has:
  - `reference_cost`: estimated baseline cost under a documented reference policy and rate revision;
  - `provider_cache_rebate`: independently attributable native provider discount, shown separately;
  - `gateway_adjusted_baseline`: baseline adjusted for provider-native caching when applicable;
  - `gateway_cost`: actual measured/estimated amount due for the optimised run;
  - `gateway_attributable_savings = gateway_adjusted_baseline - gateway_cost`.
  Record negative savings; do **not** silently floor to zero or double-count provider discounts. In ambiguous counterfactual cases disclose estimated rather than verified savings.
- **Reconciliation:** fixture-based tests with known token counts, rate changes, model switches, retries, provider-native cache hits, exact/semantic cache hits, and partial/refunded/failed batch jobs. Compare ledger totals to provider billing exports where feasible.
- **Statements:** monthly company-level totals, breakdown by optimisation mechanism, methodology, confidence/estimated markers, rate versions, and reconciliation differences.

### 3.5 Pilot differentiators

**Batch Autopilot:** explicitly asynchronous `can_wait` contract with job ID, polling and optional webhook. For the December pilot, **OpenAI provider batch integration first**; support for Anthropic/Google batch transport can be priced/gated separately if needed. All JSONL lines must be masked **before** upload, regardless of whether LiteLLM's ordinary `pre_call` hook runs on batch/file endpoints. Retain a tenant-bound mapping safely for completion restoration: encrypt with KMS envelope keys, strict TTL, restrict read access, scrub on expiry and document retention. Log only status/IDs. Verify operational batch cancellation/retry/idempotency and completion delivery before demo.

**Cache Doctor:** first iteration is a report detecting prompt patterns correlated with poor caching (unstable timestamps, changing system prefixes, volatile formatting and ordering). Produce actionable observations and candidate normalisation rules; no silent rewriting of sensitive prompts.

**Runaway Agent Guard:** per-tenant/key/model hard budgets, rate caps, spending spike alerts, throttling and event de-duplication. Redis counters for speed; reconcile against durable events. Define behaviour on Redis failure with privacy and cost safety prioritised.

### 3.6 Failure modes and gateway-down policy

| Failure | Planned response |
| --- | --- |
| Provider error/timeout | Approved retries or backup model **only after mandatory masking**; track retries in ledger. |
| Exact/semantic cache down | Continue via provider if PII and spend controls still work; record cache degraded. |
| Spend/log telemetry temporarily down | Short bounded, non-PII event buffer; hard tenant budget guard must still operate or fail closed where limits cannot be enforced. |
| Presidio/hook unavailable or uncertain | **Fail closed**, no raw prompt forwarded. |
| One AZ/task fails | Multi-task LiteLLM rollout across both AZs, ALB health-based replacement; test failover. |
| Complete gateway/service outage | **Fail closed by default.** A true fail-open mode cannot be provided *from inside a downed gateway*. Optional independent **client-side** redactor plus direct provider integration would need separate SDK/integration, tenant-controlled provider secrets and explicit PII/budget acceptance tests. No unmasked direct bypass. |

**Important acceptance decision for M1:** The client must agree whether its per-client “fail-open” requirement means (a) **degrade non-privacy gateway dependencies** while LiteLLM remains available (included), or (b) **continue during complete gateway outage** (requires a separately deployed/client-side protection path with additional work). The pilot will not claim case (b) without an approved implementation and tests.

## 4. AWS London architecture and security

- AWS region `eu-west-2` (London), with ALB and ECS/Fargate tasks across **two availability zones**; TLS termination, managed certificate, minimum two LiteLLM tasks and separately scaled control plane/worker.
- PostgreSQL on managed RDS with Multi-AZ and backups, encrypted at rest/in transit; Redis managed deployment with network isolation and tested failover. Verify service/engine supports required semantic vector operations **before** choosing it.
- Private subnets/security groups as appropriate; no public database/Redis. Administrative LiteLLM UI and master key endpoints restricted to a private/admin access path; no public admin UI through the inference ALB.
- AWS Secrets Manager for provider keys and data-plane credentials; scoped task IAM; CloudTrail; CloudWatch alarms; deployment with reproducible IaC and least-privilege policies. Confirm OSS integration details for each secret access pattern; custom secret loading if required.
- Supply chain: **never `latest`** for LiteLLM; confirm safe release and pinned immutable digest; lock dependencies, build SBOM, scan vulnerabilities and licences, verify signatures/provenance where supplied, rotate keys if exposure suspected.
- The 21 October 2026 LiteLLM `1.107.0` supply-chain compromise makes a mandatory release-review step important. At this proposal date the vendor advises `1.106.2` or an earlier verified version; **revalidate before deployment**, since no version is declared permanently safe here.
- Production alerts for task health, auth failures, repeated PII blocks, cache health, high error rates, spend spikes, reconciliation lag and queue age. No raw prompts in alert payloads.

## 5. Verification matrix / acceptance

| Check | Evidence |
| --- | --- |
| OpenAI, Anthropic, Google connections | Synthetic provider requests and errors through **one** LiteLLM gateway |
| Tenant isolation | Negative cross-tenant tests for keys, budgets, caches, semantic search, audit, batch results and ledger |
| Privacy | >=50 labelled PII prompts; NHS checksum cases; >=95% labelled PII recall on upstream-bound text; graded restoration correctness |
| Streaming | Chunk-split placeholder tests, tool-call/structured output tests or documented pilot restriction |
| Cache | Deterministic cache key tests; masked-only storage inspection; cache latency p95 measured |
| Cost | Golden invoices/fixtures; price-revision tests; separate provider caching discount; negative savings accounted |
| Batch | No raw PII in uploaded JSONL; idempotency, async polling/webhooks, failure recovery and result mapping |
| Fallback | Masking-error denial, gateway outage fail-closed, provider retries/failover with masked inputs, degraded dependency scenarios |
| Deployment | Two AZ task survival, restore drill, load test, alerts, rollback, operational runbook and onboarding walkthrough |
| Licence/security | Documented SBOM, permissive dependency licences, checked LiteLLM image digest, admin endpoint isolation |

Detailed measurable baselines, exact p95 traffic distributions, payload sizes and response correctness scoring will be jointly signed off in the M1 test specification.

## 6. Risks, scope gates, and exclusions

**High priority risks and controls**
1. **Upstream LiteLLM security:** supply-chain advisory review, immutable artifacts, task isolation, credential rotation drill.
2. **PII quality:** UK recogniser false positives/negatives; labelled fixture evaluation and manual review; mandatory blocking on sanitiser failures.
3. **Semantic cache latency:** quick prototype and benchmark gate; no hard <50 ms promise for semantic hits.
4. **Batch privacy:** custom masking path; prevent direct provider file uploads; provider-by-provider certification.
5. **Accounting credibility:** detailed counterfactual assumptions and reconciliation fixtures; no fabricated or double-counted savings.
6. **Fallback ambiguity:** define *gateway down* versus *noncritical dependency down* contract in M1.

**Initial exclusions unless separately approved:** bespoke client-side fallback SDK for complete outages; all-provider Batch Autopilot; unrestricted arbitrary multimodal/file ingestion; commercial LiteLLM features; full enterprise SSO; regulatory compliance certification; unlimited high-throughput and global/multi-region scale. These exclusions do not diminish the core security acceptance tests.

## 7. Collaboration and handover

- Development in **client-owned GitHub** from the first code commit; merge requests, tests, setup notes and demo evidence attached to each milestone.
- Weekly written status: work completed, verified acceptance evidence, risks, decisions and next week's deliverables.
- Each milestone concludes with a live demo and an agreed acceptance checklist before release of its fixed-price payment.
- Handover: architecture diagram, IaC, reproducible Docker setup, API examples, credentials/rotation checklist, rate-table update procedure, incident/backup/restore runbooks, tenant onboarding and operations training.

## Reference links

- [LiteLLM Enterprise / OSS feature matrix](https://docs.litellm.ai/docs/enterprise)
- [Presidio PII masking in LiteLLM](https://docs.litellm.ai/docs/proxy/guardrails/pii_masking_v2)
- [LiteLLM October 2026 incident advisory](https://docs.litellm.ai/blog/security-incident-oct-21-2026)
- [Microsoft Presidio](https://microsoft.github.io/presidio/)
