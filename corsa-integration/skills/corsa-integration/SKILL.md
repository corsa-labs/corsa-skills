---
name: corsa-integration
description: >
  Guide developers through Corsa compliance API integration using the
  @corsa-labs/sdk. Covers SDK setup, authentication, data ingestion order,
  webhook handling, and error handling. Use when integrating
  with Corsa, setting up the Corsa SDK, ingesting clients/transactions/alerts,
  configuring webhooks, or handling API errors.
---

# Corsa Integration Assistant

You are a Corsa integration engineer. Help developers integrate with the Corsa compliance API using the `@corsa-labs/sdk` TypeScript SDK or the REST API directly. Always prefer the SDK when the developer is using TypeScript/JavaScript.

For full method signatures, code templates, and the webhook event catalog, see [references/REFERENCE.md](references/REFERENCE.md).

## Quick Start

### Install

```bash
npm install @corsa-labs/sdk
# or
yarn add @corsa-labs/sdk
```

Requires Node.js 18+ and optionally TypeScript 4.7+.

### Initialize the Client

```typescript
import { CorsaClient } from '@corsa-labs/sdk';

const client = new CorsaClient({
  BASE: "https://api.corsa.finance",
  HEADERS: {
    Authorization: `Bearer ${process.env.API_TOKEN}:${process.env.API_SECRET}`
  }
});
```

### Base URLs

| Region | URL |
|--------|-----|
| US | `https://api.corsa.finance` |
| EU | `https://api.eu.corsa.finance` |

### Authentication

All requests use Bearer token authentication. The token is formed by combining the API Token and API Secret with a colon:

```
Authorization: Bearer <API_TOKEN>:<API_SECRET>
```

Create API keys in the Corsa dashboard under **Developers Hub > API Keys**. The API Secret is shown only once — store it securely.

### Rate Limits

500 requests per 60 seconds per user. When exceeded, the API returns `429 Too Many Requests`. Respect the `Retry-After` header before retrying.

## Data Model

Corsa organizes compliance data into interconnected entities. Understanding the relationships is critical for correct ingestion.

```
Clients (Individual / Corporate)
├── Members (UBOs, Directors, Signatories) — corporate clients only
├── Bank Accounts
├── Blockchain Wallets
├── Sessions (device fingerprinting, IP geolocation)
└── Operations
    ├── Deposits → Transactions
    ├── Withdrawals → Transactions
    ├── Trades → Transactions
    └── Transfers → Transactions

Alerts ← linked to Clients and/or Transactions
└── Cases ← escalated from Alerts, linked to Clients/Transactions/Alerts
```

**Key relationships:**
- Clients are the foundation — all other entities link back to them
- Operations (Deposits, Withdrawals, Trades) contain Transactions
- Alerts can be associated with both Clients and Transactions
- Cases group related Alerts for investigation

## Ingestion Order

Follow this order. Entities reference their parents, so parents must exist first.

1. **Clients** — Individual and Corporate clients (the foundation)
2. **Members** — UBOs, directors, signatories for corporate clients
3. **Accounts & Wallets** — Bank accounts, blockchain wallets, and payment accounts, associated with clients
4. **Sessions** — Client sessions with device/IP data for fraud detection
5. **Operations** — Deposits, Withdrawals, Trades, and Transfers (reference the initiating client by ID)
6. **Alerts & Cases** — External alerts and investigation cases (reference clients/transactions)
7. **Verifications** — KYC/KYB results from identity providers (SumSub, Persona, etc.)

## SDK Services

The `CorsaClient` exposes 21 services. Access them as properties on the client instance:

| Service | Access | Purpose |
|---------|--------|---------|
| `clients` | `client.clients` | Individual & Corporate client CRUD, upsert |
| `members` | `client.members` | UBO/Director/Signatory management |
| `deposits` | `client.deposits` | Deposit operations |
| `withdrawals` | `client.withdrawals` | Withdrawal operations |
| `transfers` | `client.transfers` | Peer-to-peer transfer operations between clients |
| `trades` | `client.trades` | Trade operations, fill appending |
| `transactions` | `client.transactions` | Transaction retrieval and status updates |
| `alerts` | `client.alerts` | Alert CRUD, batch create, bulk ops |
| `cases` | `client.cases` | Case CRUD, bulk ops, entity association |
| `bankAccounts` | `client.bankAccounts` | Bank account CRUD and client association |
| `blockchainWallets` | `client.blockchainWallets` | Wallet CRUD and client association |
| `paymentAccounts` | `client.paymentAccounts` | Payment account CRUD and client association |
| `sessions` | `client.sessions` | Session CRUD, client session listing |
| `rules` | `client.rules` | Compliance rule lifecycle (draft/activate/disable) |
| `ruleTemplates` | `client.ruleTemplates` | Browse and copy rule templates |
| `evaluation` | `client.evaluation` | Evaluate rules against transactions |
| `checklists` | `client.checklists` | Checklist and template management |
| `attachments` | `client.attachments` | File upload, download URLs, entity linking |
| `subDispositions` | `client.subDispositions` | Custom sub-disposition CRUD |
| `externalRules` | `client.externalRules` | External vendor rule management |
| `platform` | `client.platform` | Platform configuration |

For the full method reference for each service, see [references/REFERENCE.md](references/REFERENCE.md).

## Common Patterns

### referenceId and Upsert

The `referenceId` is the entity's identifier in your internal system. You can use either the Corsa-generated `id` or your `referenceId` when fetching, updating, or performing other operations. Most creation endpoints support `upsert=true` — when an entity with the same `referenceId` already exists, it is updated instead of duplicated:

```typescript
const individual = await client.clients.createIndividualClient(
  {
    referenceId: "USER-12345",
    accountStatus: "APPROVED",
    activityStatus: "ACTIVE",
    general: {
      firstName: "John",
      lastName: "Doe",
      dateOfBirth: "1985-06-15",
      citizenship: "USA",
    },
    contact: {
      emailAddress: "john.doe@example.com",
    },
  },
  true // upsert
);
```

### Linking Operations to Clients

The `initiatedBy` field on operations accepts the Corsa-generated `id` of the client who initiated the operation:

```typescript
const deposit = await client.deposits.createDeposit({
  referenceId: "DEP-001",
  initiatedBy: individual.id, // Corsa client ID from creation response
  initiatedAt: "2024-01-15T08:30:00Z",
  depositTransaction: {
    referenceId: "TX-001",
    amount: { amount: 1.5, currency: "BTC", netAmount: 1.5 },
    from: { walletAddress: "0xSource..." },
    to: { walletAddress: "0xDest..." },
    blockchainNetworkId: "bitcoin-mainnet",
    statusHistory: [{ type: "SUCCESS", timestamp: "2024-01-15T08:30:00Z" }],
  },
});
```

### Error Handling

```typescript
import { ApiError } from '@corsa-labs/sdk';

try {
  await client.clients.getIndividualClient("nonexistent-id");
} catch (error) {
  if (error instanceof ApiError) {
    console.error(`Status: ${error.status}`);
    console.error(`Body: ${JSON.stringify(error.body)}`);
    console.error(`Request ID: ${error.requestId}`);

    if (error.status === 429) {
      // Rate limited — wait and retry
    }
  }
}
```

## Webhooks

### Setup

1. In the Corsa dashboard, go to **Settings > Developers > Webhooks**
2. Click **+ Add webhook**
3. Select the events to subscribe to
4. Provide your server URL and a signing secret

### Signature Verification

Always verify webhook signatures to confirm authenticity:

```typescript
import express from 'express';
import {
  WebhookEvent,
  WebhookEventType,
  WebhookSignatureHeader,
  verifyWebhookSignature,
} from '@corsa-labs/sdk';

const app = express();
app.use('/webhook', express.raw({ type: 'application/json' }));

app.post('/webhook', (req, res) => {
  const signature = req.headers[WebhookSignatureHeader] as string;
  const rawBody = (req.body as Buffer).toString('utf-8');
  const secret = process.env.WEBHOOK_SECRET!;

  if (!verifyWebhookSignature(secret, rawBody, signature)) {
    return res.status(403).send('Invalid signature');
  }

  const event = JSON.parse(rawBody) as { type: string; timestamp: string; data: Record<string, unknown> };

  switch (event.type) {
    case WebhookEventType.INDIVIDUAL_CLIENT_CREATED:
    case WebhookEventType.ALERT_CREATED:
    case WebhookEventType.CASE_CREATED:
      // Handle event
      break;
  }

  res.status(200).send('OK');
});
```

### Event Types (SDK enum)

| Event | Trigger |
|-------|---------|
| `individual_client.created` / `.updated` | Individual client changes |
| `corporate_client.created` / `.updated` | Corporate client changes |
| `alert.created` / `.updated` | Alert lifecycle |
| `case.created` / `.updated` | Case lifecycle |
| `transaction.created` / `.updated` | Transaction changes |
| `deposit.created` / `.updated` | Deposit operations |
| `withdrawal.created` / `.updated` | Withdrawal operations |
| `trade.created` / `.updated` | Trade operations |
| `individual_member.created` / `.updated` | Member changes |
| `corporate_member.created` / `.updated` | Corporate member changes |
| `blockchain_wallet.created` / `.updated` | Wallet changes |
| `bank_account.created` / `.updated` | Bank account changes |
| `payment_account.created` / `.updated` | Payment account changes |
| `checklist.created` / `.updated` | Checklist changes |
| `attachment.created` / `.updated` / `.deleted` | Attachment lifecycle |
| `form_template.public_form_submitted` | Client submitted a form |

## Critical Rules

1. **Use `CorsaClient`** — `ComplianceClient` is a deprecated alias
2. **Use `RiskDto.level`** for risk enums — `ClientRiskDto` does not exist in SDK exports
3. **Auth format is `Bearer <TOKEN>:<SECRET>`** — not just the token alone
4. **Provide `referenceId`** — your internal system ID for the entity; you can use either `referenceId` or Corsa's generated `id` in subsequent API calls
5. **Use the `HEADERS` constructor option for auth** — `TOKEN` only sets `Bearer <value>` which doesn't support the `TOKEN:SECRET` compound format
6. **Operations `initiatedBy` expects a Corsa `id`** — use the `id` returned from client creation
7. **Rate limit is 500 req/60s** — implement retry with `Retry-After` header
8. **Webhook raw body** — use `express.raw()` not `express.json()` for signature verification

## Common Pitfalls

| Mistake | Fix |
|---------|-----|
| Omitting `referenceId` | Include it — it maps to your internal system ID and enables upsert via `upsert=true` |
| Using `Bearer <TOKEN>` without secret | Format is `Bearer <TOKEN>:<SECRET>` |
| Parsing webhook body as JSON before verification | Use `express.raw()` to get the raw Buffer, verify, then parse |
| Creating operations before clients | Follow the ingestion order — clients must exist first |
| Using deprecated `ComplianceClient` | Import and use `CorsaClient` |
| Ignoring 429 responses | Check `Retry-After` header and back off |
| Hardcoding US base URL for EU customers | Use `https://api.eu.corsa.finance` for EU region |

## Links

- [API Reference](https://api.corsa.finance/api-spec.json)
- [Documentation](https://docs.corsa.finance)
- [SDK on npm](https://www.npmjs.com/package/@corsa-labs/sdk)
- [Full method reference](references/REFERENCE.md)
