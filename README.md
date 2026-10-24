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
- Following the 21 October 2026 LiteLLM supply-chain incident, never pull floating `latest` images. A verified release, immutable digest, dependency scanning, scoped IAM and a rotation plan are prerequisites.
- Sub-50 ms is a **p95 target for exact cache hits**, not an untested promise for semantic hits. Exact/semantic/miss latency acceptance must be separately measured.

## References

- [LiteLLM OSS/Enterprise capability matrix](https://docs.litellm.ai/docs/enterprise)
- [LiteLLM Presidio guardrail documentation](https://docs.litellm.ai/docs/proxy/guardrails/pii_masking_v2)
- [LiteLLM 21 October 2026 security advisory](https://docs.litellm.ai/blog/security-incident-oct-21-2026)

**Scope note:** This is an illustrative bid plan. All amounts, dates, hours, milestones and acceptance criteria must be agreed in writing before work begins.
