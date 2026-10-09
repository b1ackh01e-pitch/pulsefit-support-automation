# Verification report

Actual n8n end-to-end execution with Ollama qwen3:14b. Original six tickets plus three additional cases. All nine produced the expected classification/action outcome and log records.

| Ticket | Intent | Count | Action status | Execution status | Fallback |
|---|---|---|---|---|---|
| 10001 | cancellation | 1 | simulated_success | Success | False |
| 10002 | cancellation_with_billing_clarification | 1 | simulated_success | Failed | True |
| 10003 | refund | 2 | simulated_success | Failed | True |
| 10004 | technical_issue | 0 | not_executed | Success | False |
| 10005 | None | None | not_executed | Failed | False |
| 10006 | other | 0 | queued | Failed | True |
| 20001 | cancellation | 1 | not_executed | Success | False |
| 20002 | cancellation | 2 | requires_review | Failed | True |
| 20003 | refund | 0 | not_executed | Failed | True |

Failed execution_status for generated replies indicates the original model draft was rejected and a truthful deterministic fallback was used. It does not undo a successful simulated billing action. Ticket 10005 is the expected input validation failure.

Also passed 19 deterministic regression checks, Compose syntax validation, live/export configuration comparison, and published error handler configuration inspection. Earlier bonus checks verified 503 → 503 → 200 retry recovery and automatic logging of a deliberately failed webhook execution.

Known limits: regular-expression response guards are conservative and incomplete; retries are not selective; current standalone container logs need backup before container deletion. No real billing API or email delivery was exercised.

## Explicit mock mode verification

Default mock mode was tested in actual n8n executions with six and nine tickets. No Ollama HTTP nodes executed, including when the configured model endpoint was deliberately unreachable. Eight valid tickets succeeded; invalid ticket 10005 failed as expected. Customer drafts match between the independent runs. A separate malformed-response test for 10003 produced a truthful fallback and did not stop other tickets. The regression harness now passes 21 checks. Historical real-model results above are preserved separately in ollama-audit-output.json.
