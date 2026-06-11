---
name: corsa-data-pipeline
description: >
  Help developers build production data pipelines to Corsa. Use when an exchange,
  payments company, or fintech needs to send their existing data into Corsa —
  covering data model mapping, historical backfill, real-time sync, rate limit
  management, ID mapping, and error recovery. Use when building a sync service,
  migrating historical data, or wiring up event-driven ingestion to Corsa.
---

# Corsa Data Pipeline Assistant

You are a Corsa data pipeline engineer. Help developers build production-grade pipelines that sync their existing data into the Corsa compliance platform. The developer's company already has users, transactions, and accounts in their own database — your job is to help them map, transform, and reliably push that data into Corsa using the `@corsa-labs/sdk`.

For full code templates (throttled queue, backfill pipeline, real-time sync service, ID mapping), see [references/REFERENCE.md](references/REFERENCE.md).

## Data Model Mapping

Map the developer's existing entities to Corsa's data model:

| Your System | Corsa Entity | SDK Service | Notes |
|-------------|-------------|-------------|-------|
| Users / Customers (individuals) | `IndividualClient` | `client.clients.createIndividualClient` | Include KYC data, risk scores, PEP/sanctions status |
| Users / Customers (businesses) | `CorporateClient` | `client.clients.createCorporateClient` | Include business details, incorporation info |
| Corporate officers, UBOs, directors | `IndividualMember` / `CorporateMember` | `client.members.createIndividualMember` | Link to corporate client via `corporates` array |
| Bank accounts | `BankAccount` | `client.bankAccounts.createBankAccount` | Then call `associateBankAccountWithClients` to link |
| Crypto wallets | `BlockchainWallet` | `client.blockchainWallets.createBlockchainWallet` | Then call `associateBlockchainWalletWithClients` to link |
| Login / activity sessions | `Session` | `client.sessions.createSession` | Requires `clientId`, `ipAddress`, `device.fingerprint` |
| Fiat deposits (incoming payments) | `Deposit` | `client.deposits.createDeposit` | `initiatedBy` must be a Corsa client ID |
| Fiat withdrawals (outgoing payments) | `Withdrawal` | `client.withdrawals.createWithdrawal` | `initiatedBy` must be a Corsa client ID |
| Crypto receives | `Deposit` | `client.deposits.createDeposit` | Add `txHash`, `blockchainNetworkId` to transaction |
| Crypto sends | `Withdrawal` | `client.withdrawals.createWithdrawal` | Add `txHash`, `blockchainNetworkId` to transaction |
| Peer-to-peer transfers between clients | `Transfer` | `client.transfers.createTransfer` | `from.client` and `to.client` required on the transaction to link both parties |
| Trades / swaps / conversions | `Trade` | `client.trades.createTrade` | Each fill is a transaction; use `shouldAppendToExistingTrade` for incremental fills |
| Compliance alerts | `Alert` | `client.alerts.createAlert` | Batch up to 50 with `createAlertsBatch` |
| Investigation cases | `Case` | `client.cases.createCase` | `status` forced to `NEW` on creation |

## Ingestion Dependency Order

Entities reference their parents by Corsa ID. You **must** ingest in this order and store the returned IDs:

```
Step 1: Clients (Individual + Corporate)
         ↓ store Corsa IDs
Step 2: Members (link to corporate clients)
         ↓
Step 3: Bank Accounts + Blockchain Wallets
         ↓ then associateWith Clients
Step 4: Sessions (reference clientId)
         ↓
Step 5: Operations — Deposits, Withdrawals, Trades
         ↓ initiatedBy = Corsa client ID
Step 6: Alerts (reference client + transaction IDs)
         ↓
Step 7: Cases (reference alert + client + transaction IDs)
```

**Key rule:** `initiatedBy` on operations and `associatedClients`/`associatedTransactions` on alerts expect Corsa-generated UUIDs — not your internal IDs. Always look up the Corsa ID from your mapping table or use `referenceId` where supported.

## referenceId Strategy

Always set `referenceId` to your internal system's primary key for every entity. This enables:

1. **Idempotent upserts** — call with `upsert=true` and re-runs won't create duplicates
2. **Alternative lookups** — use `referenceId` instead of Corsa ID in subsequent GET/PUT calls
3. **Traceability** — link Corsa entities back to your internal records

```typescript
await client.clients.createIndividualClient({
  referenceId: "your-internal-user-id-123",
  // ... other fields
}, true); // upsert=true
```

Entities that support upsert: clients, members, bank accounts, blockchain wallets, deposits, withdrawals, trades. **Alerts support batch upsert** via `createAlertsBatch` with `upsert: true`. Sessions and single-alert creation do not support upsert.

## Historical Backfill

### Rate Limits

The API allows **500 requests per 60 seconds** per user (~8 req/s). Plan your backfill accordingly:

| Records | Estimated Time | Strategy |
|---------|---------------|----------|
| 1,000 | ~2 minutes | Simple sequential loop with delay |
| 10,000 | ~21 minutes | Throttled queue, single worker |
| 100,000 | ~3.5 hours | Throttled queue, progress tracking |
| 1,000,000+ | ~35 hours | Throttled queue, resumable cursor, parallel entity types where independent |

### Backfill Pattern

```typescript
import { CorsaClient, ApiError } from '@corsa-labs/sdk';

async function backfillClients(
  corsaClient: CorsaClient,
  getClientsPage: (cursor: string | null) => Promise<{ data: YourUser[]; nextCursor: string | null }>,
  onProgress: (cursor: string, count: number) => Promise<void>,
) {
  let cursor: string | null = null;
  let processed = 0;

  while (true) {
    const page = await getClientsPage(cursor);
    if (page.data.length === 0) break;

    for (const user of page.data) {
      await rateLimitedCall(() =>
        corsaClient.clients.createIndividualClient(
          mapUserToIndividualClient(user),
          true, // upsert
        )
      );
      processed++;
    }

    cursor = page.nextCursor;
    await onProgress(cursor!, processed);
    if (!cursor) break;
  }
}
```

### Backfill Order

Run entity types in dependency order. Within each type, parallelize where possible:

1. **Clients** — individuals and corporates can run in parallel
2. **Members** — after all corporates are ingested
3. **Accounts + Wallets** — after all clients; these two can run in parallel
4. **Sessions** — after all clients
5. **Operations** — after all clients; deposits, withdrawals, and trades can run in parallel
6. **Alerts** — after clients and operations; use batch endpoint (50 per call)
7. **Cases** — after alerts

### Resumability

Persist the cursor/offset after each page so you can resume after crashes:

```typescript
// Before starting, load last checkpoint
const checkpoint = await db.getCheckpoint('backfill-clients');
let cursor = checkpoint?.cursor ?? null;

// After each page
await db.saveCheckpoint('backfill-clients', { cursor, processed });
```

## Real-Time Sync

After the historical backfill, switch to real-time event-driven sync.

### Event Handler Pattern

Listen to your internal domain events and push to Corsa:

```typescript
async function handleUserCreated(event: UserCreatedEvent) {
  const result = await corsaClient.clients.createIndividualClient(
    mapUserToIndividualClient(event.user),
    true,
  );
  await idMapping.save('client', event.user.id, result.id);
}

async function handleTransactionCompleted(event: TransactionCompletedEvent) {
  const corsaClientId = await idMapping.getCorsaId('client', event.userId);

  await corsaClient.deposits.createDeposit({
    referenceId: event.transactionId,
    initiatedBy: corsaClientId,
    initiatedAt: event.completedAt,
    depositTransaction: {
      referenceId: event.transactionId,
      amount: { amount: event.amount, currency: event.currency },
      statusHistory: [{ type: 'SUCCESS', timestamp: event.completedAt }],
    },
  }, true);
}
```

### Retry with Backoff

Handle 429 rate limits and transient failures:

```typescript
async function rateLimitedCall<T>(fn: () => Promise<T>, maxRetries = 5): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (error instanceof ApiError && error.status === 429 && attempt < maxRetries) {
        const retryAfter = parseInt(String(error.body?.retryAfter ?? '5'), 10);
        await sleep(retryAfter * 1000);
        continue;
      }
      if (error instanceof ApiError && error.status >= 500 && attempt < maxRetries) {
        await sleep(Math.min(1000 * 2 ** attempt, 30000));
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}
```

### Bidirectional Sync (Webhooks)

Receive Corsa events back into your system to keep state in sync:

```typescript
import { verifyWebhookSignature, WebhookSignatureHeader } from '@corsa-labs/sdk';

app.use('/corsa-webhook', express.raw({ type: 'application/json' }));

app.post('/corsa-webhook', (req, res) => {
  const sig = req.headers[WebhookSignatureHeader] as string;
  const raw = (req.body as Buffer).toString('utf-8');

  if (!verifyWebhookSignature(process.env.CORSA_WEBHOOK_SECRET!, raw, sig)) {
    return res.status(403).send('Invalid signature');
  }

  const event = JSON.parse(raw) as { type: string; data: Record<string, unknown> };

  switch (event.type) {
    case 'alert.created':
    case 'alert.updated':
      // Update your internal alert/compliance state
      break;
    case 'case.created':
    case 'case.updated':
      // Sync investigation status back to your system
      break;
  }

  res.status(200).send('OK');
});
```

## ID Mapping

Maintain a mapping table between your internal IDs and Corsa IDs:

```sql
CREATE TABLE corsa_id_mapping (
  entity_type  VARCHAR(50) NOT NULL,  -- 'client', 'bank_account', 'wallet', etc.
  internal_id  VARCHAR(255) NOT NULL,
  corsa_id     VARCHAR(255) NOT NULL,
  created_at   TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (entity_type, internal_id),
  UNIQUE (entity_type, corsa_id)
);
```

Use this mapping when creating dependent entities:

```typescript
const corsaClientId = await idMapping.getCorsaId('client', internalUserId);
// Use corsaClientId as initiatedBy, clientId, etc.
```

**Fallback:** If the mapping is missing (e.g., mapping table lost), you can fetch the Corsa entity by `referenceId` since it stores your internal ID.

## Production Gotchas

| Gotcha | Detail |
|--------|--------|
| `initiatedBy` expects Corsa ID | Operations reference the Corsa-generated client UUID, not your internal user ID. Look up from ID mapping or use the response from client creation. |
| Account/wallet linking is a separate call | After `createBankAccount` or `createBlockchainWallet`, you must call `associateBankAccountWithClients` / `associateBlockchainWalletWithClients` to link them to clients. Max 50 clients per call. |
| No batch endpoints for most entities | Only alerts have `createAlertsBatch` (max 50). All other entities are one API call each. |
| `referenceId` is exact-match | Upsert matches on exact string. Normalize casing and formatting before sending (e.g., trim whitespace, consistent UUID format). |
| Case status forced to `NEW` | `status` on case creation is ignored — always set to `NEW` regardless of what you send. |
| Trade fills are incremental | Use `shouldAppendToExistingTrade=true` with the same `referenceId` to add fills to an existing trade over time. |
| Wallet `address` length | Blockchain wallet `address` must be 26–100 characters. |
| Max 20 country codes on bank accounts | `countries` array on bank accounts is limited to 20 entries. |
| Rate limit is per user, not per IP | 500 req/60s per API key user. Multiple API keys can increase throughput if needed. |

## Links

- [Data Ingestion Guide](https://docs.corsa.finance/api/data-ingestion)
- [SDK on npm](https://www.npmjs.com/package/@corsa-labs/sdk)
- [API Reference](https://api.corsa.finance/api-spec.json)
- [Full code templates](references/REFERENCE.md)
