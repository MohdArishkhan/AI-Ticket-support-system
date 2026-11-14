# AI Support Ticket Triage

An n8n workflow that uses an LLM to classify support tickets, validates the
model response, and safely routes each ticket to the correct internal queue.

The workflow is designed around one principle:

> The model makes a bounded classification decision. Code enforces the safety
> rules, and uncertain or risky tickets are sent for human review.

## Table of Contents

- [Overview](#overview)
- [Workflow](#workflow)
- [Classification Contract](#classification-contract)
- [Validation and Safety Rules](#validation-and-safety-rules)
- [Routes](#routes)
- [Setup](#setup)
- [Testing](#testing)
- [Project Structure](#project-structure)
- [Limitations and Roadmap](#limitations-and-roadmap)
- [Author](#author)

## Overview

Support teams often spend significant time sorting tickets manually. Sending
tickets directly to an LLM without safeguards can also create problems, such
as invalid categories, malformed output, or unsafe automatic decisions.

This workflow addresses those problems by:

- Sending one classification request to an OpenRouter LLM.
- Requiring a fixed JSON response format.
- Validating the response in n8n Code nodes.
- Detecting duplicate `request_id` values.
- Rejecting empty or missing ticket messages before calling the LLM.
- Sending high-risk and low-confidence results to human review.
- Persisting a structured log for every completed path.

The workflow only performs internal tagging and routing. It does not send
customer replies, issue refunds, cancel services, or perform other irreversible
customer-facing actions.

## Workflow

```text
01 Trigger
    ↓
02 Set fields
    ↓
02c Duplicate Check
    ├── Duplicate → Tag: duplicate → 06 Persist
    └── New ticket
          ↓
    02b Input Guard
          ├── Invalid message → Tag: error → 06 Persist
          └── Valid message
                ↓
          03 LLM (OpenRouter)
                ↓
          04 Parse + validate
                ↓
          05 Route
                ├── Invalid/API failure → Tag: error
                ├── Risky/uncertain → Tag: human_review
                └── Valid/safe → Tag: auto_tagged
                                      ↓
                                  06 Persist
```

### Workflow stages

| Stage | Node | Responsibility |
| --- | --- | --- |
| 1 | `01 Trigger` | Starts the workflow. The current version uses a manual trigger. |
| 2 | `02 Set fields` | Creates `request_id`, `received_at`, and `message`. |
| 3 | `02c Duplicate Check` | Checks whether the request ID has already been processed. |
| 4 | `02b Input Guard` | Rejects an empty or missing message. |
| 5 | `03 LLM` | Sends the ticket to OpenRouter for classification. |
| 6 | `04 Parse + validate` | Parses and validates the model response. |
| 7 | `05 Route` | Selects the error, human review, or automatic tagging path. |
| 8 | `06 Persist` | Creates the final log entry. |

## Classification Contract

The LLM receives a structured contract containing:

- **Job:** Classify one support ticket.
- **Context:** A free-text customer message.
- **Allowed categories:** `billing`, `technical`, `account`, `spam`, `other`.
- **Rules:** Return JSON only and never invent a category.
- **Tools:** None. The model only classifies the ticket.
- **Output:** A fixed set of fields.

### Expected output

```json
{
  "category": "billing",
  "confidence": 0.92,
  "summary": "Duplicate charge reported",
  "risk": "medium",
  "needs_human": false
}
```

| Field | Type | Allowed values or requirements |
| --- | --- | --- |
| `category` | string | `billing`, `technical`, `account`, `spam`, or `other` |
| `confidence` | number | Between `0.0` and `1.0` |
| `summary` | string | Must be non-empty |
| `risk` | string | `low`, `medium`, or `high` |
| `needs_human` | boolean | Indicates whether human review is required |

## Validation and Safety Rules

Validation is implemented in the n8n Code node named `04 Parse + validate`.
The code is embedded in the workflow JSON rather than stored in a separate
JavaScript file.

The workflow:

1. Checks for API errors and timeouts.
2. Reads the model response from `choices[0].message.content`.
3. Removes optional Markdown code fences around JSON.
4. Parses the response with `JSON.parse()`.
5. Checks that all required fields are present.
6. Checks the category against the allowed category list.
7. Checks that confidence is a number from `0` to `1`.
8. Checks that the summary is non-empty.
9. Checks the risk value against the allowed risk list.
10. Forces `needs_human` to `true` when:
    - `risk` is `high`, or
    - `confidence` is below `0.6`.

Invalid responses are preserved with an error message and routed to the
`error` path. The workflow never guesses a replacement value for invalid model
output.

## Routes

| Route | Condition | Result |
| --- | --- | --- |
| `duplicate_skipped` | The `request_id` was already processed | The LLM is not called. |
| `error` | Input, API, parsing, or validation failure | The error is logged. |
| `human_review` | High risk, low confidence, or model-requested review | A human reviews the ticket. |
| `auto_tagged` | Valid result that does not require review | The ticket is routed to its category queue. |

All routes converge at `06 Persist`, which records the request ID, timestamp,
model output, route, queue, and error details.

## Setup

### Requirements

- An n8n instance.
- An OpenRouter API key.
- Access to an OpenRouter model.

### Import the workflow

1. Open n8n.
2. Go to **Workflows → Import from File**.
3. Import the full workflow:
   `checkpoint01_ticket_triage_v1_duplicate _queries.json`.
4. Open the `03 LLM` node.
5. Create or select an n8n **Header Auth** credential:
   - Header name: `Authorization`
   - Header value: `Bearer <your_openrouter_api_key>`
6. Attach the credential to the `03 LLM` node.
7. Open `02 Set fields` and edit the ticket values.
8. Execute the workflow.

The default model is `anthropic/claude-sonnet-4.5`. You can replace it with
another model slug supported by OpenRouter.

## Testing

The project includes five test cases covering normal processing and failure
handling. See [Test Case README.md](./Test%20Case%20README.md) for the complete
test plan.

| Test | Scenario | Expected result |
| --- | --- | --- |
| T1 | Duplicate billing charge | `auto_tagged` with category `billing` |
| T2 | Vague ticket | `human_review` because confidence is low |
| T3 | Legal or fraud-related threat | `human_review` because risk is high |
| T4 | Empty message | `error`; the LLM is not called |
| T5 | Reused request ID | `duplicate_skipped`; the LLM is not called |

T5 must be run after T1 with the same `request_id`, because duplicate detection
uses workflow static data.

### Test data and evidence

- Inputs: [`Testcases/`](./Testcases/)
- Output screenshots: [`Output/`](./Output/)
- Workflow screenshots: [`System Screenshots/`](./System%20Screenshots/)

## Project Structure

```text
AI-Support-Ticket-Triage/
├── checkpoint01_ticket_triage_v0_happy_path.json
├── checkpoint01_ticket_triage_v1_openrouter.json
├── checkpoint01_ticket_triage_v1_duplicate _queries.json
├── Test Case README.md
├── Testcases/
│   ├── README.md
│   └── Test_Case_01.json ... Test_Case_05.json
├── Output/
└── System Screenshots/
```

### Workflow versions

- `checkpoint01_ticket_triage_v0_happy_path.json`: Basic happy-path workflow.
- `checkpoint01_ticket_triage_v1_openrouter.json`: Validation and error-path
  workflow.
- `checkpoint01_ticket_triage_v1_duplicate _queries.json`: Full version with
  duplicate detection. This is the recommended workflow to import.

## Limitations and Roadmap

- Duplicate detection currently uses n8n workflow static data and is not a
  database-backed solution.
- The workflow currently uses a manual trigger.
- Logging is represented by the `06 Persist` node and should be connected to a
  durable database or logging service for production use.
- No customer-facing actions are performed by design.

Planned improvements:

- Add a webhook or form trigger.
- Use a database for duplicate detection and persistent logging.
- Tune risk definitions for routine billing tickets.
- Evaluate the workflow against a larger labeled ticket dataset.

## Author

**Mohd Arish Khan**  
