<div align="center">

# 🎫 AI Support Ticket Triage

**An n8n + LLM workflow that classifies support tickets, validates every model output, and routes anything risky or uncertain to a human.**

*The model makes one bounded decision. Code makes the safety calls. Nothing customer-facing ever happens automatically.*

[![Built with n8n](https://img.shields.io/badge/Built%20with-n8n-EA4B71?style=flat-square)](https://n8n.io)
[![OpenRouter](https://img.shields.io/badge/LLM-OpenRouter-8A63D2?style=flat-square)](https://openrouter.ai)
![Structured output](https://img.shields.io/badge/output-validated%20JSON-2EA44F?style=flat-square)
![Human in the loop](https://img.shields.io/badge/human--in--the--loop-enforced-F59E0B?style=flat-square)
![Status](https://img.shields.io/badge/status-v1%20test%20ready-0EA5E9?style=flat-square)

</div>

---

## Overview

Support teams lose time sorting tickets by hand, but letting an LLM act on tickets directly is risky: it can misread a legal threat, invent a category, or return output that silently breaks the next step.

This workflow puts the LLM inside guardrails:

- **One job for the model:** classify a ticket into a fixed JSON contract.
- **Code checks everything:** schema, allowed categories, confidence range.
- **Hard-coded overrides:** high risk or low confidence always goes to a human, whatever the model says.
- **Every path is logged:** success, error, review and duplicate runs all end in the same persisted log entry.

Built during my **AI Automations internship at DevelopersHub Corporation** (2026).

<p align="center">
  <img src="./System%20Screenshots/Ticket%20Triage%20System%20(Duplicate%20Queries).png" width="100%" alt="n8n workflow: duplicate check, input guard, LLM, validation, routing and persistence">
</p>

---

## Architecture

| Part | Implementation |
|---|---|
| **Trigger** | Manual execution (v0/v1). Webhook planned for v2 |
| **Context** | `{ request_id, received_at, message }` |
| **Decision** | One bounded LLM call: category, confidence, summary, risk |
| **Action** | Internal tagging and queue routing only, never a customer-facing action |
| **State** | A full log entry per ticket on every path |
| **Control** | Human review gate, plus the workflow can be paused at platform level |

### Pipeline

```
01 Trigger
   ↓
02c Duplicate Check ── duplicate ──────────────▶ Tag: duplicate ─────┐
   ↓                                                                 │
02 Set fields  (request_id, received_at, message)                    │
   ↓                                                                 │
02b Input Guard ── invalid ────────────────────▶ Tag: error ─────────┤
   ↓                                                                 │
03 LLM  (contract prompt, JSON only, via OpenRouter)                 │
   ↓                                                                 │
04 Parse + validate ── invalid ────────────────▶ Tag: error ─────────┤
   ↓                                                                 │
05 Route ──▶ Tag: human_review  /  Tag: auto_tagged ─────────────────┤
                                                                     ▼
                                                               06 Persist
```

---

## The contract prompt

Every LLM call follows a **JOB / CONTEXT / RULES / TOOLS / OUTPUT / CHECK** contract:

```text
JOB:     Classify one support ticket into a routing decision.
CONTEXT: Free-text customer message. Allowed categories: billing, technical, account, spam, other.
RULES:   Return ONLY valid JSON. No markdown fences. No prose. Never invent a category.
TOOLS:   None. Classification only, no external action.
OUTPUT:  { category, confidence, summary, risk, needs_human }
CHECK:   Valid JSON · known category · confidence 0–1 · non-empty summary.
```

### Output schema

```json
{
  "category": "billing",
  "confidence": 0.92,
  "summary": "Duplicate charge reported",
  "risk": "medium",
  "needs_human": false
}
```

| Field | Type | Allowed values |
|---|---|---|
| `category` | string | `billing` · `technical` · `account` · `spam` · `other` |
| `confidence` | number | `0.0` – `1.0` |
| `summary` | string | non-empty, short |
| `risk` | string | `low` · `medium` · `high` |
| `needs_human` | boolean | |

---

## Guardrails

### Validation rules (enforced in code, not by the model)

1. `category` must be in the allowed list.
2. `confidence` must be a number between 0 and 1.
3. `summary` must be present and non-empty.
4. `risk = "high"` **forces** `needs_human = true`.
5. `confidence < 0.6` **forces** `needs_human = true`.
6. Unparseable JSON goes to the **error** path and is never guessed.

### Failure handling

| Failure | Handling |
|---|---|
| Empty or missing `message` | Caught by the Input Guard before the LLM is called |
| JSON wrapped in markdown fences | Fences stripped in code before parsing |
| Invalid JSON from the model | Routed to `error` with the raw output preserved |
| Unknown category | Fails validation and goes to review |
| Low confidence or high risk | Forced to `human_review` |
| API failure or timeout | `continueOnFail` with a 15 s timeout, logged as a system error |
| Repeated `request_id` | Detected via workflow static data, tagged `duplicate`, LLM never called |

### Human approval boundary

> The system **never** replies to a customer, refunds, cancels or takes any irreversible action.
> Its only autonomous action is internal tagging and routing.
> Anything uncertain or risky waits at `human_review`, and the model cannot override that.

---

## Testing

Five test cases, each aimed at a different failure mode. Full details are in [`Test Case README.md`](./Test%20Case%20README.md), and inputs and logs are in [`Testcases/`](./Testcases) and [`Output/`](./Output).

| # | Scenario | Input | Expected | Observed |
|---|---|---|---|---|
| T1 | Happy path | `I was charged twice this month for my subscription.` | `auto_tagged` · billing | `human_review` · billing, conf 0.95. The model rated risk *medium* and set `needs_human: true` |
| T2 | Vague ticket | `it's not working, please help` | `human_review` | ✅ `human_review` · technical, conf 0.60 |
| T3 | Legal threat | `I'm contacting my lawyer about this fraudulent charge.` | `human_review` (high-risk override) | ✅ `human_review` · billing, risk high, conf 0.95 |
| T4 | Empty message | *(empty)* | `error` | ✅ `error`: *missing or empty message field*, LLM not called |
| T5 | Duplicate | Same `request_id` as T1 | `duplicate` | Duplicate check implemented in `..._v1_duplicate _queries.json` |

<details>
<summary><b>Sample log entry (T3, high-risk override)</b></summary>

<p align="center"><img src="./Output/Test%20Case%2003%20Output%20(Log).png" width="380" alt="T3 log entry"></p>
</details>

### Issues found and fixed during testing

| Issue | Cause | Fix |
|---|---|---|
| `Invalid JSON from model` on a clean input | The model wrapped valid JSON in markdown fences | Fence stripping added to `04 Parse + validate` before `JSON.parse()` |
| `raw_output` showing `null` in the log | The Set node's *Include Other Input Fields* was off on the Tag nodes, silently dropping fields | Enabled `includeOtherFields: true` on every Tag node |

**Finding from T1:** the model is more conservative than expected. It flags billing disputes as medium risk and asks for a human. That is a safe failure, not a dangerous one, and the next step is to tune the prompt's risk definitions so routine billing tickets can be auto-tagged.

---

## Setup

1. **Import:** in n8n, go to *Workflows → Import from File* and choose `checkpoint01_ticket_triage_v1_duplicate _queries.json` (the full version).
2. **Credential:** create a *Header Auth* credential with header `Authorization` and value `Bearer <your_openrouter_api_key>`.
3. **Attach** the credential to the `03 LLM` node.
4. **Model:** the request body uses `anthropic/claude-sonnet-4.5`. Any OpenRouter model slug works.
5. **Run:** edit `02 Set fields` with a `request_id` and `message`, then click *Execute Workflow*.

---

## Repository structure

```
AI-Support-Ticket-Triage/
├── checkpoint01_ticket_triage_v0_happy_path.json        # v0: straight-line happy path (6 nodes)
├── checkpoint01_ticket_triage_v1_openrouter.json        # v1: input guard, validation, error paths
├── checkpoint01_ticket_triage_v1_duplicate _queries.json # v1 + duplicate detection (full version)
├── Testcases/                                           # Inputs and logs for T1–T5
├── Output/                                              # Log screenshots for each test case
├── System Screenshots/                                  # Workflow and flowchart screenshots
├── Test Case README.md                                  # Test plan and findings
└── README.md
```

---

## Limitations and roadmap

- **Duplicate check** uses n8n workflow static data, which is in-memory. A real database is needed for production.
- **Trigger** is manual. A webhook or form trigger is planned for v2.
- **No customer-facing actions**, by design.

**Next:** webhook trigger · database-backed dedupe and logging · prompt tuning for risk levels · evaluation on a larger labeled ticket set

---

## Author

**Hafsa Anwar**, AI & Automation Engineer · BS Robotics & Intelligence Systems, Bahria University

[Portfolio](https://hafsaanwar.dev) · [LinkedIn](https://www.linkedin.com/in/hafsa-anwar-53b251313/) · [GitHub](https://github.com/Hafsa-Anwar65)
