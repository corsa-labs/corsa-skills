# Corsa SDK — Full API Reference

Complete method signatures and code templates for all 22 services exposed by `CorsaClient` from `@corsa-labs/sdk`.

## Authentication Setup

### Constructor Config (recommended)

```typescript
import { CorsaClient } from '@corsa-labs/sdk';

const client = new CorsaClient({
  BASE: "https://api.corsa.finance",
  HEADERS: {
    Authorization: `Bearer ${process.env.API_TOKEN}:${process.env.API_SECRET}`
  }
});
```

### Global Config (alternative)

```typescript
import { CorsaClient, OpenAPI } from '@corsa-labs/sdk';

OpenAPI.BASE = 'https://api.corsa.finance';
OpenAPI.HEADERS = {
  Authorization: `Bearer ${process.env.API_TOKEN}:${process.env.API_SECRET}`
};

const client = new CorsaClient();
```

### Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `BASE` | `string` | `''` | API base URL |
| `HEADERS` | `Record<string, string>` | `undefined` | Custom headers (use for auth) |
| `TOKEN` | `string` | `undefined` | Bearer token (sets `Bearer <value>` — use HEADERS for Corsa's `TOKEN:SECRET` format) |
| `USERNAME` | `string` | `undefined` | Basic auth username |
| `PASSWORD` | `string` | `undefined` | Basic auth password |
| `WITH_CREDENTIALS` | `boolean` | `false` | Include credentials in requests |
| `CREDENTIALS` | `string` | `'include'` | Fetch credentials mode |
| `ENCODE_PATH` | `function` | `undefined` | Custom path encoder |

---

## Clients Service

Access: `client.clients`

### Methods

| Method | Description |
|--------|-------------|
| `createIndividualClient(requestBody, upsert?)` | Create or upsert an individual client |
| `updateIndividualClient(clientId, requestBody)` | Update an individual client |
| `getIndividualClient(clientId, integrationId?)` | Get an individual client by ID |
| `enableIndividualClientAutoModelRisk(clientId, requestBody)` | Enable automatic model-driven risk for an individual client |
| `bulkUpdateIndividualClients(requestBody)` | Update multiple individual clients with the same fields (max 100) |
| `createCorporateClient(requestBody, upsert?)` | Create or upsert a corporate client |
| `updateCorporateClient(clientId, requestBody)` | Update a corporate client |
| `getCorporateClient(clientId, includeMembers?, integrationId?)` | Get a corporate client by ID |
| `enableCorporateClientAutoModelRisk(clientId, requestBody)` | Enable automatic model-driven risk for a corporate client |
| `bulkUpdateCorporateClients(requestBody)` | Update multiple corporate clients with the same fields (max 100) |

### Create Individual Client

```typescript
import {
  CorsaClient,
  CreateIndividualClientDto,
  RiskDto,
  IndividualClientGeneralInformationDto,
} from '@corsa-labs/sdk';

const { id } = await client.clients.createIndividualClient(
  {
    referenceId: "USER-12345",
    activityStatus: CreateIndividualClientDto.activityStatus.ACTIVE,
    accountStatus: CreateIndividualClientDto.accountStatus.APPROVED,
    general: {
      firstName: "John",
      lastName: "Doe",
      dateOfBirth: "1985-06-15",
      citizenship: "USA",
      personalId: "123-45-6789",
      gender: IndividualClientGeneralInformationDto.gender.MALE,
    },
    address: {
      addressLine1: "123 Main St",
      city: "New York",
      country: "USA",
      postalCode: "10001",
    },
    contact: {
      emailAddress: "john.doe@example.com",
      phoneNumber: "+1-555-0199",
    },
    currentRisk: {
      score: 35,
      level: RiskDto.level.LOW,
      reason: "Standard retail customer",
      calculatedAt: "2024-01-15T10:00:00Z",
    },
    tags: ["retail"],
    sanctions: { isSanctioned: false },
    politicalExposure: { isPoliticallyExposed: false },
    adverseMedia: { isAdverseMedia: false },
  },
  true // upsert — update if an entity with this referenceId already exists
);
```

### Create Corporate Client

```typescript
const { id } = await client.clients.createCorporateClient(
  {
    referenceId: "CORP-98765",
    accountStatus: "APPROVED",
    activityStatus: "ACTIVE",
    general: {
      legalEntityName: "Acme Innovations LLC",
      dateOfIncorporation: "2018-04-12",
      countryOfIncorporation: "USA",
    },
    business: {
      industry: "Technology",
      description: "SaaS provider for financial services",
      businessType: "FINANCIAL_INSTITUTIONS",
      incorporationType: "LIMITED_LIABILITY_COMPANY",
    },
    address: {
      registrationAddress: {
        addressLine1: "456 Tech Park Blvd",
        city: "Austin",
        country: "USA",
        postalCode: "73301",
      },
    },
    tags: ["enterprise", "saas"],
  },
  true // upsert
);
```

### Update a Client

```typescript
await client.clients.updateIndividualClient("client-uuid-123", {
  contact: { emailAddress: "new.email@example.com" },
  activityStatus: "ACTIVE",
});
```

---

## Members Service

Access: `client.members`

### Methods

| Method | Description |
|--------|-------------|
| `createIndividualMember(requestBody, upsert?)` | Create an individual member (UBO/Director/Signatory) |
| `updateIndividualMember(memberId, requestBody)` | Update an individual member |
| `getIndividualMember(memberId)` | Get an individual member by ID |
| `addIndividualMemberDocument(memberId, requestBody)` | Add a document to a member |
| `getIndividualMemberDocuments(memberId)` | List member documents |
| `updateIndividualMemberDocument(memberId, documentId, requestBody)` | Update a member document |
| `deleteIndividualMemberDocument(memberId, documentId)` | Delete a member document |
| `createCorporateMember(requestBody, upsert?)` | Create a corporate member |
| `updateCorporateMember(memberId, requestBody)` | Update a corporate member |
| `getCorporateMember(memberId)` | Get a corporate member by ID |

---

## Deposits Service

Access: `client.deposits`

### Methods

| Method | Description |
|--------|-------------|
| `createDeposit(requestBody, upsert?)` | Create or upsert a deposit operation |
| `getDeposit(id)` | Get a deposit by ID or referenceId |

### Create Crypto Deposit

```typescript
const deposit = await client.deposits.createDeposit(
  {
    referenceId: "DEP-2024-001",
    initiatedBy: "client-uuid-123", // Corsa client ID
    initiatedAt: "2024-01-15T08:30:00Z",
    depositTransaction: {
      referenceId: "TX-BLOCK-888",
      txHash: "0x123abc...",
      amount: { amount: 1.5, currency: "BTC", netAmount: 1.5 },
      convertedAmount: { amount: 65000.0, currency: "USD" },
      from: { walletAddress: "0xSourceWallet..." },
      to: { walletAddress: "0xDepositAddress..." },
      blockchainNetworkId: "bitcoin-mainnet",
      statusHistory: [
        { type: "SUCCESS", timestamp: "2024-01-15T08:30:00Z" },
      ],
    },
  },
  true // upsert
);
```

---

## Withdrawals Service

Access: `client.withdrawals`

### Methods

| Method | Description |
|--------|-------------|
| `createWithdrawal(requestBody, upsert?)` | Create or upsert a withdrawal operation |
| `getWithdrawal(id)` | Get a withdrawal by ID or referenceId |

### Create Fiat Withdrawal

```typescript
const withdrawal = await client.withdrawals.createWithdrawal(
  {
    referenceId: "WDR-FIAT-001",
    initiatedBy: "client-uuid-123",
    initiatedAt: "2024-01-16T14:20:00Z",
    withdrawTransaction: {
      referenceId: "TX-BANK-999",
      amount: { amount: 5000, currency: "USD" },
      to: { bankAccountNumber: "1234567890" },
      paymentMethod: "WIRE_TRANSFER",
      statusHistory: [
        { type: "PENDING", timestamp: "2024-01-16T14:20:00Z" },
      ],
    },
  },
  true // upsert
);
```

### Create Crypto Withdrawal

```typescript
const withdrawal = await client.withdrawals.createWithdrawal(
  {
    referenceId: "WDR-CRYPTO-001",
    initiatedBy: "client-uuid-123",
    initiatedAt: "2024-01-16T14:20:00Z",
    withdrawTransaction: {
      referenceId: "TX-BLOCK-999",
      txHash: "0x456def...",
      amount: { amount: 5000, currency: "USDC" },
      to: { walletAddress: "0xDestWallet..." },
      blockchainNetworkId: "ethereum-mainnet",
      statusHistory: [
        { type: "PENDING", timestamp: "2024-01-16T14:20:00Z" },
      ],
    },
  },
  true // upsert
);
```

---

## Trades Service

Access: `client.trades`

### Methods

| Method | Description |
|--------|-------------|
| `createTrade(requestBody, shouldAppendToExistingTrade?, upsert?)` | Create a trade or append fills |
| `updateTradeStatus(id, requestBody)` | Update a trade's status |
| `getTrade(id)` | Get a trade by ID or referenceId |
| `addTransaction(id, requestBody)` | Add a transaction to an existing trade |

### Atomic Trade (all fills at once)

```typescript
const trade = await client.trades.createTrade({
  referenceId: "TRADE-100",
  initiatedBy: "client-uuid-123",
  initiatedAt: "2024-01-17T10:00:00Z",
  tradeType: "BUY",
  instrumentBaseAsset: "BTC",
  instrumentQuoteAsset: "USD",
  price: 60000,
  quantity: 1,
  status: "SUCCESS",
  transactions: [
    {
      referenceId: "TX-FILL-1",
      initiatedAt: "2024-01-17T10:00:00Z",
      amount: { amount: 0.4, currency: "BTC" },
      paymentMethod: "CRYPTO_TRANSFER",
      statusHistory: [{ type: "SUCCESS", timestamp: "2024-01-17T10:00:00Z" }],
    },
    {
      referenceId: "TX-FILL-2",
      initiatedAt: "2024-01-17T10:00:05Z",
      amount: { amount: 0.6, currency: "BTC" },
      paymentMethod: "CRYPTO_TRANSFER",
      statusHistory: [{ type: "SUCCESS", timestamp: "2024-01-17T10:00:05Z" }],
    },
  ],
});
```

### Incremental Trade (append fills)

Use `shouldAppendToExistingTrade=true` with the same `referenceId`:

```typescript
// First fill
await client.trades.createTrade(
  { referenceId: "TRADE-100", /* ... trade fields + first fill */ },
  true // shouldAppendToExistingTrade
);

// Second fill — appended to the same trade
await client.trades.createTrade(
  { referenceId: "TRADE-100", /* ... trade fields + second fill */ },
  true // shouldAppendToExistingTrade
);
```

---

## Transactions Service

Access: `client.transactions`

### Methods

| Method | Description |
|--------|-------------|
| `getTransactionById(id, integrationId?)` | Get a transaction by ID or referenceId |
| `updateTransaction(id, requestBody)` | Update a transaction |
| `updateTransactionStatus(id, requestBody)` | Update a transaction's status |
| `bulkUpdateTransactions(requestBody)` | Bulk update up to 100 transactions with the same fields |
| `lookupTransactionsByHash(requestBody)` | Batch lookup transactions by blockchain txHash (max 100) |

### Update Transaction Status

```typescript
await client.transactions.updateTransactionStatus("TX-FILL-1", {
  type: "SUCCESS",
  timestamp: "2024-01-17T10:05:00Z",
  reason: "Blockchain confirmation received",
  subStatus: "CONFIRMED_6_BLOCKS",
});
```

---

## Alerts Service

Access: `client.alerts`

### Methods

| Method | Description |
|--------|-------------|
| `createAlert(requestBody, failOnAssociation?)` | Create an alert |
| `createAlertsBatch(requestBody, failOnAssociation?)` | Batch create alerts (max 50) |
| `getAlert(alertId)` | Get an alert by ID |
| `getAlertsByEntity(entityType, entityId)` | Get all alerts linked to a specific entity (client, transaction, case) |
| `updateAlert(alertId, requestBody)` | Update an alert |
| `bulkUpdateAlert(requestBody)` | Bulk update alert fields across up to 100 alerts |
| `bulkUpdateAlertStatus(requestBody)` | Bulk update alert statuses (max 100) |
| `bulkAssignAlert(requestBody)` | Bulk assign alerts (max 100) |
| `bulkEscalateAlert(requestBody)` | Bulk escalate alerts (max 100) |
| `associateAlertWithTransactions(alertId, requestBody)` | Link alert to transactions |
| `associateAlertWithClients(alertId, requestBody)` | Link alert to clients |
| `addScreeningMatches(alertId, requestBody)` | Attach 1–100 screening matches to an existing screening alert |
| `updateScreeningMatch(alertId, matchId, requestBody)` | Update a pending screening match before a decision is recorded |
| `recordScreeningMatchDecision(alertId, matchId, requestBody)` | Record TRUE_MATCH / FALSE_MATCH / ESCALATED decision on a match |
| `recordBulkScreeningMatchDecision(alertId, requestBody)` | Apply the same decision to multiple matches (max 100) |
| `deleteScreeningMatch(alertId, matchId)` | Delete a pending screening match (recomputes client screening status) |

### Create Alert

```typescript
const alert = await client.alerts.createAlert({
  referenceId: "ALERT-001",
  description: "High-value transaction of $50,000 from a sanctioned country",
  category: "SCREENING_SANCTIONS",
  priority: "HIGH",
  status: "NEW",
  raisedAt: "2024-01-15T12:00:00Z",
  associatedClients: ["client-uuid-123"],
  associatedTransactions: ["tx-uuid-456"],
});
```

---

## Cases Service

Access: `client.cases`

### Methods

| Method | Description |
|--------|-------------|
| `createCase(requestBody)` | Create a case |
| `createCasesBatch(requestBody)` | Batch create multiple cases (max 50) |
| `getCase(caseId)` | Get a case by ID |
| `updateCase(caseId, requestBody)` | Update a case |
| `bulkUpdateCase(requestBody)` | Update multiple cases with the same fields (max 100) |
| `bulkUpdateCaseStatus(requestBody)` | Bulk update case statuses (max 100) |
| `bulkAssignCase(requestBody)` | Bulk assign cases (max 100) |
| `bulkUpdateCaseReviewers(requestBody)` | Bulk update case reviewers (max 100) |
| `associateCaseWithTransactions(caseId, requestBody)` | Link case to transactions |
| `associateCaseWithClients(caseId, requestBody)` | Link case to clients |
| `associateCaseWithAlerts(caseId, requestBody)` | Link case to alerts |

---

## Bank Accounts Service

Access: `client.bankAccounts`

### Methods

| Method | Description |
|--------|-------------|
| `createBankAccount(requestBody, upsert?)` | Create or upsert a bank account |
| `getBankAccount(bankAccountId)` | Get by ID or referenceId |
| `updateBankAccount(bankAccountId, requestBody)` | Update a bank account |
| `associateBankAccountWithClients(bankAccountId, requestBody)` | Link to clients |

### Create and Link Bank Account

```typescript
const account = await client.bankAccounts.createBankAccount(
  {
    referenceId: "BA-001",
    bankName: "First National Bank",
    accountNumber: "1234567890",
    routingNumber: "021000021",
    currency: "USD",
    country: "USA",
  },
  true // upsert
);

await client.bankAccounts.associateBankAccountWithClients(account.id, {
  clients: [{ clientId: "client-uuid-123" }],
});
```

---

## Blockchain Wallets Service

Access: `client.blockchainWallets`

### Methods

| Method | Description |
|--------|-------------|
| `createBlockchainWallet(requestBody, upsert?)` | Create or upsert a wallet |
| `getBlockchainWallet(blockchainWalletId)` | Get by ID, referenceId, or address |
| `updateBlockchainWallet(blockchainWalletId, requestBody)` | Update a wallet |
| `associateBlockchainWalletWithClients(blockchainWalletId, requestBody)` | Link to clients |

### Create and Link Wallet

```typescript
const wallet = await client.blockchainWallets.createBlockchainWallet(
  {
    referenceId: "WALLET-001",
    address: "0x1234...abcd",
    blockchainNetworkId: "ethereum-mainnet",
    currency: "ETH",
  },
  true // upsert
);

await client.blockchainWallets.associateBlockchainWalletWithClients(wallet.id, {
  associatedClients: [{ clientId: "client-uuid-123" }],
});
```

---

## Payment Accounts Service

Access: `client.paymentAccounts`

### Methods

| Method | Description |
|--------|-------------|
| `createPaymentAccount(requestBody, upsert?)` | Create or upsert a payment account |
| `getPaymentAccount(paymentAccountId)` | Get by ID or referenceId |
| `updatePaymentAccount(paymentAccountId, requestBody)` | Update a payment account |
| `associatePaymentAccountWithClients(paymentAccountId, requestBody)` | Link to clients |

### Create and Link Payment Account

```typescript
const account = await client.paymentAccounts.createPaymentAccount(
  {
    referenceId: "PIX-001",
    pixKey: "john.doe@example.com",
    country: "BRA",
    currency: "BRL",
  },
  true // upsert
);

await client.paymentAccounts.associatePaymentAccountWithClients(account.id, {
  clients: [{ clientId: "client-uuid-123" }],
});
```

---

## Sub-Dispositions Service

Access: `client.subDispositions`

### Methods

| Method | Description |
|--------|-------------|
| `listSubDispositions(parentStatus?, entityType?, isActive?)` | List sub-dispositions for the current platform, optionally filtered |
| `createSubDisposition(requestBody)` | Create a custom sub-disposition |
| `getSubDisposition(id)` | Get a sub-disposition by ID |
| `updateSubDisposition(id, requestBody)` | Update a sub-disposition |
| `deleteSubDisposition(id)` | Soft-delete a custom sub-disposition |

Sub-dispositions add granular resolution reasons to Corsa's standard statuses. Each sub-disposition qualifies a **parent status** (`RESOLVED`, `ESCALATED`, `CLOSED_DISMISSED`, or `CLOSED_ESCALATION_TO_SAR`) for either `ALERT` or `CASE` entities — e.g. a `RESOLVED` alert with sub-disposition `false_positive_rule_tuning`.

**`listSubDispositions` filter parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `parentStatus` | `string?` | Filter by parent status: `RESOLVED`, `ESCALATED`, `IN_REVIEW`, `CLOSED_DISMISSED`, or `CLOSED_ESCALATION_TO_SAR` |
| `entityType` | `string?` | Filter by entity: `ALERT` or `CASE` |
| `isActive` | `boolean?` | `true` for active only, `false` for inactive only, omit for all |

---

## External Rules Service

Access: `client.externalRules`

### Methods

| Method | Description |
|--------|-------------|
| `createExternalRule(requestBody)` | Create an external vendor rule |
| `listExternalRules(params?)` | List external rules with pagination |
| `getExternalRule(id)` | Get an external rule by ID |
| `updateExternalRule(id, requestBody)` | Update an external vendor rule |
| `deleteExternalRule(id)` | Soft-delete an external rule |
| `checkRuleNameExists(name)` | Check if a rule name is already taken |
| `getExternalRuleVendors()` | Get distinct vendor names across external rules |

External rules let you ingest alert signals from third-party vendors (Chainalysis, TRM Labs, etc.) and route them through Corsa's alert management workflow.

---

## Sessions Service

Access: `client.sessions`

### Methods

| Method | Description |
|--------|-------------|
| `createSession(requestBody)` | Create a session |
| `getSession(id)` | Get by ID or referenceId |
| `updateSession(id, requestBody)` | Update a session |
| `getClientSessions(clientId)` | List sessions for a client |

---

## Rules Service

Access: `client.rules`

### Methods

| Method | Description |
|--------|-------------|
| `createRule(requestBody)` | Create a rule (draft) |
| `listRules(page?, limit?, ...)` | List rules with filtering and pagination |
| `getRule(id, version?)` | Get a rule by ID |
| `updateRule(id, requestBody)` | Update a rule |
| `activateRule(id, requestBody?)` | Activate a rule |
| `disableRule(id, requestBody?)` | Disable a rule |
| `deleteRule(id, reason?)` | Soft delete a rule |

---

## Rule Templates Service

Access: `client.ruleTemplates`

### Methods

| Method | Description |
|--------|-------------|
| `listRuleTemplates(page?, limit?, ...)` | List rule templates with filtering |
| `getRuleTemplate(id)` | Get a rule template by ID |
| `copyRuleTemplate(id)` | Copy a template to your workspace as a draft rule |

---

## Evaluation Service

Access: `client.evaluation`

### Methods

| Method | Description |
|--------|-------------|
| `evaluate(requestBody)` | Evaluate rules against a transaction |
| `getTransactionEvaluations(transactionId, page?, pageSize?)` | Get evaluations for a transaction |
| `getRuleEvaluations(ruleId, page?, pageSize?)` | Get evaluations for a rule |

---

## Checklists Service

Access: `client.checklists`

### Methods

| Method | Description |
|--------|-------------|
| `getEntityChecklist(entityId)` | Get the newest active checklist for an entity |
| `updateChecklistItem(checklistId, itemId, requestBody)` | Update a checklist item |
| `createChecklistTemplate(requestBody)` | Create a checklist template |
| `getChecklistTemplatesByPlatform(platformId, entityType?)` | List checklist templates |
| `getChecklistTemplateById(id)` | Get a template by ID |
| `updateChecklistTemplate(id, requestBody)` | Update a template |
| `deleteChecklistTemplate(id)` | Delete a template |
| `addItemToTemplate(id, requestBody)` | Add an item to a template |
| `updateTemplateItem(itemId, requestBody)` | Update a template item |
| `deleteTemplateItem(itemId)` | Delete a template item |

---

## Attachments Service

Access: `client.attachments`

### Methods

| Method | Description |
|--------|-------------|
| `getAttachmentsByEntity(entityType, entityId)` | Get attachments for an entity |
| `uploadAttachments(entityType, entityId, formData)` | Upload files |
| `getDownloadUrlsByIds(ids)` | Get download URLs |
| `updateAttachment(attachmentId, requestBody)` | Update attachment metadata |
| `deleteAttachment(attachmentId)` | Delete an attachment |
| `relateAttachments(requestBody)` | Relate attachments to an entity |
| `createExternalDocument(requestBody)` | Create an external document attachment |

---

## Verifications Service

Access: `client.verifications`

### Methods

| Method | Description |
|--------|-------------|
| `createVerification(clientId, requestBody)` | Create a KYC/KYB verification for a client |
| `updateVerification(clientId, verificationId, requestBody)` | Update an existing verification's status or details |
| `getVerification(clientId, provider, providerId)` | Look up a verification by provider and provider ID |

Verifications record KYC/KYB results from identity providers (SumSub, Persona, etc.) against a client. Use `createVerification` to ingest results after onboarding, and `getVerification` to look up existing records by the provider's own applicant ID.

---

## Platform Service

Access: `client.platform`

### Methods

| Method | Description |
|--------|-------------|
| `getEncryptionConfiguration()` | Get the platform's configuration |

---

## Status Enums

### Operation Statuses (Deposits, Withdrawals, Trades)

| Status | Description |
|--------|-------------|
| `PENDING` | Operation initiated, not yet final |
| `SUCCESS` | Completed successfully |
| `FAILED` | Operation failed |
| `REJECTED` | Operation was rejected |
| `EXPIRED` | Time-bound operation expired |

### Transaction Statuses

| Status | Description |
|--------|-------------|
| `PENDING` | Transaction is processing |
| `SUCCESS` | Transaction confirmed |
| `FAILED` | Transaction failed |
| `CANCELLED` | Transaction was cancelled |
| `FROZEN` | Transaction has been frozen |

---

## Webhook Event Catalog

### Event Types

All events follow the pattern `<entity>.<action>` where action is `created` or `updated`.

| Event Type | Description |
|------------|-------------|
| `individual_client.created` | Individual client created |
| `individual_client.updated` | Individual client updated |
| `corporate_client.created` | Corporate client created |
| `corporate_client.updated` | Corporate client updated |
| `alert.created` | Alert created |
| `alert.updated` | Alert updated (status, assignment, priority) |
| `case.created` | Case created |
| `case.updated` | Case updated (status, assignment, investigation) |
| `transaction.created` | Transaction created |
| `transaction.updated` | Transaction updated (status change) |
| `deposit.created` | Deposit operation created |
| `deposit.updated` | Deposit operation updated |
| `withdrawal.created` | Withdrawal operation created |
| `withdrawal.updated` | Withdrawal operation updated |
| `trade.created` | Trade operation created |
| `trade.updated` | Trade operation updated |
| `individual_member.created` | Individual member created |
| `individual_member.updated` | Individual member updated |
| `corporate_member.created` | Corporate member created |
| `corporate_member.updated` | Corporate member updated |
| `blockchain_wallet.created` | Blockchain wallet created |
| `blockchain_wallet.updated` | Blockchain wallet updated |
| `bank_account.created` | Bank account created |
| `bank_account.updated` | Bank account updated |
| `payment_account.created` | Payment account (PIX, CLABE, mobile money) created |
| `payment_account.updated` | Payment account updated |
| `checklist.created` | Checklist created for an entity |
| `checklist.updated` | Checklist or checklist item updated |
| `attachment.created` | File attachment uploaded or linked to an entity |
| `attachment.updated` | Attachment metadata updated |
| `attachment.deleted` | Attachment deleted |
| `form_template.public_form_submitted` | Client submitted a public form |

### Webhook Payload Structure

```typescript
// SDK generic types (two type parameters):
interface WebhookEvent<M, T extends EntityCreatedPayload<M> | EntityUpdatedPayload<M>> {
  data: T;
  type: WebhookEventType;
  timestamp: string;
}

interface EntityCreatedPayload<T> {
  id: string;
  referenceId?: string;
  entity: T;
}

interface EntityUpdatedPayload<T> {
  id: string;
  referenceId?: string;
  updated: Partial<T>;
}

// For a generic handler, parse without SDK generics:
const event = JSON.parse(rawBody) as { type: string; timestamp: string; data: Record<string, unknown> };
```

### Webhook Headers

| Header | Description |
|--------|-------------|
| `x-hub-signature-256` | HMAC SHA256 signature for verification |
| `x-hook-id` | Unique ID of the webhook configuration |
| `x-hook-delivery` | Unique ID for this delivery attempt |
| `x-hook-event` | Event type that triggered the webhook |
| `x-request-id` | Request trace ID |
| `x-request-origin` | Origin of the request |

### Signature Verification

The signature uses HMAC SHA256 with your webhook secret:

```
sha256=<hex digest of HMAC-SHA256(secret, rawBody)>
```

Use `verifyWebhookSignature(secret, rawBody, signatureHeader)` from the SDK for timing-safe comparison.

### Express Handler Template

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

  if (!signature || !process.env.WEBHOOK_SECRET) {
    return res.status(400).send('Missing signature or secret');
  }

  if (!verifyWebhookSignature(process.env.WEBHOOK_SECRET, rawBody, signature)) {
    return res.status(403).send('Invalid signature');
  }

  const event = JSON.parse(rawBody) as { type: string; timestamp: string; data: Record<string, unknown> };

  switch (event.type) {
    case WebhookEventType.INDIVIDUAL_CLIENT_CREATED:
      // Handle individual client creation
      break;
    case WebhookEventType.ALERT_CREATED:
      // Handle new alert
      break;
    case WebhookEventType.CASE_UPDATED:
      // Handle case status change
      break;
    default:
      console.log(`Unhandled event: ${event.type}`);
  }

  res.status(200).send('OK');
});
```

---

## Error Handling

### ApiError

All SDK methods throw `ApiError` on non-2xx responses:

```typescript
import { ApiError } from '@corsa-labs/sdk';

try {
  const result = await client.clients.getIndividualClient("some-id");
} catch (error) {
  if (error instanceof ApiError) {
    console.error(`HTTP ${error.status}: ${error.statusText}`);
    console.error(`Response body:`, error.body);
    console.error(`Request ID:`, error.requestId);
  }
}
```

### Rate Limit Retry Pattern

```typescript
async function withRetry<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (error instanceof ApiError && error.status === 429 && attempt < maxRetries) {
        const retryAfter = parseInt(error.body?.retryAfter ?? '5', 10);
        await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}

// Usage
const result = await withRetry(() =>
  client.clients.createIndividualClient(payload, true)
);
```

### CancelablePromise

All SDK methods return `CancelablePromise`, which supports cancellation:

```typescript
const request = client.clients.getIndividualClient("client-123");

// Cancel if needed (e.g., component unmount, timeout)
request.cancel();
```

---

## REST API Endpoints (for non-SDK usage)

For developers using the REST API directly (curl, fetch, etc.), all endpoints are under `/v1/`:

| Entity | Create | Get | Update |
|--------|--------|-----|--------|
| Individual Clients | `POST /v1/clients/individuals?upsert=true` | `GET /v1/clients/individuals/{id}` | `PUT /v1/clients/individuals/{id}` |
| Corporate Clients | `POST /v1/clients/corporates?upsert=true` | `GET /v1/clients/corporates/{id}` | `PUT /v1/clients/corporates/{id}` |
| Individual Members | `POST /v1/members/individuals?upsert=true` | `GET /v1/members/individuals/{id}` | `PUT /v1/members/individuals/{id}` |
| Corporate Members | `POST /v1/members/corporates?upsert=true` | `GET /v1/members/corporates/{id}` | `PUT /v1/members/corporates/{id}` |
| Deposits | `POST /v1/operations/deposits?upsert=true` | `GET /v1/operations/deposits/{id}` | — |
| Withdrawals | `POST /v1/operations/withdrawals?upsert=true` | `GET /v1/operations/withdrawals/{id}` | — |
| Trades | `POST /v1/operations/trades` | `GET /v1/operations/trades/{id}` | `PUT /v1/operations/trades/{id}/updateStatus` |
| Transfers | `POST /v1/operations/transfers?upsert=true` | `GET /v1/operations/transfers/{id}` | — |
| Transactions | — | `GET /v1/transactions/{id}` | `PUT /v1/transactions/{id}`, `PUT /v1/transactions/{id}/updateStatus` |
| Alerts | `POST /v1/alerts` | `GET /v1/alerts/{alertId}` | `PUT /v1/alerts/{alertId}/update` |
| Cases | `POST /v1/cases` | `GET /v1/cases/{caseId}` | `PUT /v1/cases/{caseId}/update` |
| Bank Accounts | `POST /v1/bank-accounts?upsert=true` | `GET /v1/bank-accounts/{id}` | `PUT /v1/bank-accounts/{id}` |
| Blockchain Wallets | `POST /v1/blockchain-wallets?upsert=true` | `GET /v1/blockchain-wallets/{id}` | `PUT /v1/blockchain-wallets/{id}` |
| Payment Accounts | `POST /v1/payment-accounts?upsert=true` | `GET /v1/payment-accounts/{id}` | `PUT /v1/payment-accounts/{id}` |
| Sessions | `POST /v1/sessions` | `GET /v1/sessions/{id}` | `PUT /v1/sessions/{id}` |
| Verifications | `POST /v1/clients/{clientId}/verifications` | `GET /v1/clients/{clientId}/verifications/lookup` | `PUT /v1/clients/{clientId}/verifications/{verificationId}` |

Full endpoint documentation: [Corsa API Docs](https://docs.corsa.finance/api/data-ingestion)
