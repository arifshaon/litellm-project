# AI Cost-Optimisation Gateway — Pilot Proposal

**Status:** Pre-NDA technical and commercial proposal, not an implemented gateway.  
**Prepared:** 24 October 2026  
**Target:** UK company pilot by 31 December 2026, subject to agreement on the MVP acceptance boundaries.

This repository contains a **proposal and implementation plan**, not source code for an already-built LiteLLM solution. It is based solely on the publicly shared project brief; no confidential material or credentials should be committed here.

## Start here

- [Technical implementation plan](docs/implementation-plan.md): architecture, privacy controls, UK PII detection, tenancy, caching, financial reconciliation, batch processing, failover, testing and risks.
- [Detailed delivery and commercial proposal](docs/delivery-proposal.md): deliverables, milestone acceptance, schedule, fixed-price proposal, collaboration and assumptions.

## Design at a glance

```text
Client applications
     |
     v
 AWS ALB (TLS, inference route only)
     |
     v
 LiteLLM Gateway (sole AI gateway / proxy)
   |-- Python pre-call PII mask + per-request restore
   |-- Tenant isolation / budget / model policy
   |-- Masked exact cache + optional semantic cache
   |-- Provider routing and circuit breakers
     |
     +----> OpenAI / Anthropic / Google

 Next.js dashboard ---> NestJS control plane ---> PostgreSQL ledger
                                  |                  Redis cache
                                  +----> Internal batch worker
                                             |
                                             +----> Mask JSONL first; then provider batches
```

NestJS is a **control plane, not a second inference proxy**. A batch worker is a background job processor, not a gateway.

### Security invariant

**No request containing unmasked detected PII may be sent to any provider.** Normal gateway operation defaults to fail-closed if the masking step is unavailable. A true gateway outage cannot be overcome by an internal LiteLLM fallback: an optional *client-managed*, independently validated, local-redaction direct-provider mode would require explicit agreement and separate acceptance tests. Otherwise the safe outcome is fail-closed.

### Deliverables

M1 Foundation; M2 Privacy and reliability; M3 Caching and savings ledger; M4 Pilot differentiators and reports; M5 AWS London two-AZ readiness, operational controls and handover.

### Comparable, verifiable engineering work

The proposer authored a FastAPI HTTP file proxy with URL/path controls, Azure storage mounting, GET/HEAD endpoints and defensive path resolution. The actual repository is private and can be demonstrated after an NDA: **NCAP Secure Image Proxy** (`arifshaon/ncap-proxy`). This is relevant **proxy/middleware experience**, not a claim that an AI gateway or LiteLLM deployment already exists.

### Due diligence

- LiteLLM OSS contains virtual keys, budget/spend controls and a guardrail framework; several administrator/audit/reporting controls are Enterprise features. All required custom controls here must be implemented with permissively licensed components, with a licence inventory and approval gate.
- LiteLLM had two supply-chain compromises in 2026: PyPI `1.82.7`/`1.82.8` (24 March) and `1.107.0` on PyPI, Docker Hub and GHCR (21 October, when the `latest`, `main-latest` and `main-stable` tags were also hijacked). CVE-2026-42208 was also added to CISA's Known Exploited Vulnerabilities list. Never pull floating tags. Pin an explicitly reviewed release by digest that is newer than the post-March audit fixes (`1.83.0`) and is not `1.107.0` (vendor currently advises `1.106.2`). Dependency scanning, scoped IAM and a credential-rotation plan are prerequisites.
- Batch cost tracking on LiteLLM's unified `/batches` route is documented as Enterprise-only, so batch costs in the savings ledger are computed from our own versioned price table rather than taken from LiteLLM.
- LiteLLM's Presidio `during_call` mode runs in parallel with the provider call, and `logging_only` masks logs only. Neither can be used for upstream masking. Only a blocking pre-call hook meets the privacy invariant.
- Sub-50 ms is a **p95 target for exact cache hits**, not an untested promise for semantic hits. Exact/semantic/miss latency acceptance must be separately measured.

## References

- [LiteLLM OSS/Enterprise capability matrix](https://docs.litellm.ai/docs/enterprise)
- [LiteLLM Presidio guardrail documentation](https://docs.litellm.ai/docs/proxy/guardrails/pii_masking_v2)
- [LiteLLM Presidio PII masking tutorial](https://docs.litellm.ai/docs/tutorials/presidio_pii_masking)
- [LiteLLM batches documentation](https://docs.litellm.ai/docs/batches)
- [LiteLLM 21 October 2026 security advisory](https://docs.litellm.ai/blog/security-incident-oct-21-2026)
- [LiteLLM security blog, including the March 2026 incident](https://docs.litellm.ai/blog/tags/security)

**Scope note:** This is an illustrative bid plan. All amounts, dates, hours, milestones and acceptance criteria must be agreed in writing before work begins.
