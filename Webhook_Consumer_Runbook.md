# Webhook Consumer Runbook

## Purpose
This runbook is for external teams receiving report batch webhooks from Vitract.

Use this when your team has already provided a webhook URL and needs to safely receive, verify, and process webhook events in production.

## Event Types
Your endpoint can receive these event types:

- `report.batch.completed`
- `report.batch.partial_failed`
- `report.batch.failed`

## Expected HTTP Request
Vitract sends an HTTP `POST` request to your endpoint.

Example endpoint placeholder:

- `https://{{your-app.com}}/webhooks/vitract/report-jobs`

Headers sent by Vitract:

- `Content-Type: application/json`
- `User-Agent: vitract-report-handler/webhooks`
- `X-Vitract-Event-Type: {{event-type}}`
- `X-Vitract-Delivery-Id: {{delivery-id-uuid}}`
- `X-Vitract-Signature: t={{unix-timestamp}},v1={{hmac-hex}}`

## Payload Contract
Example payload:

```json
{
  "id": "evt_{{deterministic-event-id}}",
  "type": "report.batch.completed",
  "created": "{{rfc3339-timestamp}}",
  "data": {
    "batch": {
      "batchId": "{{batch-id-uuid}}",
      "status": "completed",
      "totalItems": 3,
      "succeededItems": 3,
      "failedItems": 0,
      "validationFailedItems": 0,
      "duplicateItems": 0,
      "createdAt": "{{rfc3339-timestamp}}",
      "updatedAt": "{{rfc3339-timestamp}}",
      "startedAt": "{{rfc3339-timestamp}}",
      "completedAt": "{{rfc3339-timestamp}}",
      "statusUrl": "/api/report/jobs/{{batch-id-uuid}}",
      "itemsUrl": "/api/report/jobs/{{batch-id-uuid}}/items"
    },
    "items": [
      {
        "id": "{{item-id-uuid}}",
        "batchId": "{{batch-id-uuid}}",
        "ordinal": 1,
        "kitId": "{{kit-id}}",
        "status": "completed",
        "attemptCount": 1,
        "maxAttempts": 3,
        "pdfUrl": "{{report-pdf-url}}"
      }
    ]
  }
}
```

## Consumer Setup Steps
1. Expose a public HTTPS endpoint that accepts `POST`.
2. Read the raw request body exactly as received.
3. Verify `X-Vitract-Signature` with your shared secret.
4. Reject invalid signature with `401` or `403`.
5. Process valid payload idempotently.
6. Return `2xx` quickly after durable acknowledgment.

## Signature Verification
The signature format is:

- `t={{unix-timestamp}},v1={{hex-hmac-sha256}}`

Compute expected HMAC-SHA256 over:

- `{{timestamp}}.{{raw-request-body}}`

Use your shared secret as HMAC key.

### Verification Pseudocode
```text
header = X-Vitract-Signature
parse t and v1
if abs(now - t) > {{allowed-clock-skew-seconds}} then reject
signed_payload = t + "." + raw_body
expected = HMAC_SHA256_HEX(secret, signed_payload)
constant_time_compare(expected, v1)
```

## Idempotency Requirements
Treat webhooks as at-least-once delivery.

Use one of these as your idempotency key:

- `X-Vitract-Delivery-Id`
- payload `id`

If already processed, return `200` and do not process again.

## Response Expectations
Your endpoint should:

- Return `2xx` when accepted.
- Return non-`2xx` only when you want Vitract to retry.

Vitract retry behavior:

- Retries on network errors, `429`, and `5xx` responses.
- Does not retry on most `4xx` responses.
- Retries until max attempts, then marks delivery failed.

## Operational Checklist
Before production:

1. Confirm endpoint URL is reachable: `https://{{your-app.com}}/webhooks/vitract/report-jobs`.
2. Confirm TLS certificate is valid.
3. Confirm signature verification is implemented.
4. Confirm idempotency store is implemented.
5. Confirm payload logging redacts sensitive fields where required.
6. Confirm alerting on repeated `5xx` responses.

## Recommended Security Controls
- Enforce HTTPS only.
- Enforce signature verification on every request.
- Enforce timestamp freshness window.
- Apply rate limiting and request size limits.
- Restrict outbound processing side effects until verification passes.

## Troubleshooting
### Invalid Signature
- Verify you used raw request body, not re-serialized JSON.
- Verify shared secret value is correct.
- Verify timestamp parsing and clock synchronization.

### Duplicate Events
- Confirm idempotency key is persisted before downstream side effects.

### Missing Events
- Check your endpoint availability.
- Check non-`2xx` response rates.
- Ask Vitract to review delivery logs for `{{batch-id-uuid}}`.

## Handover Data You Should Receive from Vitract
- `{{webhook-target-url-confirmed}}`
- `{{signing-secret}}`
- `{{subscribed-event-types}}`
- `{{production-go-live-date}}`
- `{{support-contact}}`
