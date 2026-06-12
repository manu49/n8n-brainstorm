# Customer Escalation Commander workflow setup

The importable workflow is [`customer-escalation-commander.json`](customer-escalation-commander.json). It implements **Workflow 1: Multi-Agent Customer Escalation Commander** from the repository's main interview specification.

## Compatibility

The export uses these n8n capabilities:

- Chat Trigger `1.3`
- Webhook `2.1`
- Data Table `1.1`
- Code `2`
- Basic LLM Chain `1.9`
- OpenAI Chat Model `1.2`
- HTTP Request `4.3`
- IF `2.2`

Use a current n8n release that includes the Data Table node and these Advanced AI nodes. Node versions may be migrated automatically by a newer n8n instance when the workflow is imported.

## Import

1. In n8n, select **Create Workflow**.
2. Open the workflow menu and select **Import from File**.
3. Import `workflows/customer-escalation-commander.json`.
4. Complete all configuration items below before activation.
5. Run the acceptance tests in the root README manually with controlled data.

The workflow is intentionally exported as inactive and contains no credential IDs or secrets.

## Required Data Tables

Data Table names must match these names exactly unless the corresponding workflow nodes are changed.

### `accounts`

| Column | Type | Required | Notes |
|---|---|---:|---|
| `account_id` | String | Yes | Stable unique account identifier |
| `company_name` | String | Yes | Exact lookup value used when no account ID is supplied |
| `plan` | String | Yes | For example, `enterprise` |
| `support_tier` | String | Yes | For example, `premium` |
| `renewal_date` | Date or String | No | ISO-8601 recommended |
| `annual_contract_value` | Number or String | No | Used as context, not for authorization |
| `account_owner` | String | No | Internal owner name or ID |
| `approved_contacts` | String | No | Prefer a JSON-encoded list if multiple values are needed |
| `risk_notes` | String | No | Internal context; do not place secrets here |

### `support_events`

| Column | Type | Required | Notes |
|---|---|---:|---|
| `event_id` | String | Yes | Stable support-event identifier |
| `account_id` | String | Yes | Joins to `accounts.account_id` |
| `created_at` | Date or String | Yes | ISO-8601 recommended |
| `event_type` | String | Yes | Ticket, incident, commitment, and so on |
| `summary` | String | Yes | Treated as untrusted data by the prompts |
| `status` | String | No | Open, resolved, pending, and so on |
| `source_url` | String | No | Internal evidence URL |

### `knowledge_base`

| Column | Type | Required | Notes |
|---|---|---:|---|
| `document_id` | String | Yes | Stable evidence identifier |
| `title` | String | Yes | Human-readable source title |
| `category` | String | Yes | Must align with expected issue categories |
| `excerpt` | String | Yes | Keep excerpts bounded and approved |
| `source_url` | String | Yes | Internal source URL |
| `effective_at` | Date or String | No | Effective date |
| `expires_at` | Date or String | No | Expiration date |
| `authority` | String | No | For example, `policy`, `product_docs`, or `sla` |

### `agent_cases`

| Column | Type | Required |
|---|---|---:|
| `case_id` | String | Yes |
| `source_event_id` | String | No |
| `account_id` | String | No |
| `status` | String | Yes |
| `severity` | String | Yes |
| `confidence` | String or Number | Yes |
| `evidence_json` | String | Yes |
| `internal_plan` | String | No |
| `customer_draft` | String | No |
| `approval_reason` | String | No |
| `created_at` | Date or String | Yes |
| `updated_at` | Date or String | Yes |

`case_id` should be unique. The workflow checks for an existing completed case and uses a Data Table upsert at the persistence boundary. If strict concurrency guarantees are required, enforce uniqueness in an external transactional store or put an idempotent intake service in front of the workflow.

## Credentials and variables

### OpenAI

Select an OpenAI credential on all five model nodes:

1. `Triage OpenAI model`
2. `Investigator OpenAI model`
3. `Writer OpenAI model`
4. `Reviewer OpenAI model`
5. `Revision OpenAI model`

The fifth call runs only when the reviewer requires changes. The normal path uses four model calls.

### Optional Tavily search

Create an n8n variable named `TAVILY_API_KEY`. The search node runs only when both the recent-incident result and internal knowledge retrieval are empty and triage supplied search terms.

If external search is prohibited, replace the search node with an approved provider or force `need_external_search` to `false` in `Assemble bounded evidence`.

## HTTP endpoints

Replace the placeholder `https://api.example.com` URLs in these nodes:

- `Get current incidents`
- `Get account telemetry`
- `Request human approval`

Expected contracts follow.

### Recent incidents

`GET /status/incidents?from=<ISO-8601 timestamp>`

Example response:

```json
{
  "incidents": [
    {
      "incident_id": "inc_123",
      "title": "Export processing delays",
      "status": "investigating",
      "started_at": "2026-06-12T08:00:00Z",
      "source_url": "https://status.example.com/incidents/inc_123"
    }
  ]
}
```

### Account telemetry

`GET /telemetry/account/<account_id>`

Example response:

```json
{
  "account_id": "acct_acme",
  "window_start": "2026-06-11T00:00:00Z",
  "export_attempts": 12,
  "export_failures": 9,
  "last_error_code": "EXPORT_TIMEOUT",
  "source_url": "https://internal.example.com/telemetry/acct_acme"
}
```

### Human approval

`POST /approvals`

The payload contains `approval_key`, `case_id`, severity, approval reason, internal plan, and customer draft. The approval service must treat `approval_key` as an idempotency key.

## Trigger payloads

### Chat

A user can enter a message such as:

```text
Acme says exports have failed since yesterday and renewal is next week. Help me respond.
```

The triage model extracts the likely account name. If the account name is missing or does not resolve to exactly one account, the workflow asks for an exact company name or account ID.

### Webhook

Send a `POST` request to the generated `customer-escalation` webhook path.

```json
{
  "event_id": "support_evt_1001",
  "account_id": "acct_acme",
  "company_name": "Acme",
  "message": "Exports have failed since yesterday and the customer needs an update.",
  "requester_email": "csm@example.com",
  "metadata": {
    "support_ticket_id": "TICKET-42"
  }
}
```

For webhook retries, keep `event_id` stable. The workflow derives a deterministic `case_id` from it. A caller may instead supply a stable `case_id` explicitly.

## Workflow stages

1. Normalize chat and webhook input into one case schema.
2. Reject invalid input.
3. Look up an existing completed or approval-pending case.
4. Triage the message into validated structured JSON.
5. Resolve exactly one account.
6. Retrieve bounded support history, incident data, telemetry, and approved knowledge.
7. Use public web search only as a bounded fallback.
8. Have an investigator create an evidence-grounded hypothesis.
9. Have a writer create an internal plan and customer draft.
10. Have a risk reviewer identify unsupported claims and required changes.
11. Conditionally revise the response.
12. Apply deterministic approval rules.
13. Upsert the auditable case record.
14. Submit high-risk cases for human approval or return a completed response.

## Deterministic approval rules

Human approval is required when any of these conditions is true:

- a structured AI stage fails validation;
- severity is `high` or `critical`;
- combined evidence confidence is below `0.70`;
- renewal is within 30 days;
- the reviewer does not mark the response safe to send.

These gates run in a Code node and are not delegated to an AI model.

## Security notes

- Retrieved customer, support, policy, API, and web text is explicitly labeled as untrusted data in model prompts.
- The external search query is bounded and strips email-shaped values before transmission.
- The workflow does not send the company name, account ID, requester identity, raw ticket history, or telemetry to web search.
- No model node can directly issue a refund, modify customer data, send a customer message, or approve a case.
- Model output used for routing or storage is parsed and validated by Code nodes.
- Configure authentication, allowlisted destinations, TLS, and timeouts on all production HTTP endpoints.
