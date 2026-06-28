---
name: corsa-webhook-debugging
description: >
  Debug Corsa webhook delivery, signature verification, and event handling.
  Use when webhook signatures fail, events are missing, payloads are malformed,
  webhook handlers return errors, or you need to set up a webhook receiver.
---

# Corsa Webhook Debugging

You are a Corsa webhook specialist. Help developers set up, debug, and troubleshoot webhook integrations with the Corsa compliance platform.

For complete webhook headers, all 27 event types with payload structures, signature verification code, and failure diagnosis patterns, see [references/REFERENCE.md](references/REFERENCE.md).

## How Corsa Webhooks Work

1. You register a webhook in the Corsa dashboard (**Settings > Developers > Webhooks**) with a URL, event subscriptions, and an optional signing secret
2. When a subscribed event occurs, Corsa's delivery service POSTs the event payload to your URL
3. Your handler verifies the signature, parses the payload, and processes the event

### Delivery Behavior

| Property | Value |
|----------|-------|
| Method | `POST` |
| Content-Type | `application/json` |
| Timeout | 5 seconds |
| Redirects | Not followed (3xx = failure) |
| Retries | Up to 3 retries with 10s delay between attempts |
| HTTPS | Required — HTTP URLs are rejected |
| URL restrictions | Must have a TLD, no raw IPs, no explicit ports |

## Webhook Headers

Every delivery includes these headers:

| Header | SDK Constant | Description |
|--------|-------------|-------------|
| `x-hub-signature-256` | `WebhookSignatureHeader` | HMAC SHA256 signature: `sha256=<hex>` |
| `x-hook-id` | `WebhookIdHeader` | ID of the webhook configuration |
| `x-hook-delivery` | `WebhookDeliveryIdHeader` | Unique ID for this delivery attempt (use as idempotency key) |
| `x-hook-event` | `WebhookEventTypeHeader` | Event type (e.g., `individual_client.created`) |
| `x-request-id` | `WebhookRequestIdHeader` | Request trace ID for debugging |
| `x-request-origin` | `WebhookRequestOriginHeader` | Origin of the triggering request (`WEB` or `API`) |

## Signature Verification

The signature is an HMAC SHA256 of the **raw request body** using your webhook secret:

```
sha256=<hex digest of HMAC-SHA256(secret, rawBody)>
```

### Correct Express Setup

```typescript
import express from 'express';
import {
  WebhookSignatureHeader,
  verifyWebhookSignature,
} from '@corsa-labs/sdk';

const app = express();

// CRITICAL: Use express.raw() on the webhook route — NOT express.json()
app.use('/webhook', express.raw({ type: 'application/json' }));

app.post('/webhook', (req, res) => {
  const signature = req.headers[WebhookSignatureHeader] as string;
  const rawBody = (req.body as Buffer).toString('utf-8');
  const secret = process.env.WEBHOOK_SECRET!;

  if (!signature || !secret) {
    return res.status(400).send('Missing signature or secret');
  }

  // verifyWebhookSignature is SYNCHRONOUS (returns boolean, not a Promise)
  const isValid = verifyWebhookSignature(secret, rawBody, signature);

  if (!isValid) {
    return res.status(403).send('Invalid signature');
  }

  const event = JSON.parse(rawBody);
  console.log(`Event: ${event.type}, Delivery: ${req.headers['x-hook-delivery']}`);

  // Process event...

  res.status(200).send('OK');
});
```

### How Signing Works Under the Hood

```typescript
// Server-side (Corsa delivery service):
const signature = `sha256=${createHmac('sha256', secret).update(JSON.stringify(event)).digest('hex')}`;

// Client-side verification (SDK):
const expected = `sha256=${createHmac('sha256', secret).update(rawBody).digest('hex')}`;
// Compared using crypto.timingSafeEqual for constant-time comparison
```

## Event Types

All events follow the pattern `<entity>.<action>`:

| Entity | Events |
|--------|--------|
| Individual Client | `individual_client.created`, `individual_client.updated` |
| Corporate Client | `corporate_client.created`, `corporate_client.updated` |
| Alert | `alert.created`, `alert.updated` |
| Case | `case.created`, `case.updated` |
| Transaction | `transaction.created`, `transaction.updated` |
| Deposit | `deposit.created`, `deposit.updated` |
| Withdrawal | `withdrawal.created`, `withdrawal.updated` |
| Trade | `trade.created`, `trade.updated` |
| Individual Member | `individual_member.created`, `individual_member.updated` |
| Corporate Member | `corporate_member.created`, `corporate_member.updated` |
| Blockchain Wallet | `blockchain_wallet.created`, `blockchain_wallet.updated` |
| Bank Account | `bank_account.created`, `bank_account.updated` |
| Payment Account | `payment_account.created`, `payment_account.updated` |
| Checklist | `checklist.created`, `checklist.updated` |
| Attachment | `attachment.created`, `attachment.updated`, `attachment.deleted` |
| Form | `form_template.public_form_submitted` |

**Note:** The SDK `WebhookEventType` enum only contains a subset (clients, alerts, cases). For the full list above, compare `event.type` as a string.

## Payload Structure

### Created Events

```json
{
  "type": "individual_client.created",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "data": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "referenceId": "USER-12345",
    "entity": { /* full entity object */ }
  }
}
```

### Updated Events

```json
{
  "type": "individual_client.updated",
  "timestamp": "2024-01-15T11:00:00.000Z",
  "data": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "referenceId": "USER-12345",
    "updated": { /* only changed fields */ },
    "previousValues": { /* previous values of changed fields */ }
  }
}
```

**Note:** `previousValues` is present in actual payloads but not in the SDK `EntityUpdatedPayload` type. If typing strictly, extend the type or use a broader type.

## Debugging Guide

### Signature Verification Fails

| Cause | Fix |
|-------|-----|
| Body parsed as JSON before verification | Use `express.raw({ type: 'application/json' })` on the webhook route |
| Re-stringified body differs from original | Always verify against the raw bytes, never `JSON.stringify(JSON.parse(...))` |
| Wrong secret | Verify the secret matches what's configured in the Corsa dashboard |
| No secret configured on webhook | If the webhook was created without a secret, the `x-hub-signature-256` header is omitted entirely |
| Trimmed or modified body | Ensure no middleware modifies the raw body before verification |

### Events Not Received

| Cause | Fix |
|-------|-----|
| Webhook is inactive | Check webhook status in the Corsa dashboard — it must be active |
| Event type not subscribed | Verify the event type is in the webhook's event subscription list |
| URL validation failed | Must be HTTPS, have a valid TLD, no raw IPs, no explicit port numbers |
| DNS resolution failed | Ensure the URL's hostname resolves publicly |
| Handler responds slowly | Corsa has a 5s timeout — process asynchronously and respond 200 immediately |

### Receiving Duplicate Events

The same event may be delivered multiple times due to retries. Use the `x-hook-delivery` header as an idempotency key:

```typescript
const deliveryId = req.headers['x-hook-delivery'] as string;

if (await isAlreadyProcessed(deliveryId)) {
  return res.status(200).send('Already processed');
}
```

### Handler Keeps Failing

| Cause | Fix |
|-------|-----|
| 3xx redirect response | Corsa does not follow redirects — ensure the URL returns 200 directly |
| Timeout (>5s) | Offload processing to a queue/job and return 200 immediately |
| TLS/certificate errors | Ensure valid TLS certificate on your endpoint |
| Retries exhausted | After 4 total attempts (initial + 3 retries), delivery stops |

### Testing Locally

For local development, use a tunnel service to expose your local server:

```bash
# Start your webhook handler
node webhook-server.js

# Expose via tunnel (e.g., ngrok, cloudflare tunnel)
ngrok http 3000
# Use the HTTPS URL from the tunnel as your webhook URL
```

## SDK Typed Event Helpers

```typescript
import {
  IndividualClientCreatedEvent,
  IndividualClientUpdatedEvent,
  CorporateClientCreatedEvent,
  CorporateClientUpdatedEvent,
  AlertCreatedEvent,
  AlertUpdatedEvent,
  WebhookEventType,
} from '@corsa-labs/sdk';
```

**Coverage note:** The SDK provides typed event aliases for individual clients, corporate clients, and alerts only. Case events exist in the `WebhookEventType` enum (`CASE_CREATED`, `CASE_UPDATED`) but have no typed aliases. For cases, transactions, deposits, trades, members, wallets, bank accounts, payment accounts, checklists, attachments, and forms, parse the payload as a plain object and switch on `event.type` as a string:

```typescript
const event = JSON.parse(rawBody) as { type: string; timestamp: string; data: Record<string, unknown> };
```

## Links

- [Webhook Setup Guide](https://docs.corsa.finance/webhooks)
- [Event Payloads Reference](https://docs.corsa.finance/webhooks/event-payloads)
- [Webhook Handler Example](https://docs.corsa.finance/sdk/webhook-example)
- [API Reference](https://api.corsa.finance/api-spec.json)
