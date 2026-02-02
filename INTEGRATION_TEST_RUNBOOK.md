# Integration Test Setup Runbook

**File:** `docs/testing/INTEGRATION_TEST_RUNBOOK.md`  
**System:** NestJS + TypeORM + Microsoft SQL Server  
**Audience:** Staff / Principal Engineers  
**Status:** Production-grade standard  
**Last Updated:** 2026-02-02  

---

## Goals

1. Run **real DB integration tests** against MSSQL (transactions, locking hints, FK behavior).
2. Keep tests **fast** via **template DB cloning** (backup/restore) per Jest worker.
3. Keep tests **stable** by isolating each worker DB and **resetting tables** between tests.
4. **Fully mock Bull/Redis** (no Redis container, no real connections).

---

## Definitions

### Integration test (in this codebase)
Verifies multi-component behavior with a **real MSSQL database**:

- Service → Repository → MSSQL correctness
- Transaction boundaries (commit/rollback)
- Raw SQL + query builder correctness on MSSQL
- FK constraints / cascades
- Concurrency behavior (e.g., `SERIALIZABLE`, `UPDLOCK`, `READPAST`, `ROWLOCK`)

### Out of scope
- External HTTP calls (Stripe, Postmark, S3, third-party APIs) → **mock**
- Bull queue infra + processors → **mock / don’t import**
- Full HTTP request lifecycle → **E2E**, not integration

---

## High-level Architecture

### Local
- Run MSSQL via `docker-compose.test.yml`
- Jest global setup creates template DB + runs migrations + backs up template
- Each Jest worker restores its own DB from the template backup
- Each test cleans DB tables (`NOCHECK` → `TRUNCATE` → `WITH CHECK CHECK`)

### CI
- Use GitHub Actions service MSSQL container
- Same template cloning strategy (backup/restore)
- Bull/Redis fully mocked (no Redis service)

---

## Folder Layout

````
test/
integration/
payment/
kit/
reporting/
user/
factories/
setup/
database-template.ts
database-cleanup.ts
TestAppModule.ts
create-test-app.ts
test-hooks.ts
global-setup.ts
global-teardown.ts
jest.integration.config.js

docker-compose.test.yml
.env.test.example

````

---

## Local: MSSQL via Docker Compose

### `docker-compose.test.yml`
> Only MSSQL is required. No Redis.

```yaml
version: '3.8'
services:
  mssql-test:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: Y
      SA_PASSWORD: Test_Password_123!
      MSSQL_PID: Developer
    ports:
      - "1434:1433"
    volumes:
      - mssql-test-backup:/var/opt/mssql/backup
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'Test_Password_123!' -Q 'SELECT 1' -b -o /dev/null"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

volumes:
  mssql-test-backup:
````

---

## Env Files

### `.env.test.example` (commit this)

```bash
DATABASE_HOST=localhost
DATABASE_PORT=1434
DATABASE_USER=sa
DATABASE_PASSWORD=Test_Password_123!
DATABASE_NAME=vitract_test_template
NODE_ENV=test

# Mock external services
STRIPE_SECRET_KEY=sk_test_mock_placeholder
POSTMARK_API_TOKEN=mock_postmark_token
AWS_ACCESS_KEY=mock_aws_key
AWS_SECRET_KEY=mock_aws_secret
BUCKET_NAME=test-bucket
VITRACT_QUESTIONAIRE_API_BASE_URL=http://localhost:9999/mock
```

### `.env.test` (DO NOT COMMIT)

Same keys as `.env.test.example` with your local values.

---

## Package Scripts

```json
{
  "scripts": {
    "build": "nest build",
    "test:integration": "yarn build && jest --config test/jest.integration.config.js",
    "test:integration:watch": "yarn build && jest --config test/jest.integration.config.js --watch"
  }
}
```

**Why build first?**
TypeORM config points to `dist/**/*.entity.js` and `dist/common/migrations/*.js`.

---

## Jest Config

### `test/jest.integration.config.js`

```js
module.exports = {
  displayName: 'integration',
  rootDir: '../',
  testMatch: ['<rootDir>/test/integration/**/*.spec.ts'],
  testEnvironment: 'node',

  globalSetup: '<rootDir>/test/setup/global-setup.ts',
  globalTeardown: '<rootDir>/test/setup/global-teardown.ts',

  // IMPORTANT: used for per-worker DB restore + shared utilities
  setupFilesAfterEnv: ['<rootDir>/test/setup/test-hooks.ts'],

  testTimeout: 30000,
  maxWorkers: 4,
  retryTimes: 2,

  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  moduleFileExtensions: ['js', 'json', 'ts'],
};
```

---

## Database Strategy: Template + Per-worker Clone

### Key facts

* SQL Server creates/restores backups **inside the container**, not your host.
* The Node test runner **never reads the `.bak`** — it runs SQL `BACKUP DATABASE` / `RESTORE DATABASE`.

### `test/setup/database-template.ts`

```ts
import { DataSource } from 'typeorm';
import { dbconfig } from '../../src/config/db';

const TEMPLATE_DB = 'vitract_test_template';
const BACKUP_PATH = '/var/opt/mssql/backup/vitract_test_template.bak';

function validateDbName(name: string): void {
  if (!/^[A-Za-z0-9_]+$/.test(name)) {
    throw new Error(`Invalid database name: ${name}. Only alphanumeric + underscore allowed.`);
  }
}

function qIdent(name: string): string {
  // identifier quoting for MSSQL: escape closing bracket
  return `[${name.replace(/]/g, ']]')}]`;
}

function qNString(value: string): string {
  // safe unicode string literal: escape single quotes
  return `N'${value.replace(/'/g, "''")}'`;
}

export async function createTemplateDatabase(): Promise<void> {
  validateDbName(TEMPLATE_DB);
  const config = dbconfig.getTypeOrmConfig();

  const masterDs = new DataSource({ ...config, database: 'master' });
  await masterDs.initialize();

  // Drop template if exists
  await masterDs.query(`
    IF EXISTS (SELECT 1 FROM sys.databases WHERE name = ${qNString(TEMPLATE_DB)})
    BEGIN
      ALTER DATABASE ${qIdent(TEMPLATE_DB)} SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
      DROP DATABASE ${qIdent(TEMPLATE_DB)};
    END
  `);

  await masterDs.query(`CREATE DATABASE ${qIdent(TEMPLATE_DB)};`);
  await masterDs.destroy();

  // Run migrations against template DB
  const templateDs = new DataSource({ ...config, database: TEMPLATE_DB });
  await templateDs.initialize();
  await templateDs.runMigrations();

  // Backup template for cloning
  await templateDs.query(`
    BACKUP DATABASE ${qIdent(TEMPLATE_DB)}
    TO DISK = ${qNString(BACKUP_PATH)}
    WITH FORMAT, INIT, SKIP, NOREWIND, NOUNLOAD, STATS = 10;
  `);

  await templateDs.destroy();
}

export async function restoreWorkerDatabase(workerId: string): Promise<void> {
  const config = dbconfig.getTypeOrmConfig();
  const workerDb = `vitract_test_worker_${workerId}`;
  validateDbName(workerDb);

  const masterDs = new DataSource({ ...config, database: 'master' });
  await masterDs.initialize();

  // Drop worker DB if exists
  await masterDs.query(`
    IF EXISTS (SELECT 1 FROM sys.databases WHERE name = ${qNString(workerDb)})
    BEGIN
      ALTER DATABASE ${qIdent(workerDb)} SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
      DROP DATABASE ${qIdent(workerDb)};
    END
  `);

  // Discover logical file names
  const fileList = await masterDs.query(`
    RESTORE FILELISTONLY FROM DISK = ${qNString(BACKUP_PATH)};
  `);

  const dataFile = fileList.find((f: any) => f.Type === 'D');
  const logFile = fileList.find((f: any) => f.Type === 'L');
  if (!dataFile || !logFile) throw new Error('Backup missing data/log file entries.');

  const dataLogical = String(dataFile.LogicalName);
  const logLogical = String(logFile.LogicalName);

  // Restore
  await masterDs.query(`
    RESTORE DATABASE ${qIdent(workerDb)}
    FROM DISK = ${qNString(BACKUP_PATH)}
    WITH MOVE ${qNString(dataLogical)} TO ${qNString(`/var/opt/mssql/data/${workerDb}.mdf`)},
         MOVE ${qNString(logLogical)}  TO ${qNString(`/var/opt/mssql/data/${workerDb}_log.ldf`)},
         REPLACE;
  `);

  await masterDs.destroy();
}
```

---

## Jest Global Setup / Teardown

### `test/setup/global-setup.ts`

```ts
import { createTemplateDatabase } from './database-template';

export default async function globalSetup() {
  console.log('🔧 Creating template DB + running migrations...');
  await createTemplateDatabase();
  console.log('✅ Template DB ready');
}
```

### `test/setup/global-teardown.ts`

```ts
import { DataSource } from 'typeorm';
import { dbconfig } from '../../src/config/db';

function qIdent(name: string): string {
  return `[${name.replace(/]/g, ']]')}]`;
}
function qNString(value: string): string {
  return `N'${value.replace(/'/g, "''")}'`;
}

export default async function globalTeardown() {
  console.log('🧹 Dropping vitract_test* databases...');

  const config = dbconfig.getTypeOrmConfig();
  const masterDs = new DataSource({ ...config, database: 'master' });
  await masterDs.initialize();

  const dbs = await masterDs.query(`
    SELECT name FROM sys.databases WHERE name LIKE ${qNString('vitract_test%')};
  `);

  for (const row of dbs) {
    const dbName = String(row.name);
    // Only drop expected prefix
    if (!dbName.startsWith('vitract_test')) continue;

    await masterDs.query(`
      ALTER DATABASE ${qIdent(dbName)} SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
      DROP DATABASE ${qIdent(dbName)};
    `);
  }

  await masterDs.destroy();
  console.log('✅ Teardown complete');
}
```

---

## Per-worker DB Restore (Jest has no native per-worker hook)

We use a **lock-file strategy** inside `setupFilesAfterEnv` to ensure each worker restores once.

### `test/setup/test-hooks.ts`

```ts
import fs from 'fs';
import path from 'path';
import os from 'os';
import { restoreWorkerDatabase } from './database-template';

async function ensureWorkerDatabaseReady(): Promise<void> {
  const workerId = process.env.JEST_WORKER_ID || 'main';
  const lockFile = path.join(os.tmpdir(), `vitract-worker-db-ready-${workerId}.lock`);

  if (fs.existsSync(lockFile)) return;

  console.log(`[Worker ${workerId}] Restoring DB from template...`);
  await restoreWorkerDatabase(workerId);
  fs.writeFileSync(lockFile, new Date().toISOString());
  console.log(`[Worker ${workerId}] DB ready`);
}

beforeAll(async () => {
  await ensureWorkerDatabaseReady();
}, 120000);
```

---

## Database Cleanup Between Tests

### `test/setup/database-cleanup.ts`

```ts
import { DataSource } from 'typeorm';

export async function cleanDatabase(dataSource: DataSource): Promise<void> {
  // Disable constraints
  await dataSource.query(`EXEC sp_MSforeachtable 'ALTER TABLE ? NOCHECK CONSTRAINT ALL'`);

  const tables = await dataSource.query(`
    SELECT SCHEMA_NAME(schema_id) AS SchemaName, name AS TableName
    FROM sys.tables
    WHERE is_ms_shipped = 0
  `);

  for (const row of tables) {
    const schema = String(row.SchemaName).replace(/]/g, ']]');
    const table = String(row.TableName).replace(/]/g, ']]');
    await dataSource.query(`TRUNCATE TABLE [${schema}].[${table}]`);
  }

  // Re-enable constraints with validation
  await dataSource.query(`EXEC sp_MSforeachtable 'ALTER TABLE ? WITH CHECK CHECK CONSTRAINT ALL'`);
}
```

---

## NestJS Bootstrapping: TestAppModule

### Rules

* **Do not** import `BullModule.forRoot()` (it tries to connect to Redis).
* **Do not** import modules that register Bull processors/consumers.
* Use a dedicated `TestAppModule` so repositories bind to the test DataSource.

### `test/setup/TestAppModule.ts`

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule } from '@nestjs/config';
import { EventEmitterModule } from '@nestjs/event-emitter';
import { CacheModule } from '@nestjs/cache-manager';
import { dbconfig } from '../../src/config/db';

// Import feature modules (mirror AppModule) but EXCLUDE queue processor modules
import { UserModule } from '../../src/user/user.module';
import { PractitionerModule } from '../../src/practitioner/practitioner.module';
import { KitModule } from '../../src/kit/kit.module';
import { PaymentModule } from '../../src/payment/payment.module';
import { OrderModule } from '../../src/order/order.module';
import { BillingModule } from '../../src/billing/billing.module';
import { ReportingModule } from '../../src/reporting/reporting.module';

export function getTestTypeOrmConfig(workerId: string) {
  const base = dbconfig.getTypeOrmConfig();
  return {
    ...base,
    database: `vitract_test_worker_${workerId}`,
    logging: process.env.CI ? false : ['error', 'warn'],
    entities: ['dist/**/*.entity.js'],
    migrations: ['dist/common/migrations/*.js'],
    migrationsRun: false,
  };
}

@Module({
  imports: [
    EventEmitterModule.forRoot(),
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRoot(getTestTypeOrmConfig(process.env.JEST_WORKER_ID || 'main')),
    CacheModule.register({
      isGlobal: true,
      ttl: 60,
      store: 'memory',
    }),

    UserModule,
    PractitionerModule,
    KitModule,
    PaymentModule,
    OrderModule,
    BillingModule,
    ReportingModule,
  ],
})
export class TestAppModule {}
```

---

## Fully Mock Bull/Redis at Provider Level

### `test/setup/create-test-app.ts`

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { getQueueToken } from '@nestjs/bull';
import { TestAppModule } from './TestAppModule';
import Stripe from 'stripe';

export async function createTestApp() {
  const mockStripe = {
    customers: { create: jest.fn(), retrieve: jest.fn(), update: jest.fn() },
    checkout: { sessions: { create: jest.fn(), retrieve: jest.fn() } },
    paymentIntents: { create: jest.fn(), confirm: jest.fn() },
    invoices: { create: jest.fn(), finalizeInvoice: jest.fn(), pay: jest.fn() },
  } as any as jest.Mocked<Stripe>;

  const mockPostmark = {
    sendEmail: jest.fn().mockResolvedValue({ MessageID: 'mock-id' }),
    sendEmailBatch: jest.fn().mockResolvedValue([{ MessageID: 'mock-id' }]),
  };

  const mockS3 = {
    upload: jest.fn().mockResolvedValue({ Location: 'https://mock-s3.com/file.pdf' }),
    deleteObject: jest.fn().mockResolvedValue({}),
    getSignedUrl: jest.fn().mockReturnValue('https://mock-s3.com/signed-url'),
  };

  // List every Bull queue name used in your app:
  const queueNames = ['email', 'payment', 'billing', 'reporting'];

  const moduleBuilder = Test.createTestingModule({
    imports: [TestAppModule],
  })
    .overrideProvider('STRIPE_CLIENT').useValue(mockStripe)
    .overrideProvider('POSTMARK_CLIENT').useValue(mockPostmark)
    .overrideProvider('S3_CLIENT').useValue(mockS3)
    .overrideProvider('VitractHealthInfoClient')
    .useValue({
      checkHealthInfoStatus: jest.fn().mockResolvedValue({ completed: true }),
      submitHealthInfo: jest.fn().mockResolvedValue({ success: true }),
    });

  // Override Bull queues (no Redis)
  for (const name of queueNames) {
    moduleBuilder.overrideProvider(getQueueToken(name)).useValue({
      add: jest.fn().mockResolvedValue({ id: 'mock-job-id' }),
      on: jest.fn(),
      getJob: jest.fn(),
      getJobs: jest.fn().mockResolvedValue([]),
    });
  }

  const moduleFixture: TestingModule = await moduleBuilder.compile();
  const app: INestApplication = moduleFixture.createNestApplication();
  await app.init();

  return {
    app,
    module: moduleFixture,
    mocks: { stripe: mockStripe, postmark: mockPostmark, s3: mockS3 },
  };
}
```

---

## Example Test Skeleton

```ts
import { INestApplication } from '@nestjs/common';
import { TestingModule } from '@nestjs/testing';
import { DataSource } from 'typeorm';
import { createTestApp } from '../../setup/create-test-app';
import { cleanDatabase } from '../../setup/database-cleanup';
import { BaseOrderService } from '../../../src/order/service/base-order.service';

describe('BaseOrderService (Integration)', () => {
  let app: INestApplication;
  let module: TestingModule;
  let dataSource: DataSource;
  let service: BaseOrderService;

  beforeAll(async () => {
    const testApp = await createTestApp();
    app = testApp.app;
    module = testApp.module;
    service = module.get(BaseOrderService);
    dataSource = module.get(DataSource);
  });

  afterAll(async () => {
    await app.close();
  });

  beforeEach(async () => {
    await cleanDatabase(dataSource);
    jest.clearAllMocks();
  });

  it('...', async () => {
    // arrange (factories)
    // act
    // assert (verify db state)
  });
});
```

---

## CI (GitHub Actions) Notes

* Run only `yarn test:integration` after `yarn build`.
* Ensure env uses `DATABASE_PORT=1433` for the service container.
* Keep `CI=true` in env to disable noisy SQL logs.

**CI must rely on cloning**: template DB + backup + per-worker restore.

---

## Anti-patterns (hard no)

* Using SQLite for integration tests
* `synchronize: true`
* Mocking TypeORM repositories / DataSource
* Importing `BullModule.forRoot()` in test modules
* Importing queue processor modules in tests
* Sharing data between tests (always clean DB)

---

## Troubleshooting Cheatsheet

### Backup/restore fails in CI

* Check MSSQL logs for permissions to `/var/opt/mssql/backup`
* Ensure backup path is inside container
* Confirm global setup runs before tests
* Increase `beforeAll` timeout in `test-hooks.ts` (restore can be slow)

### “Redis connection refused”

* You imported `BullModule.forRoot()` somewhere in test module path
* You didn’t override a queue token used by the app

### Worker DB not found

* `setupFilesAfterEnv` not pointing to `test/setup/test-hooks.ts`
* Lock file created but restore failed (delete lock files in `/tmp` and retry)

---

## Checklist

* [ ] `docker-compose.test.yml` has MSSQL only (no Redis)
* [ ] `globalSetup` creates template DB + backups to `/var/opt/mssql/backup/...`
* [ ] `test-hooks.ts` restores per worker exactly once (lock file)
* [ ] `TestAppModule` excludes Bull infra and processor modules
* [ ] `create-test-app.ts` overrides all queue tokens via `getQueueToken`
* [ ] Each test calls `cleanDatabase(dataSource)` in `beforeEach`
* [ ] CI uses cloning strategy (not per-worker migrations)

---
