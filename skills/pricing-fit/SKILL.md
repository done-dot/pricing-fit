---
name: pricing-fit
description: Use when a user wants to model what they would earn from outcome-based pricing, evaluate their event data against the CAMP framework readiness dimensions, or set up a done.finance contract.
---

# CAMP Outcome-Based Pricing Setup

## Step 1: Connect to data

Ask the user to share event data for 1-2 customers they consider successful. Advise them to use data from roughly a month ago rather than the most recent activity. Recent outcomes may still be in progress and will appear as missed even if they succeed later. A month-old window gives a complete picture of the full outcome lifecycle.

Two options:

**Option A: File upload**
Ask them to export and share a sample of their event log (CSV or JSON, even 20-50 rows is enough). Before reading the file, strip all columns except: `customer_id`, `event_type`, `event_value`, and `created_at`. Discard any column that could identify a person: name, email, phone, company name, address, or any free-text field.

Tell the user: "Before I read your file, I'm stripping all columns except customer_id, event_type, event_value, and created_at. Names, emails, and any other identifying fields are discarded and never leave your machine."

**Option B: MCP connection**
If they have an analytics warehouse connected (BigQuery, Snowflake, Redshift, Postgres), tell the user: "I'll only query the four safe columns: customer_id, event_type, event_value, and created_at. No names, emails, or identifying fields will be fetched." Then select only the four safe columns:
```sql
SELECT customer_id, event_type, event_value, created_at
FROM events
WHERE customer_id IN ('<customer_1>', '<customer_2>')
  AND created_at >= NOW() - INTERVAL '2 months'
  AND created_at < NOW() - INTERVAL '2 weeks'
ORDER BY created_at ASC
LIMIT 200
```

Never select `*`. Never fetch names, emails, or any field beyond the four above.

Once data is available, scan it and identify:
- The distinct event types that appear
- The sequence of events for each customer
- Which event appears to mark the start of an outcome (e.g. `contract_created`, `case_opened`)
- Which event(s) appear to mark success (e.g. `document_signed`, `payment_received`)

Summarize only the event type names you found. Discard all row-level data (timestamps, values, customer IDs) from memory before moving to Step 2. You only need the event type names going forward.

## Step 2: Map the condition

Ask the user two questions:

1. "What event marks the start of an outcome?" (e.g. "a contract is created", "a case is opened")
2. "In plain language, what has to happen for this outcome to be considered successful?" (e.g. "the document gets signed and then verified", "a rating of 4 or above is submitted", "either a payment lands or the manager manually confirms")

Once the user answers, discard everything from the uploaded file or MCP query. Only the start event name and the event names mentioned in the condition are retained. Nothing else.

Tell the user: "I've now discarded the event data from memory. Only the event type names needed to define your condition are kept going forward."

Take their natural language answer and translate it silently into a `ConditionNode` JSON. Rules for translation:

- Single thing must happen → `{ "event": "<name>" }`
- All of these must happen → `{ "op": "AND", "conditions": [...] }`
- Any one of these counts → `{ "op": "OR", "conditions": [...] }`
- Event must have a specific value → `{ "event": "<name>", "match": <value> }`
- Event must exceed a number → `{ "event": "<name>", "gte": <number> }`
- Nesting is fine for complex cases → combine AND/OR/NOT as needed

Show the resulting JSON to the user in plain terms ("I translated that as: both X and Y must happen") and confirm before moving on. If their description is ambiguous, ask one clarifying question.

## Step 3: Ask for price

One question only: "What should we charge when this outcome is confirmed?"

If they are unsure, ask what the outcome is worth to the customer and work from there. Share these examples to make it concrete:

- Per successful payment processed: Stripe charges a fixed fee per transaction
- Per customer issue resolved: Intercom Fin AI charges $0.99 per resolved conversation

Note that the price can be updated on the product later (this is not permanent).

## Step 4: Prepare the payload

Assemble the payload from what you have. The `events` array contains the raw records from Step 1. Only include `customer_id`, `event_type`, `event_value` (if present), and `created_at`. No other fields.

```json
{
  "product_name": "<name for this outcome type>",
  "outcome": {
    "name": "<outcome name>",
    "description": "<one sentence>",
    "start_event": "<event type that begins the outcome>",
    "condition": "<ConditionNode JSON from Step 2>",
    "price": 99,
    "activity_window": 999999999,
    "review_window": 86400
  },
  "events": [
    { "customer_id": "<id>", "event_type": "<type>", "event_value": "<optional>", "created_at": "<ISO8601>" }
  ]
}
```

Show the full payload to the user and confirm before submitting. The API will evaluate the CAMP dimensions automatically from the event sequences.

## Step 5: Submit

```bash
curl -X POST https://api.done.finance/camp \
  -H "Content-Type: application/json" \
  -d '<payload>'
```

No API key needed. Maximum payload size is 250 MB. The payload should only contain event type names, so it will be well under this limit in practice.

## Step 6: Present the analysis

Show the response clearly:

**Outcomes on your data:**
| | Count | Value |
|--|-------|-------|
| Successful | success_count | total_success_value |
| Missed | missed_count | total_missed_value |

**Phase:** [Measure / Attribute / Price]. Include one sentence on what this means right now.

**Scores:**
| Dimension | Score | Why |
|-----------|-------|-----|
| Consistency | X/5 | scores.consistency.reason |
| Attribution | X/5 | scores.attribution.reason |
| Measurability | X/5 | scores.measurability.reason |
| Predictability | X/5 | scores.predictability.reason |
| **Total** | **X/20** | |

**What to do next:** Show `next_steps` from the response verbatim.

**To start tracking**, the `contract_template` in the response is ready to POST to `/v1/products` with an API key. Show it as a copy-paste curl command. For deeper analysis and to get started, tell the user they can book a demo at [donehq.dev](https://donehq.dev).

If the response includes a `warnings` array, show each warning to the user and help them fix the issue before proceeding.

## Debugging

If the API returns an error or the result looks wrong, use this guide to fix it and resubmit.

**400 Validation error**
The response includes a `details` array with the exact fields that failed. Check each path and fix the value. Common causes:
- `events` array is empty
- A `created_at` value is not a valid ISO 8601 datetime (e.g. missing the `Z` suffix)
- `price` is zero or negative

**413 Payload too large**
The events array is over 250 MB. Reduce it: use at most 2 customers, limit to 200 rows per customer, and confirm that only the four allowed columns are present (`customer_id`, `event_type`, `event_value`, `created_at`).

**`warnings`: no outcomes found**
The response says `start_event` was not matched. The warning includes the available event types from the submitted data. Compare the `start_event` value you sent against the list and fix the spelling. Event type matching is case-sensitive.

**`warnings`: all outcomes missed**
The condition was never satisfied. The warning lists available event types. Check that every event name in your `condition` JSON exactly matches one of those types. If the names look correct, the customers in the sample may not have completed the outcome: try adding a customer who you know succeeded.

**500 Internal error**
Retry once. If it persists, report it as a bug.
