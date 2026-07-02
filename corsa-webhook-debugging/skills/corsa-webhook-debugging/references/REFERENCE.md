# Corsa Webhook Debugging — Full Reference

Complete webhook headers, event types, payload structures, signature verification code, and failure diagnostic patterns.

---

## Webhook Headers

Every Corsa webhook delivery includes these headers:

| Header | Description |
|--------|-------------|
| `x-hub-signature-256` | HMAC SHA256 signature — format: `sha256=<hex>`. Use this to verify the payload. |
| `x-hook-id` | Your webhook configuration ID. |
| `x-hook-delivery` | Unique delivery ID — use as an **idempotency key** to deduplicate retries. |
| `x-hook-event` | The event type (e.g., `individual_client.created`). |
| `x-request-id` | Request trace ID for support. |
| `x-request-origin` | `WEB` (triggered from the app UI) or `API` (triggered from an API call). |

---

## Delivery Behavior

| Property | Value |
|----------|-------|
| Timeout | 5 seconds — your handler must respond within 5s |
| Redirects | Not followed — 3xx responses count as failures |
| Retries | Up to 3 retries with ~10s delay |
| Total attempts | 4 (initial + 3 retries) |
| Protocol | HTTPS only |
| URL format | Must have a TLD, no raw IPs, no explicit port numbers |
| Success | Any 2xx response |

---

## Signature Verification

### Node.js (Express)

```typescript
import express from 'express';
import crypto from 'crypto';

const app = express();

// CRITICAL: use express.raw() to get the raw buffer — JSON.parse alters whitespace
app.post('/webhooks/corsa', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-hub-signature-256'] as string;
  const secret = process.env.CORSA_WEBHOOK_SECRET!;

  if (!verifySignature(req.body, secret, signature)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const event = JSON.parse(req.body.toString());
  const deliveryId = req.headers['x-hook-delivery'] as string;

  // Deduplicate using deliveryId
  // Process event...

  res.status(200).json({ received: true });
});

function verifySignature(body: Buffer, secret: string, signature: string): boolean {
  const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(body).digest('hex');
  try {
    return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
  } catch {
    return false;
  }
}
```

### Using the SDK helper

```typescript
import { verifyWebhookSignature } from '@corsa-labs/sdk';

// verifyWebhookSignature is synchronous — returns boolean, not a Promise
// Parameter order: (secret, rawBody, signatureHeader)
const isValid = verifyWebhookSignature(
  process.env.CORSA_WEBHOOK_SECRET!,        // secret
  req.body,                                 // raw Buffer — must use express.raw()
  req.headers['x-hub-signature-256']        // header value
);
```

### Why `express.raw()` is required

Parsing the body with `express.json()` before verification re-serializes the JSON, which can alter whitespace and field ordering. The HMAC is computed over the **exact bytes Corsa sent** — any modification breaks the signature. Always use `express.raw()` and verify before parsing.

---

## All 32 Event Types

### Clients (4 events)

| Event | Trigger |
|-------|---------|
| `individual_client.created` | Individual client created |
| `individual_client.updated` | Individual client updated |
| `corporate_client.created` | Corporate client created |
| `corporate_client.updated` | Corporate client updated |

### Members (4 events)

| Event | Trigger |
|-------|---------|
| `individual_member.created` | Individual member (UBO/Director/Signatory) created |
| `individual_member.updated` | Individual member updated |
| `corporate_member.created` | Corporate member created |
| `corporate_member.updated` | Corporate member updated |

### Alerts & Cases (4 events)

| Event | Trigger |
|-------|---------|
| `alert.created` | Alert created |
| `alert.updated` | Alert updated |
| `case.created` | Case created |
| `case.updated` | Case updated |

### Transactions (2 events)

| Event | Trigger |
|-------|---------|
| `transaction.created` | Transaction created |
| `transaction.updated` | Transaction updated |

### Financial Operations (6 events)

| Event | Trigger |
|-------|---------|
| `deposit.created` | Deposit created |
| `deposit.updated` | Deposit updated |
| `withdrawal.created` | Withdrawal created |
| `withdrawal.updated` | Withdrawal updated |
| `trade.created` | Trade created |
| `trade.updated` | Trade updated |

### Instruments (6 events)

| Event | Trigger |
|-------|---------|
| `blockchain_wallet.created` | Blockchain wallet created |
| `blockchain_wallet.updated` | Blockchain wallet updated |
| `bank_account.created` | Bank account created |
| `bank_account.updated` | Bank account updated |
| `payment_account.created` | Payment account created |
| `payment_account.updated` | Payment account updated |

### Checklists (2 events)

| Event | Trigger |
|-------|---------|
| `checklist.created` | Checklist created for an entity |
| `checklist.updated` | Checklist or checklist item updated |

### Attachments (3 events)

| Event | Trigger |
|-------|---------|
| `attachment.created` | File attachment uploaded or linked to an entity |
| `attachment.updated` | Attachment metadata updated |
| `attachment.deleted` | Attachment deleted |

### Forms (1 event)

| Event | Trigger |
|-------|---------|
| `form_template.public_form_submitted` | Customer submitted a public form |

---

## Payload Structures

### Created Event

```json
{
  "type": "individual_client.created",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "data": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "referenceId": "USER-12345",
    "entity": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "referenceId": "USER-12345",
      "accountStatus": "APPROVED",
      "activityStatus": "ACTIVE",
      "general": {
        "firstName": "John",
        "lastName": "Doe",
        "dateOfBirth": "1985-06-15",
        "citizenship": "USA"
      },
      "currentRisk": { "score": 15, "level": "LOW", "calculatedAt": "2024-01-15T10:00:00Z" },
      "createdAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  }
}
```

### Updated Event

```json
{
  "type": "individual_client.updated",
  "timestamp": "2024-02-10T08:15:00.000Z",
  "data": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "referenceId": "USER-12345",
    "updated": {
      "accountStatus": "FROZEN",
      "currentRisk": { "score": 85, "level": "HIGH", "calculatedAt": "2024-02-10T08:15:00Z" }
    },
    "previousValues": {
      "accountStatus": "APPROVED",
      "currentRisk": { "score": 15, "level": "LOW", "calculatedAt": "2024-01-15T10:00:00Z" }
    }
  }
}
```

### Alert Created

```json
{
  "type": "alert.created",
  "timestamp": "2024-01-15T14:00:00.000Z",
  "data": {
    "id": "alert-uuid-123",
    "referenceId": null,
    "entity": {
      "id": "alert-uuid-123",
      "category": "TRANSACTION_MONITORING",
      "priority": "HIGH",
      "status": "NEW",
      "clientIds": ["123e4567-e89b-12d3-a456-426614174000"],
      "transactionIds": ["txn-uuid-456"],
      "triggeredRuleIds": ["rule-uuid-789"],
      "createdAt": "2024-01-15T14:00:00Z"
    }
  }
}
```

### Alert Updated

```json
{
  "type": "alert.updated",
  "timestamp": "2024-01-15T15:00:00.000Z",
  "data": {
    "id": "alert-uuid-123",
    "referenceId": null,
    "updated": {
      "status": "IN_REVIEW",
      "assigneeId": "analyst-user-uuid"
    },
    "previousValues": {
      "status": "NEW",
      "assigneeId": null
    }
  }
}
```

---

## Idempotency

Use `x-hook-delivery` to deduplicate events. Corsa may deliver the same event more than once on retry. Store processed delivery IDs and skip duplicates:

```typescript
const deliveryId = req.headers['x-hook-delivery'] as string;

if (await db.hasProcessed(deliveryId)) {
  return res.status(200).json({ skipped: true });
}

// process event...

await db.markProcessed(deliveryId);
res.status(200).json({ received: true });
```

---

## Common Failure Diagnosis

### Signature verification fails

**Checklist (in order):**

1. **Wrong body parser** — are you using `express.raw()`? `express.json()` re-serializes JSON and breaks the signature.
2. **Wrong secret** — verify the secret matches the one shown in Settings → Developers → Webhooks. The secret is only shown once.
3. **Modified body** — middleware between Express and your handler shouldn't transform the body.
4. **Encoding mismatch** — ensure you're comparing byte buffers, not strings. `crypto.timingSafeEqual` requires same-length Buffers.
5. **No signing secret** — if you created the webhook without a secret, `x-hub-signature-256` will not be present.

```typescript
// Correct: raw buffer comparison
const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(body).digest('hex');
crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));

// Wrong: parsing before verification
const body = JSON.parse(req.body); // ← breaks signature
```

### Events not received

1. Webhook is **inactive** — check Settings → Developers → Webhooks → status.
2. Event type **not subscribed** — verify the event is in your subscription list.
3. **URL validation failed** — URL must be HTTPS, have a TLD, no raw IPs, no explicit ports.
4. **DNS resolution failure** — your endpoint must be publicly reachable.
5. **All 4 attempts failed** — check your server logs for the period around the event timestamp.

### Handler timeout

Your handler must respond within **5 seconds**. Long-running processing must be done asynchronously:

```typescript
app.post('/webhooks/corsa', express.raw({ type: 'application/json' }), async (req, res) => {
  if (!verifySignature(req.body, secret, req.headers['x-hub-signature-256'])) {
    return res.status(401).send();
  }

  // Respond immediately — process async
  res.status(200).json({ received: true });

  const event = JSON.parse(req.body.toString());
  await queue.enqueue(event); // process in background
});
```

### Redirects not followed

If your endpoint returns a `301`, `302`, or any 3xx, Corsa treats it as a failure. The webhook URL must return a `2xx` directly — update your server to respond at the exact registered URL without redirecting.

### Duplicate events

Corsa may retry on network errors even if your server received the request. Always deduplicate on `x-hook-delivery`.
