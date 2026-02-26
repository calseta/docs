# Calseta — Project Context

This file describes the Calseta product in full. It is the authoritative reference for generating documentation, marketing copy, and any AI-assisted content for this project.

---

## What Calseta Is

Calseta is an open-source, self-hostable data layer for security AI agents. It ingests security alerts from any source, normalizes them to a common schema, enriches them with threat intelligence and identity context, and delivers clean, context-rich payloads to AI agents — so agents spend their tokens on reasoning, not plumbing.

Calseta is **not** an AI SOC product. It does not build, host, or run AI agents. It is the data infrastructure layer that makes customer-built agents fast, accurate, and cost-efficient.

**Tagline:** The data layer for your security agents.

**License:** Apache 2.0 — fully open source, self-hostable.

---

## The Problem Calseta Solves

Security teams building AI agents for alert investigation consistently hit the same walls:

- **Context gap.** Agents lack access to organizational knowledge — detection rule documentation, runbooks, IR plans, SOPs, and workflow documentation. Without this, agents produce generic output that doesn't reflect the organization's environment.
- **Integration burden.** Investigating a single alert requires calling 5+ external APIs (SIEM, threat intel, identity provider, ticketing). Each integration is custom code that's expensive to build and fragile to maintain.
- **Token waste.** Raw API responses are verbose and unstructured. Agents stuffing them into context windows burn tokens and produce worse output. Pre-normalized, pre-enriched data reduces token consumption and improves reasoning quality.
- **No deterministic layer.** Tasks like IOC enrichment, normalization, and alert routing are deterministic — they should never consume LLM tokens. Today, agents often perform these tasks themselves because no purpose-built infrastructure handles them.

---

## The Five-Step Pipeline

Every alert that enters Calseta passes through five deterministic steps before an agent sees it:

### 1. Ingest
**Endpoint:** `POST /v1/ingest/{source_name}`

Alerts arrive via webhook from any source system. Each source integration validates the raw payload and hands it off to the normalization step. Returns `202 Accepted` immediately — all downstream processing is async.

Supported in v1: Microsoft Sentinel, Elastic Security, Splunk, Generic OCSF webhook (`POST /v1/alerts`).

### 2. Normalize
**Schema:** OCSF Security Finding (class_uid: 2001)

All alerts are normalized to the Open Cybersecurity Schema Framework regardless of source. Source-specific fields that don't map to OCSF go in `ocsf_data.unmapped`. The original payload is always preserved in `raw_payload`. Normalization happens synchronously at ingest time.

### 3. Enrich
**Mode:** Async, parallel, cached

Threat indicators extracted from the alert (IPs, domains, file hashes, URLs, accounts) are enriched by all configured providers in parallel. Results are cached with provider-specific TTLs. A slow or failing provider never blocks others. Enrichment can also be triggered on-demand via `POST /v1/enrichments` for any indicator value.

**v1 enrichment providers:**
- **VirusTotal** — IP reputation, domain reputation, file hash analysis
- **AbuseIPDB** — IP abuse confidence score and category
- **Okta** — Account details, group membership, recent activity, MFA status
- **Microsoft Entra** — Account details, sign-in risk, group membership, conditional access

### 4. Contextualize
**Endpoint:** `GET /v1/alerts/{uuid}/context`

Relevant organizational documents are attached to the alert based on targeting rules. Documents can be global (always included) or targeted to specific alert types, severities, source names, or detection rules. The context resolution engine evaluates all documents and returns those that match.

Context sources include: runbooks, IR plans, SOPs, detection rule documentation, workflow documentation, past alert history.

### 5. Dispatch
Once enrichment is complete, the enriched alert payload is dispatched to registered agents via webhook. The payload includes the normalized alert, all enrichment results, associated detection rule documentation, and matched context documents — everything an agent needs to investigate and respond, in a single structured object.

Agents can also pull alerts at any time via REST or MCP.

---

## Features

### Alert Ingestion
Plugin-based source system. Each integration is a Python class implementing `AlertSourceBase`. Ships with Sentinel, Elastic, and Splunk out of the box. Any source can be added by implementing `validate_payload()`, `normalize()`, and `extract_indicators()`. Community integrations are welcome.

### Enrichment Engine
Async, parallel enrichment for every extracted indicator. Cached per provider with configurable TTLs (1 hour for IPs, 6 hours for domains, 24 hours for hashes). On-demand enrichment API for Slack bots and ad-hoc queries. Add new providers with a single Python class implementing `EnrichmentProviderBase`.

### Detection Rule Library
Auto-created when alerts arrive. Each detection rule has structured metadata (MITRE tactics and techniques, data sources, false positive tags, severity) and a free-form markdown documentation field. Documentation covers: query, strategy, blind spots, false positives, and recommended responses. Surfaced to agents in every alert payload.

### Context Documents
Upload runbooks, IR plans, SOPs, and playbooks. Use targeting rules to attach documents to specific alert types, severities, detection rules, or source names. Agents receive the right documents for every alert automatically — no manual lookup required.

### Workflow Catalog
Document your SOC playbooks as structured markdown. Each workflow describes what should happen when a specific type of event occurs — written for both human analysts and AI agents to read. Workflows are tagged to specific alert types, severities, or detection rules using the same targeting rules system as context documents. When a matching alert arrives, Calseta surfaces the relevant workflows alongside runbooks and enrichment data in the agent's context payload. The agent reads the workflow documentation and decides what to do — the intelligence stays in the agent, not the platform.

### MCP Server
Native Model Context Protocol server running on port 8001. Any MCP-compatible agent or tool can query alerts, read detection rule documentation, browse context documents, and access workflow documentation — with zero custom client code. Framework-agnostic: works equally with LangChain, LangGraph, raw Claude API, CrewAI, or any MCP-compatible tool.

**MCP resources:** alerts, detection rules, workflows, context documents, metrics
**MCP tools:** post finding, update alert status, trigger enrichment

### Metrics API
Alert volume, false positive rates, MTTD, workflow execution stats, time saved estimates. Accessible via REST and MCP so agents can reason about SOC health as part of their investigation context.

### Auth-Ready Architecture
API key authentication in v1. BetterAuth-ready architecture for clean extension to username/password and SSO/OIDC without modifying route code.

---

## Architecture

Two long-running processes, shipped in the same repository, started with a single `docker compose up`:

```
FastAPI Server (port 8000)          MCP Server (port 8001)
  POST /v1/ingest/{source}            Resources: alerts, rules,
  REST API (/v1/...)                  workflows, context docs, metrics
  Enqueues async tasks
         │                                      │
         └──────────────┬───────────────────────┘
                        │
               PostgreSQL (port 5432)
               (primary store + task queue)
                        │
               Worker Process
                 Enrichment pipeline
                 Agent webhook dispatch
                 Alert trigger evaluation
```

### Technology Stack

| Layer | Technology |
|---|---|
| Language | Python 3.12+ |
| Web framework | FastAPI |
| Validation | Pydantic v2 |
| Database | PostgreSQL 15+ |
| ORM | SQLAlchemy 2.0 async |
| Migrations | Alembic |
| Task queue | procrastinate + PostgreSQL |
| Caching | In-memory with TTL (Redis-ready) |
| MCP server | Anthropic MCP Python SDK |
| HTTP client | httpx async |
| Auth | API keys (BetterAuth-ready) |
| Containerization | Docker + Docker Compose |

---

## The Enriched Alert Payload

This is what an agent receives on every new alert — a single structured object with everything needed to investigate and respond:

```json
{
  "event": "alert.enriched",

  "alert": {
    "uuid": "9f2a-b3c1-...",
    "title": "Impossible Travel Detected",
    "severity": "High",
    "source": "sentinel"
  },

  "indicators": [
    {
      "type": "ip",
      "value": "185.220.101.47",
      "virustotal": { "malicious": 14, "suspicious": 2 },
      "abuseipdb": { "score": 97, "categories": ["hacking"] }
    }
  ],

  "detection_rule": {
    "name": "Suspicious Auth v2",
    "mitre_tactics": ["TA0001", "TA0006"],
    "documentation": "## Overview\n> Detects impossible travel..."
  },

  "context_documents": [
    { "title": "Identity IR Runbook", "type": "runbook", "content": "..." }
  ],

  "workflows": [
    { "name": "Revoke User Session", "documentation": "..." }
  ]
}
```

---

## Targeting Rules

The same targeting rules system controls which context documents and workflows attach to which alerts. Rules are evaluated against each incoming alert using JSONB logic:

```json
{
  "match_any": [
    { "field": "source_name", "operator": "eq", "value": "sentinel" },
    { "field": "severity", "operator": "in", "values": ["High", "Critical"] }
  ],
  "match_all": [
    { "field": "severity_id", "operator": "gte", "value": 3 }
  ]
}
```

`match_any` = OR logic. `match_all` = AND logic. Both can coexist in the same rule.

Supported operators: `eq`, `neq`, `in`, `not_in`, `contains`, `gte`, `lte`.
Supported fields: `source_name`, `severity`, `severity_id`, `tags`, detection rule fields.

---

## Core Philosophy

**Deterministic operations stay deterministic.** Enrichment, normalization, and context resolution never consume LLM tokens. These run in the platform.

**Token optimization is first-class.** Every API response is designed to give agents exactly what they need with minimal noise. Not raw JSON dumps — structured, concise, well-labeled data.

**AI-readable documentation is a feature.** Every entity — detection rules, workflows, context documents, enrichment providers — has a documentation field designed to be read by agents, not just humans.

**Framework agnosticism.** The REST API and MCP server work equally well with LangChain, raw Claude API, CrewAI, n8n, a Slack slash command, or anything else. No agent framework is privileged.

**The intelligence stays in the agent.** Calseta provides context and deterministic infrastructure. The agent applies judgment. This separation is intentional and never violated.

**Open by default.** Designed to be understood, extended, and contributed to. Clear extension points with documented interfaces.

---

## Target Users

**Primary — Digital-native organizations (50–500 employees)**
SaaS, fintech, and high-growth tech companies where software is core to the business. Small or no dedicated security team. Technical staff responsible for responding to security alerts. Actively experimenting with agentic AI workflows. Titles: CTO, Cloud Engineer, IT Manager, Software Engineering Manager.

**Secondary — Security-forward organizations (500–2000 employees)**
Dedicated security engineering or small SOC team. Active interest in building internal AI tooling. Frustrated with black-box AI SOC vendors. Titles: CISO, Head of Security, Security Engineer, Detection Engineer.

**The builder persona:** Technical enough to clone a repo, run Docker Compose, and write a Python script that calls an API. Wants control over AI tooling and has the skills to exercise it.

---

## What Calseta Is Not

- Not an AI SOC product — we do not build, host, or run AI agents
- Not a SIEM — no raw log storage, no detection or query capabilities
- Not a SOAR — no visual playbook editor, no proprietary automation runtime
- Not a multi-tenant SaaS — single-tenant, self-hosted

---

## Getting Started

```bash
docker compose up
```

That's it. The platform starts three services: FastAPI server (port 8000), MCP server (port 8001), and PostgreSQL (port 5432). Full setup documentation at [docs.calseta.com](https://docs.calseta.com).

---

## Links

- Website: https://calseta.com
- Docs: https://docs.calseta.com
- GitHub: https://github.com/calseta/calseta
- License: Apache 2.0
