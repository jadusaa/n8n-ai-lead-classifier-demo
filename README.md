# n8n AI Lead Classifier

I built this small n8n workflow to show a real LLM integration with structured output, schema
validation, retries, and explicit failure handling. It uses the Hive API with GLM 5.3 Flash and
synthetic lead messages.

Files in this repo are flat: `ai-lead-classifier.json` is the workflow export, `ARCHITECTURE.md`
documents the design, `test_cases.json` / `results.json` hold the tests and real results, and
`evidence/` has the screenshots and captured responses.

## What it does

`Webhook -> Validate -> LLM -> Parse JSON -> Validate Schema -> Route -> Respond`

## What it proves

- real LLM API call from n8n (Hive API, model `zai-org/glm-5.3-flash`, OpenAI-compatible)
- structured JSON classification (category / urgency / intent / confidence / reason)
- schema validation before anything is trusted
- category / urgency routing into a queue and an SLA bucket
- retry/backoff
- malformed-output handling
- forced-failure testing
- **9/9 acceptance tests**

## Reliability

- the retry test observed **3 attempts** (initial + 2 retries) before failure
- the forced-503 test returned an explicit failure — **no fabricated classification**
- a transient 503 **recovered successfully** on retry against the real API
- the workflow refuses to return a guessed answer when model output is malformed

A design rule worth stating: this workflow never fabricates a result. If the model output cannot
be parsed and validated, the caller gets an explicit error instead of a plausible-looking guess.

## Tech

- n8n
- JavaScript
- REST API
- Webhooks
- JSON
- Hive API (OpenAI-compatible chat completions)
- GLM 5.3 Flash

## Scope

Self-built portfolio demo using synthetic data, not client work. The API key lives in n8n's
encrypted credential store — the exported workflow references a credential by id and contains
no secret.

## Evidence

![Workflow canvas](evidence/01_workflow_canvas.png)
![Successful execution](evidence/02_successful_execution.png)
