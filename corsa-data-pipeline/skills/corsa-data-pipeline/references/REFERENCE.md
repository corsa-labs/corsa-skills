# Corsa Data Pipeline — Code Templates

Production-ready code templates for building data pipelines to Corsa. Adapt these to your stack.

## Throttled Ingestion Queue

Rate-limited queue that respects 500 req/60s. Uses a token bucket approach.

```typescript
import { ApiError } from '@corsa-labs/sdk';

interface ThrottleConfig {
  maxRequestsPerWindow: number; // default 500
  windowMs: number;             // default 60000
  retryLimit: number;           // default 5
}

class ThrottledIngestionQueue {
  private tokens: number;
  private lastRefill: number;
  private readonly config: ThrottleConfig;

  constructor(config?: Partial<ThrottleConfig>) {
    this.config = {
      maxRequestsPerWindow: config?.maxRequestsPerWindow ?? 450, // leave headroom
      windowMs: config?.windowMs ?? 60_000,
      retryLimit: config?.retryLimit ?? 5,
    };
    this.tokens = this.config.maxRequestsPerWindow;
    this.lastRefill = Date.now();
  }

  private refillTokens() {
    const now = Date.now();
    const elapsed = now - this.lastRefill;
    const refill = Math.floor(
      (elapsed / this.config.windowMs) * this.config.maxRequestsPerWindow,
    );
    if (refill > 0) {
      this.tokens = Math.min(
        this.config.maxRequestsPerWindow,
        this.tokens + refill,
      );
      this.lastRefill = now;
    }
  }

  private async waitForToken(): Promise<void> {
    this.refillTokens();
    if (this.tokens > 0) {
      this.tokens--;
      return;
    }
    const waitMs = Math.ceil(
      this.config.windowMs / this.config.maxRequestsPerWindow,
    );
    await sleep(waitMs);
    return this.waitForToken();
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    await this.waitForToken();

    for (let attempt = 0; attempt <= this.config.retryLimit; attempt++) {
      try {
        return await fn();
      } catch (error) {
        if (error instanceof ApiError && error.status === 429 && attempt < this.config.retryLimit) {
          const retryAfter = parseInt(String(error.body?.retryAfter ?? '5'), 10);
          this.tokens = 0;
          await sleep(retryAfter * 1000);
          continue;
        }
        if (error instanceof ApiError && error.status >= 500 && attempt < this.config.retryLimit) {
          await sleep(Math.min(1000 * 2 ** attempt, 30_000));
          continue;
        }
        throw error;
      }
    }
    throw new Error('Max retries exceeded');
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

Usage:

```typescript
const queue = new ThrottledIngestionQueue({ maxRequestsPerWindow: 450 });

const result = await queue.execute(() =>
  corsaClient.clients.createIndividualClient(payload, true)
);
```

---

## ID Mapping Repository

Maintains the mapping between your internal IDs and Corsa IDs.

### SQL Schema

```sql
CREATE TABLE corsa_id_mapping (
  entity_type  VARCHAR(50) NOT NULL,
  internal_id  VARCHAR(255) NOT NULL,
  corsa_id     VARCHAR(255) NOT NULL,
  synced_at    TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (entity_type, internal_id),
  UNIQUE (entity_type, corsa_id)
);

CREATE INDEX idx_corsa_id_mapping_corsa ON corsa_id_mapping (entity_type, corsa_id);
```

### Repository Class

```typescript
type EntityType =
  | 'individual_client'
  | 'corporate_client'
  | 'individual_member'
  | 'corporate_member'
  | 'bank_account'
  | 'blockchain_wallet'
  | 'session'
  | 'deposit'
  | 'withdrawal'
  | 'trade'
  | 'alert'
  | 'case';

interface IdMappingRepo {
  save(entityType: EntityType, internalId: string, corsaId: string): Promise<void>;
  getCorsaId(entityType: EntityType, internalId: string): Promise<string>;
  getInternalId(entityType: EntityType, corsaId: string): Promise<string>;
  exists(entityType: EntityType, internalId: string): Promise<boolean>;
}

class IdMappingRepository implements IdMappingRepo {
  constructor(private readonly db: DatabaseConnection) {}

  async save(entityType: EntityType, internalId: string, corsaId: string): Promise<void> {
    await this.db.query(
      `INSERT INTO corsa_id_mapping (entity_type, internal_id, corsa_id)
       VALUES ($1, $2, $3)
       ON CONFLICT (entity_type, internal_id)
       DO UPDATE SET corsa_id = $3, synced_at = NOW()`,
      [entityType, internalId, corsaId],
    );
  }

  async getCorsaId(entityType: EntityType, internalId: string): Promise<string> {
    const row = await this.db.queryOne(
      `SELECT corsa_id FROM corsa_id_mapping WHERE entity_type = $1 AND internal_id = $2`,
      [entityType, internalId],
    );
    if (!row) throw new Error(`No Corsa ID mapping for ${entityType}:${internalId}`);
    return row.corsa_id;
  }

  async getInternalId(entityType: EntityType, corsaId: string): Promise<string> {
    const row = await this.db.queryOne(
      `SELECT internal_id FROM corsa_id_mapping WHERE entity_type = $1 AND corsa_id = $2`,
      [entityType, corsaId],
    );
    if (!row) throw new Error(`No internal ID mapping for ${entityType}:${corsaId}`);
    return row.internal_id;
  }

  async exists(entityType: EntityType, internalId: string): Promise<boolean> {
    const row = await this.db.queryOne(
      `SELECT 1 FROM corsa_id_mapping WHERE entity_type = $1 AND internal_id = $2`,
      [entityType, internalId],
    );
    return !!row;
  }
}
```

---

## Entity Mapping Examples

### Individual Client (from a typical exchange user)

```typescript
import {
  CorsaClient,
  CreateIndividualClientDto,
  RiskDto,
} from '@corsa-labs/sdk';

function mapUserToIndividualClient(user: YourUserType): CreateIndividualClientDto {
  return {
    referenceId: user.id,
    accountStatus: user.kycApproved ? 'APPROVED' : 'PENDING',
    activityStatus: user.isActive ? 'ACTIVE' : 'INACTIVE',
    general: {
      firstName: user.firstName,
      lastName: user.lastName,
      dateOfBirth: user.dateOfBirth,   // ISO format: "1990-01-15"
      citizenship: user.nationality,    // ISO 3166-1 alpha-3: "USA"
      personalId: user.governmentId,
    },
    address: {
      addressLine1: user.address.line1,
      addressLine2: user.address.line2,
      city: user.address.city,
      country: user.address.country,
      postalCode: user.address.postalCode,
    },
    contact: {
      emailAddress: user.email,
      phoneNumber: user.phone,
    },
    currentRisk: user.riskScore ? {
      score: user.riskScore,
      level: mapRiskLevel(user.riskLevel),
      reason: user.riskReason ?? 'Automated scoring',
      calculatedAt: user.riskCalculatedAt ?? new Date().toISOString(),
    } : undefined,
    tags: user.tags,
  };
}

function mapRiskLevel(level: string): RiskDto.level {
  switch (level?.toUpperCase()) {
    case 'HIGH': return RiskDto.level.HIGH;
    case 'MEDIUM': return RiskDto.level.MEDIUM;
    default: return RiskDto.level.LOW;
  }
}
```

### Corporate Client (from a business account)

```typescript
import { CreateCorporateClientDto } from '@corsa-labs/sdk';

function mapBusinessToCorporateClient(biz: YourBusinessType): CreateCorporateClientDto {
  return {
    referenceId: biz.id,
    accountStatus: biz.verified ? 'APPROVED' : 'PENDING',
    activityStatus: biz.isActive ? 'ACTIVE' : 'INACTIVE',
    general: {
      legalEntityName: biz.legalName,
      dateOfIncorporation: biz.incorporatedAt,
      countryOfIncorporation: biz.country,
    },
    business: {
      industry: biz.industry,
      description: biz.description,
      businessType: biz.type,            // e.g., "FINANCIAL_INSTITUTIONS"
      incorporationType: biz.entityType, // e.g., "LIMITED_LIABILITY_COMPANY"
    },
    address: {
      registrationAddress: {
        addressLine1: biz.address.line1,
        city: biz.address.city,
        country: biz.address.country,
        postalCode: biz.address.postalCode,
      },
    },
    tags: biz.tags,
  };
}
```

### Deposit (from a fiat payment)

```typescript
import { CreateDepositOperationDto } from '@corsa-labs/sdk';

function mapPaymentToDeposit(
  payment: YourPaymentType,
  corsaClientId: string,
): CreateDepositOperationDto {
  return {
    referenceId: payment.id,
    initiatedBy: corsaClientId,
    initiatedAt: payment.createdAt,
    depositTransaction: {
      referenceId: payment.transactionId,
      amount: {
        amount: payment.amount,
        currency: payment.currency,
        netAmount: payment.netAmount ?? payment.amount,
      },
      from: payment.senderBankAccount
        ? { bankAccountNumber: payment.senderBankAccount }
        : undefined,
      to: payment.receiverBankAccount
        ? { bankAccountNumber: payment.receiverBankAccount }
        : undefined,
      paymentMethod: payment.method, // "WIRE_TRANSFER", "ACH", etc.
      statusHistory: [
        { type: mapPaymentStatus(payment.status), timestamp: payment.updatedAt },
      ],
    },
  };
}
```

### Deposit (from a crypto receive)

```typescript
function mapCryptoReceiveToDeposit(
  tx: YourCryptoTxType,
  corsaClientId: string,
): CreateDepositOperationDto {
  return {
    referenceId: tx.id,
    initiatedBy: corsaClientId,
    initiatedAt: tx.timestamp,
    depositTransaction: {
      referenceId: tx.hash,
      txHash: tx.hash,
      amount: {
        amount: tx.amount,
        currency: tx.asset,
        netAmount: tx.amount,
      },
      from: { walletAddress: tx.fromAddress },
      to: { walletAddress: tx.toAddress },
      blockchainNetworkId: tx.network, // "ethereum-mainnet", "bitcoin-mainnet"
      statusHistory: [
        { type: 'SUCCESS', timestamp: tx.confirmedAt },
      ],
    },
  };
}
```

### Withdrawal (from a fiat payout)

```typescript
import { CreateWithdrawalOperationDto } from '@corsa-labs/sdk';

function mapPayoutToWithdrawal(
  payout: YourPayoutType,
  corsaClientId: string,
): CreateWithdrawalOperationDto {
  return {
    referenceId: payout.id,
    initiatedBy: corsaClientId,
    initiatedAt: payout.createdAt,
    withdrawTransaction: {
      referenceId: payout.transactionId,
      amount: {
        amount: payout.amount,
        currency: payout.currency,
      },
      to: { bankAccountNumber: payout.destinationAccount },
      paymentMethod: payout.method,
      statusHistory: [
        { type: mapPaymentStatus(payout.status), timestamp: payout.updatedAt },
      ],
    },
  };
}
```

### Trade (from an exchange order)

```typescript
import { CreateTradeOperationDto } from '@corsa-labs/sdk';

function mapOrderToTrade(
  order: YourOrderType,
  corsaClientId: string,
): CreateTradeOperationDto {
  return {
    referenceId: order.id,
    initiatedBy: corsaClientId,
    initiatedAt: order.createdAt,
    tradeType: order.side, // "BUY" or "SELL"
    instrumentBaseAsset: order.baseAsset,
    instrumentQuoteAsset: order.quoteAsset,
    price: order.avgPrice,
    quantity: order.filledQuantity,
    status: mapOrderStatus(order.status),
    transactions: order.fills.map((fill) => ({
      referenceId: fill.id,
      initiatedAt: fill.executedAt,
      amount: {
        amount: fill.quantity,
        currency: order.baseAsset,
      },
      paymentMethod: 'CRYPTO_TRANSFER',
      statusHistory: [
        { type: 'SUCCESS', timestamp: fill.executedAt },
      ],
    })),
  };
}
```

### Bank Account (with client association)

```typescript
import { CreateBankAccountDto } from '@corsa-labs/sdk';

async function syncBankAccount(
  account: YourBankAccountType,
  corsaClientId: string,
  corsaClient: CorsaClient,
  queue: ThrottledIngestionQueue,
  idMapping: IdMappingRepo,
) {
  const result = await queue.execute(() =>
    corsaClient.bankAccounts.createBankAccount(
      {
        referenceId: account.id,
        bankName: account.bankName,
        accountNumber: account.accountNumber,
        routingNumber: account.routingNumber,
        currency: account.currency,
        country: account.country,
      },
      true, // upsert
    ),
  );

  await idMapping.save('bank_account', account.id, result.id);

  await queue.execute(() =>
    corsaClient.bankAccounts.associateBankAccountWithClients(result.id, {
      clients: [{ clientId: corsaClientId }],
    }),
  );
}
```

### Blockchain Wallet (with client association)

```typescript
async function syncWallet(
  wallet: YourWalletType,
  corsaClientId: string,
  corsaClient: CorsaClient,
  queue: ThrottledIngestionQueue,
  idMapping: IdMappingRepo,
) {
  const result = await queue.execute(() =>
    corsaClient.blockchainWallets.createBlockchainWallet(
      {
        referenceId: wallet.id,
        address: wallet.address,
        chain: wallet.chain,
      },
      true, // upsert
    ),
  );

  await idMapping.save('blockchain_wallet', wallet.id, result.id);

  await queue.execute(() =>
    corsaClient.blockchainWallets.associateBlockchainWalletWithClients(result.id, {
      associatedClients: [{ clientId: corsaClientId }],
    }),
  );
}
```

---

## Full Backfill Pipeline

End-to-end pipeline that syncs all entity types in dependency order.

```typescript
import { CorsaClient } from '@corsa-labs/sdk';

interface BackfillProgress {
  entityType: string;
  cursor: string | null;
  processed: number;
  status: 'pending' | 'running' | 'completed' | 'failed';
  error?: string;
}

class CorsaBackfillPipeline {
  constructor(
    private readonly corsaClient: CorsaClient,
    private readonly queue: ThrottledIngestionQueue,
    private readonly idMapping: IdMappingRepo,
    private readonly progressStore: ProgressStore,
    private readonly dataSource: YourDataSource,
  ) {}

  async run() {
    const steps: Array<{ name: string; fn: () => Promise<void> }> = [
      { name: 'individual_clients', fn: () => this.backfillIndividualClients() },
      { name: 'corporate_clients', fn: () => this.backfillCorporateClients() },
      { name: 'members', fn: () => this.backfillMembers() },
      { name: 'bank_accounts', fn: () => this.backfillBankAccounts() },
      { name: 'blockchain_wallets', fn: () => this.backfillWallets() },
      { name: 'deposits', fn: () => this.backfillDeposits() },
      { name: 'withdrawals', fn: () => this.backfillWithdrawals() },
      { name: 'trades', fn: () => this.backfillTrades() },
      { name: 'alerts', fn: () => this.backfillAlerts() },
    ];

    for (const step of steps) {
      const progress = await this.progressStore.get(step.name);
      if (progress?.status === 'completed') {
        console.log(`Skipping ${step.name} (already completed)`);
        continue;
      }

      console.log(`Starting ${step.name}...`);
      await this.progressStore.update(step.name, { status: 'running' });

      try {
        await step.fn();
        await this.progressStore.update(step.name, { status: 'completed' });
        console.log(`Completed ${step.name}`);
      } catch (error) {
        const message = error instanceof Error ? error.message : String(error);
        await this.progressStore.update(step.name, { status: 'failed', error: message });
        throw new Error(`Backfill failed at ${step.name}: ${message}`);
      }
    }
  }

  private async backfillIndividualClients() {
    await this.backfillEntity(
      'individual_clients',
      (cursor) => this.dataSource.getIndividualUsersPage(cursor),
      async (user) => {
        const result = await this.queue.execute(() =>
          this.corsaClient.clients.createIndividualClient(
            mapUserToIndividualClient(user),
            true,
          ),
        );
        await this.idMapping.save('individual_client', user.id, result.id);
      },
    );
  }

  private async backfillCorporateClients() {
    await this.backfillEntity(
      'corporate_clients',
      (cursor) => this.dataSource.getCorporateUsersPage(cursor),
      async (biz) => {
        const result = await this.queue.execute(() =>
          this.corsaClient.clients.createCorporateClient(
            mapBusinessToCorporateClient(biz),
            true,
          ),
        );
        await this.idMapping.save('corporate_client', biz.id, result.id);
      },
    );
  }

  private async backfillBankAccounts() {
    await this.backfillEntity(
      'bank_accounts',
      (cursor) => this.dataSource.getBankAccountsPage(cursor),
      async (account) => {
        const corsaClientId = await this.idMapping.getCorsaId(
          account.ownerType === 'corporate' ? 'corporate_client' : 'individual_client',
          account.ownerId,
        );
        await syncBankAccount(account, corsaClientId, this.corsaClient, this.queue, this.idMapping);
      },
    );
  }

  private async backfillDeposits() {
    await this.backfillEntity(
      'deposits',
      (cursor) => this.dataSource.getDepositsPage(cursor),
      async (payment) => {
        const corsaClientId = await this.idMapping.getCorsaId('individual_client', payment.userId);
        await this.queue.execute(() =>
          this.corsaClient.deposits.createDeposit(
            mapPaymentToDeposit(payment, corsaClientId),
            true,
          ),
        );
      },
    );
  }

  private async backfillAlerts() {
    const BATCH_SIZE = 50;
    let cursor: string | null = (await this.progressStore.get('alerts'))?.cursor ?? null;
    let processed = (await this.progressStore.get('alerts'))?.processed ?? 0;

    while (true) {
      const page = await this.dataSource.getAlertsPage(cursor);
      if (page.data.length === 0) break;

      for (let i = 0; i < page.data.length; i += BATCH_SIZE) {
        const batch = page.data.slice(i, i + BATCH_SIZE);
        const alerts = await Promise.all(
          batch.map(async (alert) => {
            const corsaClientIds = await Promise.all(
              alert.clientIds.map((id: string) =>
                this.idMapping.getCorsaId('individual_client', id),
              ),
            );
            return {
              referenceId: alert.id,
              description: alert.description,
              category: alert.category,  // CreateAlertDto.category enum
              priority: alert.priority,  // CreateAlertDto.priority enum
              status: alert.status,      // CreateAlertDto.status enum
              raisedAt: alert.createdAt,
              associatedClients: corsaClientIds,
            };
          }),
        );

        await this.queue.execute(() =>
          this.corsaClient.alerts.createAlertsBatch({ alerts, upsert: true }),
        );
        processed += batch.length;
      }

      cursor = page.nextCursor;
      await this.progressStore.update('alerts', { cursor, processed });
      if (!cursor) break;
    }
  }

  private async backfillEntity<T>(
    name: string,
    getPage: (cursor: string | null) => Promise<{ data: T[]; nextCursor: string | null }>,
    processItem: (item: T) => Promise<void>,
  ) {
    const progress = await this.progressStore.get(name);
    let cursor = progress?.cursor ?? null;
    let processed = progress?.processed ?? 0;

    while (true) {
      const page = await getPage(cursor);
      if (page.data.length === 0) break;

      for (const item of page.data) {
        await processItem(item);
        processed++;
      }

      cursor = page.nextCursor;
      await this.progressStore.update(name, { cursor, processed });
      if (!cursor) break;
    }
  }
}
```

---

## Real-Time Sync Service

Event-driven sync that processes domain events and pushes to Corsa.

```typescript
interface SyncEvent {
  type: string;
  payload: Record<string, unknown>;
  timestamp: string;
}

class CorsaRealtimeSyncService {
  constructor(
    private readonly corsaClient: CorsaClient,
    private readonly queue: ThrottledIngestionQueue,
    private readonly idMapping: IdMappingRepo,
  ) {}

  async handleEvent(event: SyncEvent): Promise<void> {
    switch (event.type) {
      case 'user.created':
      case 'user.updated':
        return this.syncClient(event);
      case 'bank_account.created':
        return this.syncBankAccount(event);
      case 'wallet.created':
        return this.syncWallet(event);
      case 'deposit.completed':
        return this.syncDeposit(event);
      case 'withdrawal.completed':
        return this.syncWithdrawal(event);
      case 'trade.executed':
        return this.syncTrade(event);
      default:
        console.log(`Unhandled event type: ${event.type}`);
    }
  }

  private async syncClient(event: SyncEvent) {
    const user = event.payload as YourUserType;
    const result = await this.queue.execute(() =>
      this.corsaClient.clients.createIndividualClient(
        mapUserToIndividualClient(user),
        true,
      ),
    );
    await this.idMapping.save('individual_client', user.id, result.id);
  }

  private async syncDeposit(event: SyncEvent) {
    const payment = event.payload as YourPaymentType;
    const corsaClientId = await this.idMapping.getCorsaId('individual_client', payment.userId);
    await this.queue.execute(() =>
      this.corsaClient.deposits.createDeposit(
        mapPaymentToDeposit(payment, corsaClientId),
        true,
      ),
    );
  }

  // ... similar handlers for withdrawals, trades, bank accounts, wallets
}
```

---

## Status Mapping Helpers

Map your internal statuses to Corsa-compatible values.

```typescript
function mapPaymentStatus(status: string): string {
  const mapping: Record<string, string> = {
    completed: 'SUCCESS',
    settled: 'SUCCESS',
    confirmed: 'SUCCESS',
    pending: 'PENDING',
    processing: 'PENDING',
    failed: 'FAILED',
    rejected: 'REJECTED',
    cancelled: 'CANCELLED',
    expired: 'EXPIRED',
  };
  return mapping[status.toLowerCase()] ?? 'PENDING';
}

function mapOrderStatus(status: string): string {
  const mapping: Record<string, string> = {
    filled: 'SUCCESS',
    partial: 'PENDING',
    open: 'PENDING',
    cancelled: 'CANCELLED',
    rejected: 'REJECTED',
  };
  return mapping[status.toLowerCase()] ?? 'PENDING';
}
```

---

## Upsert Helper

Generic wrapper that adds upsert behavior and handles common errors.

```typescript
async function upsertEntity<T>(
  queue: ThrottledIngestionQueue,
  createFn: (payload: T, upsert: boolean) => Promise<unknown>,
  payload: T,
): Promise<unknown> {
  return queue.execute(() => createFn(payload, true));
}

// Usage:
await upsertEntity(queue, corsaClient.clients.createIndividualClient.bind(corsaClient.clients), clientPayload);
await upsertEntity(queue, corsaClient.deposits.createDeposit.bind(corsaClient.deposits), depositPayload);
```
