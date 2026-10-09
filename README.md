# PulseFit AI Support Automation

n8n assessment workflow with explicit `mock` and `ollama` modes for classification and response drafting. All input data is synthetic. Billing operations are simulations; no emails are sent and no real subscriptions or payments are changed.

## Run

1. Install Docker. The default `mock` mode needs no Ollama, model downloads, API keys, or external AI service.
2. Start n8n with the Compose configuration in this repository: `docker compose up -d`. If the existing `gismart-n8n` container is running on port 5678, stop it first; do not start both instances on the same port.
3. Open http://localhost:5678 and create your local owner account. Import `error-workflow.json`, then `workflow.json`. Publish the error workflow (required by the tested n8n version). In the main workflow Settings select **PulseFit — Execution Error Handler** as the Error workflow: imported IDs can differ across instances.
4. Execute the workflow manually. The Sample Input node includes `sample-input.json` as six separate items.
5. Review node outputs and the JSON logs in the `logs` directory.

Tested n8n version: 2.42.6. The Configuration node centralizes the model, temperature, classifier/response prompts, log directory, and Ollama endpoint. The default endpoint is `http://host.docker.internal:11434/api/chat`, which refers to the host from Docker Desktop. Both AI calls request JSON, temperature zero, and disabled thinking. No API keys are needed for this local setup. If running n8n outside Docker, change the endpoint to localhost.

## Design

- Validate required fields before calling AI; malformed tickets use an independent failure path.
- Normalize email deterministically: trim, lowercase, remove Gmail plus aliases. Preserve dots.
- AI classifies intent only. Validate the returned intent and confidence before the Switch node.
- Separate cancellation, cancellation with billing clarification, refund, technical issue, and other branches.
- Count every subscription matching the product, irrespective of status. Check active subscriptions separately for cancellation.
- Cancel only with exactly one active match. Zero means not_executed; multiple means requires_review.
- Refund eligibility is deliberately the assessment rule: any matching subscription, including inactive, is sufficient. It is not a production refund policy.
- Technical KB matching is deterministic keyword matching. Unknown problems require review. General questions go to general support.
- Each branch returns a consistent action_type, action_status, details, and matching_subscription_count.
- Branches converge on common response-generation nodes without a barrier waiting for empty branches.
- The second AI call receives the customer request and actual branch result. It never selects or executes billing actions.
- Validate response structure and perform a basic check for internal metadata and unsupported success claims. This check is limited, not a proof of semantic correctness.

## Logging and errors

Every result is written to a separate JSON file with ticket ID and execution ID in its name. It includes timestamp, ticket_id, detected_intent, matching_subscription_count, action_type, action_status, execution_status, and error_message, plus context and the generated draft.

Business outcomes such as not_executed or requires_review can have execution_status Success: the workflow handled the case correctly. Invalid input, invalid classifier output, and generation failures have execution_status Failed. Response-generation failure produces a conservative fallback draft while retaining the action result and recording the error.

Both HTTP nodes use native Retry On Fail: at most three attempts with a fixed 1,000 ms pause. After exhausted retries they continue with the error. Native retries are not selective by HTTP status, so permanent request errors can also be retried; this is acceptable for side-effect-free inference in this assessment, but production should distinguish 429/5xx/timeouts from permanent 4xx errors. A node retry can repeat earlier item requests; do not apply this pattern to billing APIs without idempotency. HTTP nodes continue after individual request errors. Classifier failures skip action processing and go to logging. An invalid input ticket does not prevent other tickets from being processed. The assessment uses valid subscription objects; branch checks catch exceptions per ticket and route failed items directly to the error log. The separate Error Trigger workflow captures unhandled execution failures and writes a compact execution-error JSON record. Handled per-ticket failures do not trigger it. It depends on the same storage, so a storage outage still requires an independent alert destination in production.

## Expected sample outcomes

| Ticket | Intent | Result |
|---|---|---|
| 10001 | cancellation | simulated_success, count 1, normalized anna.test@gmail.com |
| 10002 | cancellation_with_billing_clarification | simulated_success, count 1, PayPal retained; charge explanation not invented |
| 10003 | refund | simulated_success, count 2 including the inactive PulseFit subscription |
| 10004 | technical_issue | troubleshooting context for freezing; no billing action |
| 10005 | none | Failed: message is null; no AI or billing action |
| 10006 | other | queued for general support; no invented workout advice |

## Low confidence

In production, calibrate confidence against labeled tickets instead of treating model confidence as probability. Route uncertain or ambiguous requests to a human, particularly before financial actions. Choose thresholds from observed error rates and business risk.

## Zendesk integration and credentials

Use a Zendesk trigger/webhook to send ticket identifiers to an authenticated endpoint. Retrieve current ticket/account data through scoped API credentials and validate event authenticity. Queue and deduplicate events; do not trust arbitrary customer-supplied subscription identifiers. Update tickets with private notes or drafts according to the approved process.

Store credentials in n8n credential storage or a secret manager, never in exported workflow JSON or Git. Apply least privilege, TLS, rotation, redacted logs, and backend authorization. Backend billing APIs enforce eligibility and return authoritative results; the LLM cannot bypass these rules.

## Three production priorities for 10,000 tickets/month

1. **Financial safety:** authoritative backend eligibility, account authorization, idempotency keys, audit records, and review gates for ambiguous/high-risk requests. This prevents duplicate or unauthorized cancellations/refunds.
2. **Reliability and visibility:** per-ticket isolation, bounded retries with backoff, queues, dead-letter processing, durable state, alerts, and reconciliation after uncertain API outcomes. This prevents silently lost or repeatedly executed actions.
3. **AI quality and privacy:** labeled evaluation cases, calibrated escalation, strict output validation, prompt-injection defenses, minimum necessary data, and monitoring of customer-facing claims. Version prompts/models and gate changes on regression checks.

## Limitations

The workflow has no real Zendesk/payment integration, idempotency store, calibrated confidence policy, or production-grade reply verification. KB matching is small and English-only. Generated replies are drafts demonstrating simulated outcomes and are never sent. In ollama mode, HTTP calls depend on local model availability and machine speed. Keep n8n volumes/logs backed up; use managed persistent storage and restricted retention in production.

## Verification

Run `node verify.cjs` for 21 deterministic checks covering normalization, missing data, no/multiple active subscriptions, refund product matching, KB misses, isolated branch/classifier failures, and unsafe AI reply fallback. This harness checks the exported Code-node logic; it does not replace an actual n8n/Ollama end-to-end run.

## Optional bonuses implemented

- Native bounded retries for both AI HTTP nodes; verified against a local endpoint returning 503 twice and 200 on the third request.
- Shared Configuration node for non-secret model settings, prompts, endpoint, and log directory. Credentials would remain separate.
- Separate Error Trigger workflow selected in main workflow settings for unhandled execution failures.
- Additional classifier/response output validation and truthful response fallback.
- Clear branch-based modular structure, but no business-logic sub-workflows; these were deferred to keep the assessment small.

Verification of the error workflow: a temporary local webhook workflow intentionally failed at Stop and Error; the separate error workflow persisted the failure with execution ID and failed node. The temporary webhook was unpublished afterwards. Manual editor executions may not invoke the automatic Error Trigger; verify with an automatic trigger.

## Configuration audit

The live main workflow matches the submitted node definitions, connections and settings. Docker Compose configuration validates. The error handler is published and linked. Fixed double-escaped regexp whitespace/digit tokens, login-specific fallback, minimum ticket ID/product type checks, and unsafe internal wording in customer replies. Current standalone Docker container stores logs in its writable filesystem; copied logs are in the project folder, and the supplied Compose file mounts a host log directory for container-replacement durability.

## Reproducible mock mode and live Ollama mode

The Configuration node defaults to `ai_mode: "mock"`. Both AI calls are explicitly bypassed through IF nodes. The mock classifier returns fixed labeled JSON fixtures for the eight valid requests in `sample-input.json` and `additional-input.json`. Unknown or modified subject/message fixtures produce an explicit error instead of inventing a classification. Confidence 1 is fixture metadata, not a measured AI probability.

The mock response node simulates a model response from the actual deterministic branch result. Validation, routing, subscription checks, action simulation, response validation and persistent logging still execute normally. Every ticket log includes `ai_mode`, including input-validation failures. Mock mode is not evidence of real-model accuracy.

To use a real model, install/start Ollama, run `ollama pull qwen3:14b`, and change only `ai_mode` to `"ollama"` in Configuration. Adjust `model` or `ollama_url` if necessary. No automatic switch to mock happens on API failure: retry/error/fallback handling remains visible. Invalid mode values are rejected.

To exercise response fallback without any model, set `mock_invalid_response_ticket_id` to `10003` and run the sample batch. Only the simulated AI reply for that ticket becomes malformed JSON. The refund action result remains simulated_success, response_fallback becomes true, the error is logged, and other tickets continue. Restore the option to null afterwards; default mock output does not inject errors.

Real-model verification results are preserved separately in `ollama-audit-output.json`. Mock customer text and routing are reproducible; timestamps and execution/file IDs naturally change between runs.

Mock verification: independent six- and nine-ticket n8n executions produced identical customer drafts for the shared tickets. Nine-case mock execution used an intentionally unreachable model URL and neither Ollama HTTP node executed. Eight valid tickets succeeded; the null-message ticket failed as expected. The injected malformed reply for ticket 10003 triggered fallback while all other valid tickets continued. Results: `mock-audit-output.json` and `mock-fallback-output.json`.
