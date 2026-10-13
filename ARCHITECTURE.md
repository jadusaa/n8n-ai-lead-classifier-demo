# Architecture

A small n8n workflow that puts a real LLM call inside a reliable pipeline. It is a companion
artifact to a larger reliability demo and is deliberately narrow in scope.

## Flow

```
Webhook -> Normalize -> Validate -> [invalid -> 400]
                        |
                   Prepare Prompt
                        |
                   LLM (Hive, HTTP Request)
                        |
                   Parse + Validate Schema
                        |
                   [invalid -> 502]   [valid -> Route -> 200]
```

## Input contract

`POST /webhook/ai-classify` with a JSON body:

| Field | Required | Notes |
|---|---|---|
| `name` | yes | Trimmed; used by the validation check |
| `email` | no | If present, must contain a single `@` with text either side |
| `message` | yes | Trimmed and capped at 1000 characters |

## LLM call

One HTTP Request node calls the Hive API's OpenAI-compatible chat-completions endpoint. Doing it
as an HTTP Request rather than a provider node keeps the request shape visible and auditable on
the canvas: model, `max_tokens`, `temperature`, system prompt and user message are all on the
node. The key is read from an n8n **credential**, never written into the workflow JSON — the
exported workflow contains only a placeholder credential id.

- Endpoint: `POST https://api.thehive.ai/api/v3/chat/completions`
- Model: `zai-org/glm-5.3-flash`
- Auth: n8n header credential (`Authorization: Bearer <key>`)

The prompt is short and constrains the model to a single compact JSON object with fixed enum
values. `temperature` is 0 because this is a classification task, not a generation task.

Note on `max_tokens`: the model spends part of its budget on internal reasoning before emitting
content, so the ceiling is set well above the size of the JSON itself. A tight limit produces an
empty response with `finish_reason: length`.

## Output schema

```json
{
  "category":   "hvac | plumbing | general | other",
  "urgency":    "low | medium | high",
  "intent":     "service_request | quote_request | question | spam | other",
  "confidence": 0.0 - 1.0,
  "reason":     "one short sentence"
}
```

## Validation and routing

A Code node locates the OpenAI-style `choices[0].message.content` payload, strips any markdown
fences, parses the JSON, and validates every enum plus the confidence range. Nothing is trusted
from the model.

When valid, a routing step derives an operational action: service categories go to a field
queue, everything else to the front desk, and the urgency maps to an SLA bucket
(`high -> same_day`, `medium -> 48h`, `low -> next_week`).

## Failure behaviour

| Condition | Result |
|---|---|
| Missing `message` or `name` | `400` with `error: invalid_input` and the specific missing fields |
| Upstream 4xx (auth, config, bad request) | `502` with `error: upstream_failed`, no classification |
| Malformed / non-JSON model output | `502` with `error: malformed_llm_output` |
| Schema-invalid values | `502` with `error: schema_invalid` and the offending fields |
| Transient upstream error | retried by the node, then `502` if still failing |

A key design decision: **the workflow never fabricates a classification.** If the model output
cannot be parsed and validated, the caller gets an explicit error rather than a guessed category.
A plausible-looking wrong answer is worse than a visible failure.

## Retry policy

The LLM node is configured with `retryOnFail` and up to 3 total attempts with an 800 ms wait
between them. Errors are allowed to propagate so the retry actually engages, and `onError:
continueRegularOutput` keeps the run alive afterwards so the failure surfaces as a structured
`502` rather than a crashed execution. This is what lets the workflow distinguish a transient
failure from a permanent one.

## Test mode

`tests/test_cases.json` defines nine deterministic inputs covering the happy paths and each
failure mode. Because classification is non-deterministic in general, the assertions are on the
**enum fields and the routing decision**, not on the exact wording of `reason` or on
`confidence` beyond its valid range.

`tests/fault_fixture.py` is a local HTTP server that can return a permanent 503, a transient 503
then proxy to the real API, non-JSON prose, or JSON with invalid enums. Pointing the LLM node's
URL at it is how the failure paths are exercised deterministically without touching the real
credential.
