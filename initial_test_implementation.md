# Test Strategy + Implementation Runbook
**Repository:** vitract-kit-api-v1  
**Runtime today:** Single-process NestJS (HTTP + Cron + Bull queues)  
**Goal:** Establish industry-grade testing + CI gates that are reliable **now**, and become even cleaner once workers split out.

---

## 0) Outcomes we want (definition of “done”)
By the end of this runbook rollout, the repo should have:

1) **Clear test layers**
- Unit tests: fast, deterministic, no network/DB/Redis
- Integration tests: real MSSQL (TypeORM) + optional Redis, real module wiring
- E2E tests: minimal business-critical flows only

2) **Test infrastructure**
- Test docker compose for MSSQL + Redis
- Test app bootstrap helpers (consistent global pipes/filters)
- DB lifecycle helpers (migrations + cleanup/reset)
- Queue mocking helpers (assert enqueues) + optional “real queue processing” mode

3) **CI gates**
- unit tests in CI
- integration tests in CI with MSSQL + Redis services
- e2e tests in CI *optional* (or nightly)
- coverage thresholds that start low and ratchet upward

4) **A phased, low-risk implementation path**
- Start by testing the highest-risk business logic (billing, payment webhook routing, health sync, mail processing)
- Add HTTP integration tests for key endpoints
- Add cron and queue guardrails (idempotency/locks) + tests
- Prepare for future worker split with minimal refactor

---

## 1) Test policy (repo-specific rules)

### 1.1 What must be unit-tested in this repo
Unit tests are mandatory for:
- **Billing** logic (`BillingSchedulerService.execute()` and core billing calculations/filters)
- **Webhook routing** (`payment-webhook` service routes events correctly, idempotency behavior)
- **Queue processor core logic** (especially `HealthInfoSyncProcessor` and Stripe sync processors)
- **Cron “should enqueue / should run”** logic (verify jobId usage + date guards)
- **Mail consumer job handlers** (payload validation + transactional boundaries)

Unit tests should:
- Mock external APIs: Stripe, Postmark, Mailgun, S3, Cloudinary, Puppeteer, VitractHealthInfoClient
- Not require Redis or MSSQL
- Run in < ~30 seconds locally for the whole suite (target)

### 1.2 What must be integration-tested in this repo
Integration tests must cover:
- TypeORM mappings/migrations sanity (migrations apply cleanly on test DB)
- Transactional flows (especially mail consumer which uses its own DataSource)
- Selected HTTP endpoints wired through Nest modules and global pipes/filters
- Enqueue behavior (assert `queue.add()` called with correct payload + jobId)

Integration tests should:
- Use real MSSQL in Docker
- Avoid calling real external services (mock/stub those)
- Prefer “assert enqueue” rather than “process job” unless explicitly testing processors end-to-end

### 1.3 What must be E2E-tested in this repo
E2E tests are intentionally few and focus on *business correctness*:
- Order create → enqueues kit registration
- Payment webhook (invoice paid) → enqueues invoice processing
- Support ticket create → enqueues initial support mail

E2E tests should:
- Start the Nest app (full AppModule)
- Use real MSSQL + Redis
- Mock external services
- Be deterministic (no cron firing in the background, no flakiness)

---

## 2) Target test pyramid for vitract-kit-api-v1

### Recommended ratio (practical, not dogmatic)
- **Unit tests:** 70–80%
- **Integration tests:** 15–25%
- **E2E tests:** 5–10% (tiny)

### Why this fits your current architecture
Because you run HTTP + cron + Bull in the same process today:
- You want most confidence to come from **unit tests + integration tests** that don’t depend on background job timing.
- E2E tests can become flaky if jobs/cron run during them, so keep E2E minimal.

---

## 3) Repository structure & naming conventions

### 3.1 File patterns
Current Jest config:
- unit tests: `src/**/*.spec.ts` (rootDir = `src`)
- e2e: `test/**/*.e2e-spec.ts`

We’ll introduce clear subfolders and optional Jest “projects” setup (recommended), but you can ship incrementally.

### 3.2 Proposed test folders
Create:
- `src/**/__tests__/` (unit tests colocated) OR keep `*.spec.ts` beside files
- `test/integration/` (integration tests)
- `test/e2e/` (e2e tests)
- `test/helpers/` (shared test tooling)

Example:

src/
billing/
services/
billing-scheduler.service.ts
**tests**/
billing-scheduler.service.spec.ts
queues/
processors/
health-info-sync.processor.ts
**tests**/
health-info-sync.processor.spec.ts

test/
integration/
db-migrations.int-spec.ts
support-flow.int-spec.ts
e2e/
order-create.e2e-spec.ts
payment-webhook-invoice-paid.e2e-spec.ts
helpers/
test-app.ts
db.ts
queues.ts
fixtures.ts


### 3.3 Naming
- Unit: `*.spec.ts`
- Integration: `*.int-spec.ts` (or `.integration-spec.ts`)
- E2E: `*.e2e-spec.ts`

---

## 4) Test environments & config

### 4.1 Environment variables
Add a test env file (local only; do not commit secrets):
- `.env.test` (gitignored)
- Optionally `.env.test.example` committed

Minimal env keys (based on your report):
- MSSQL: `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_USER`, `DATABASE_PASSWORD`, `DATABASE_NAME`
- Redis: `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` (optional)
- Stripe: use fake key, disable outbound calls, or point to mocks
- Mail providers: fake tokens, disable outbound calls
- Sentry: disable in test

### 4.2 Hard rule: no real network in tests
In test mode:
- external HTTP should be blocked or mocked.
- Use dependency injection to supply mock clients.

If you can, add a “network guard” in tests (optional):
- monkeypatch `http(s).request` to throw unless explicitly allowed
- or use `nock` to intercept and fail unknown outbound requests

### 4.3 Ensure cron does not auto-run during tests
Because cron + Bull is in same process, tests must prevent background noise.

Options:
1) **Disable scheduler in tests** (recommended)
   - Only import `ScheduleModule.forRoot()` in non-test environments
   - Or provide a config flag: `ENABLE_SCHEDULER=false` in `.env.test`
2) **Keep scheduler but ensure no cron methods run**
   - Harder to guarantee.

Recommended implementation:
- In `CronModule`:
  - conditionally register schedule module/providers when `ENABLE_SCHEDULER === 'true'`
- In tests, set `ENABLE_SCHEDULER=false`

---

## 5) Docker setup for tests (MSSQL + Redis)

### 5.1 Add a dedicated compose file
Create: `docker-compose.test.yml`

Example:
```yaml
version: "3.9"

services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: vitract_mssql_test
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "YourStrong(!)Password"
      MSSQL_PID: "Express"
    ports:
      - "14339:1433"
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourStrong(!)Password' -Q 'SELECT 1' || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 20

  redis:
    image: redis:7-alpine
    container_name: vitract_redis_test
    ports:
      - "16379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 30
```

### 5.2 Local developer workflow

* Start services:

  * `docker compose -f docker-compose.test.yml up -d`
* Run unit tests (no docker required):

  * `yarn test:unit`
* Run integration tests (requires docker):

  * `yarn test:int`
* Run e2e tests:

  * `yarn test:e2e`

---

## 6) Jest configuration strategy (recommended: projects)

Right now, Jest is inline in `package.json`. That’s workable but hard to scale.

### 6.1 Recommended: move config to files

Create:

* `jest.unit.config.ts`
* `jest.int.config.ts`
* `test/jest-e2e.json` (already exists; you can keep it or convert to TS)

Example `jest.unit.config.ts`:

```ts
import type { Config } from 'jest';

const config: Config = {
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  testEnvironment: 'node',
  transform: { '^.+\\.(t|j)s$': 'ts-jest' },
  moduleFileExtensions: ['ts', 'js', 'json'],
  collectCoverageFrom: ['**/*.ts', '!**/*.d.ts', '!**/main.ts'],
  coverageDirectory: '../coverage/unit',
  clearMocks: true,
  resetMocks: true,
  restoreMocks: true,
};

export default config;
```

Example `jest.int.config.ts`:

```ts
import type { Config } from 'jest';

const config: Config = {
  testEnvironment: 'node',
  transform: { '^.+\\.(t|j)s$': 'ts-jest' },
  testMatch: ['<rootDir>/test/integration/**/*.int-spec.ts'],
  moduleFileExtensions: ['ts', 'js', 'json'],
  setupFilesAfterEnv: ['<rootDir>/test/helpers/jest.setup.ts'],
  testTimeout: 60000,
  coverageDirectory: './coverage/integration',
  clearMocks: true,
  resetMocks: true,
  restoreMocks: true,
};

export default config;
```

### 6.2 Update scripts in package.json

Add:

```json
{
  "scripts": {
    "test:unit": "jest -c jest.unit.config.ts",
    "test:int": "jest -c jest.int.config.ts",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "test:all": "yarn test:unit && yarn test:int && yarn test:e2e"
  }
}
```

### 6.3 If you don’t want config files yet

You can keep inline jest config and just add `testMatch` overrides per script.
But projects are cleaner long term.

---

## 7) Shared test helpers (must-have)

Create a folder: `test/helpers/`

### 7.1 Test app factory (Nest bootstrap for tests)

Create: `test/helpers/test-app.ts`

Responsibilities:

* Build a Nest `INestApplication` with your global pipes/filters consistent with production
* Provide hooks to override providers easily (Stripe client, external clients)
* Allow disabling scheduler and optionally disabling queue processors

Pseudo-implementation:

```ts
import { Test } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { AppModule } from '../../src/app.module';
import { AllExceptionsFilter } from '../../src/common/filters/all-exceptions.filter';

export async function createTestApp(overrides?: {
  providers?: Array<{ token: any; useValue: any }>;
  disableScheduler?: boolean;
}): Promise<INestApplication> {
  // Set env flags early (scheduler disable)
  if (overrides?.disableScheduler) {
    process.env.ENABLE_SCHEDULER = 'false';
  }

  const moduleBuilder = Test.createTestingModule({
    imports: [AppModule],
  });

  // Apply provider overrides
  for (const p of overrides?.providers ?? []) {
    moduleBuilder.overrideProvider(p.token).useValue(p.useValue);
  }

  const moduleRef = await moduleBuilder.compile();
  const app = moduleRef.createNestApplication();

  // Mirror prod behavior
  app.setGlobalPrefix('api/v1');
  app.useGlobalPipes(new ValidationPipe({ transform: true }));
  app.useGlobalFilters(new AllExceptionsFilter());

  await app.init();
  return app;
}
```

> Note: You’ll likely need to match your real `main.ts` behavior closely (prefix exclusions, swagger middleware not needed in tests).

### 7.2 DB helpers (migrations + cleanup)

Create: `test/helpers/db.ts`

Responsibilities:

* Create a DataSource using test env vars
* Run migrations before suite
* Clean tables between tests (fastest approach: delete in correct order) OR recreate DB per test suite

Options:
A) **Migrate once per test suite + truncate tables per test** (recommended)
B) **Drop DB and recreate** (slower on MSSQL)
C) **Wrap each test in a transaction and rollback** (harder with multiple connections / background jobs)

Given you have jobs and sometimes separate DataSource pools, prefer A.

Pseudo:

```ts
import { DataSource } from 'typeorm';
import { dbConfigForTest } from './db-config';

let dataSource: DataSource | null = null;

export async function getTestDataSource(): Promise<DataSource> {
  if (!dataSource) {
    dataSource = new DataSource(dbConfigForTest());
    await dataSource.initialize();
  }
  return dataSource;
}

export async function runMigrations(): Promise<void> {
  const ds = await getTestDataSource();
  await ds.runMigrations();
}

export async function cleanupDatabase(): Promise<void> {
  const ds = await getTestDataSource();

  // Use raw SQL delete order to satisfy FKs.
  // Keep this in ONE place so it’s consistent.
  await ds.query(`DELETE FROM ...`); // fill with tables
}

export async function closeTestDataSource(): Promise<void> {
  if (dataSource) {
    await dataSource.destroy();
    dataSource = null;
  }
}
```

### 7.3 Queue helpers

Create: `test/helpers/queues.ts`

Responsibilities:

* Provide a mock queue object that captures `.add()` calls
* Provide assertion helpers for job name, payload, jobId, options

Example:

```ts
export function createMockQueue() {
  return {
    add: jest.fn().mockResolvedValue({ id: 'mock-job-id' }),
  };
}

export function expectEnqueued(queue: any, jobName: string, matcher?: any) {
  expect(queue.add).toHaveBeenCalled();
  const calls = queue.add.mock.calls;
  const matched = calls.some(([name, data, opts]: any[]) => {
    if (name !== jobName) return false;
    if (matcher?.data) expect(data).toEqual(expect.objectContaining(matcher.data));
    if (matcher?.opts) expect(opts).toEqual(expect.objectContaining(matcher.opts));
    return true;
  });
  expect(matched).toBe(true);
}
```

### 7.4 Jest global setup

Create: `test/helpers/jest.setup.ts`

Responsibilities:

* Set env flags for tests
* Set default timeout for integration tests
* Optionally block outbound network

Example:

```ts
process.env.NODE_ENV = 'test';
process.env.ENABLE_SCHEDULER = 'false';
jest.setTimeout(60000);
```

---

## 8) Unit test implementation guide (how to write them in this repo)

### 8.1 Pattern: service unit test with mocked repos

Use Nest TestingModule but override repositories and external clients.

Example: BillingSchedulerService unit test skeleton

```ts
import { Test } from '@nestjs/testing';
import { BillingSchedulerService } from '../billing-scheduler.service';

describe('BillingSchedulerService', () => {
  let service: BillingSchedulerService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        BillingSchedulerService,
        { provide: 'SomeRepoToken', useValue: { find: jest.fn() } },
        { provide: 'StripeClient', useValue: { charges: { create: jest.fn() } } },
      ],
    }).compile();

    service = module.get(BillingSchedulerService);
  });

  it('runs only when date guard passes', async () => {
    // Arrange: mock date/time
    // Act
    // Assert
  });
});
```

### 8.2 Pattern: processor unit test (core logic isolated)

Processors often do too much. The goal is to test:

* selection query criteria (only correct kits)
* batching rules
* updates on success
* no update on failure
* error propagation (so Bull retries)

Example: HealthInfoSyncProcessor unit test approach

* Mock repository query returning rows
* Mock `VitractHealthInfoClient.exists` with success/failure
* Assert DB update calls executed only for success rows
* Assert concurrency/batch sizing is adhered to (or at least not unbounded)

### 8.3 Payload validation tests for MailConsumer

Because you have multiple handlers:

* `send-mail`
* `send-mail-with-template`
* `send-initial-support-mail`
* `send-support-mail`

Write tests for:

* missing required fields → handler throws (job fails)
* valid payload → calls mail service with expected args
* transactional sections commit/rollback properly (unit test can only assert called; integration will assert DB state)

---

## 9) Integration tests implementation guide (MSSQL + module wiring)

### 9.1 Integration test lifecycle

For integration tests:

1. Start docker MSSQL + Redis
2. Configure `.env.test` to point to those ports (e.g. 14339 and 16379)
3. In Jest beforeAll:

   * run migrations
4. BeforeEach:

   * cleanup DB (delete tables)
   * seed minimal data
5. AfterAll:

   * close DataSource
   * app.close()

### 9.2 Seed strategy

Create `test/helpers/fixtures.ts` for minimal fixtures:

* create user
* create practitioner (if required)
* create kit/order rows as needed

Keep fixtures:

* minimal
* deterministic
* centralized

### 9.3 Example integration test: migration sanity

File: `test/integration/db-migrations.int-spec.ts`
Purpose:

* prove migrations apply cleanly on empty DB
* catch missing entity/migration issues early

Test:

* create DataSource
* run migrations
* assert no error
* optionally query `__migrations` table if present

### 9.4 Example integration test: mail consumer transaction

Because mail consumer creates its own DataSource pool:

* This is a risk area.
* Integration test should ensure it doesn’t leak connections and that DB transaction behaves.

Test idea:

* Insert a support ticket row
* Invoke handler method directly with payload (not through Bull)
* Assert ticket status updated + message saved
* Force mail provider mock to throw → assert rollback (no partial updates)

### 9.5 HTTP integration tests

Use `createTestApp` + `supertest`:

* Ensure validation pipe is on (transform true)
* Ensure exception filter behaves correctly (error shape stable)

---

## 10) E2E tests implementation guide (minimal, deterministic)

### 10.1 E2E prerequisites

* Scheduler disabled
* External services mocked
* Use real DB + Redis
* Only test critical flows

### 10.2 E2E test structure

In `test/e2e/`:

* each file boots app once (or use global bootstrap to reuse app)
* cleanup DB between tests

Example E2E: order create enqueues job

* Make request: `POST /api/v1/order`
* Assert: HTTP 201 + response shape
* Assert: queue.add called with `auto-register-practitioner-order-kits` (or whichever job is expected)

> If the system enqueues using `QueueService`, override that service in test app with a mocked queue service.

---

## 11) Coverage strategy (ratchet plan)

You currently have ~0 coverage meaningfulness.

### 11.1 Starting thresholds (safe)

* Global statements: 35%
* Branches: 20%
* Functions: 25%
* Lines: 35%

### 11.2 Ratchet schedule

* Every week: +5% on statements/lines until you reach 70%
* After 70%: ratchet per-module thresholds for critical modules:

  * billing
  * payment
  * queues
  * cron
  * mail/support

### 11.3 Enforcement

Add in Jest config:

```ts
coverageThreshold: {
  global: {
    statements: 35,
    branches: 20,
    functions: 25,
    lines: 35,
  },
},
```

---

## 12) CI Implementation (GitHub Actions)

### 12.1 Add a new workflow or extend existing

You already have `.github/workflows/deploy.yaml` with a `ci` job.
Extend it with service containers for MSSQL and Redis for integration tests.

High-level:

* job 1: unit (no services)
* job 2: integration (services: mssql + redis)
* optional job 3: e2e (same services, runs fewer tests)

### 12.2 Integration job requirements

* wait for MSSQL health
* create DB if needed
* run migrations
* run integration tests

---

## 13) Rollout plan (phased execution)

### Phase 1 — Foundations (1–2 days)

Deliverables:

* docker-compose.test.yml
* test helpers: `test-app.ts`, `db.ts`, `queues.ts`, `fixtures.ts`
* scripts: `test:unit`, `test:int`, `test:e2e`
* disable scheduler in tests

Acceptance:

* Unit tests run without docker
* Integration tests can run with docker locally
* No flaky cron behavior in tests

### Phase 2 — First wave unit tests (2–5 days)

Must add unit tests for:

* BillingSchedulerService date guard + selection rules
* Payment webhook event routing (invoice paid)
* HealthInfoSyncProcessor (selection + update behavior)
* MailConsumer payload validation + error propagation

Acceptance:

* Coverage moves noticeably (even if still low)
* Tests run fast and stable

### Phase 3 — Integration tests (3–7 days)

Must add:

* migration sanity
* mail consumer transaction integration
* one or two HTTP integration tests: order create, support create

Acceptance:

* CI runs integration tests with MSSQL + Redis

### Phase 4 — Minimal E2E set (optional initially)

Add 3 E2E tests:

* order create → enqueue expected job
* webhook invoice paid → enqueue job
* support create → enqueue initial support mail

Acceptance:

* Run in CI or nightly (your choice)

### Phase 5 — Hardening & worker split prep (future)

* Introduce worker mode flags
* Improve cron overlap protection + tests
* Add DLQ/monitoring

---

## 14) Work items list (ready to paste into your task tracker)

### WI-TST-001: Add docker-compose.test.yml for MSSQL + Redis

* Add file and documentation in README
* Ensure health checks work

### WI-TST-002: Add test helper layer

* `test/helpers/test-app.ts`
* `test/helpers/db.ts`
* `test/helpers/queues.ts`
* `test/helpers/fixtures.ts`
* `test/helpers/jest.setup.ts`

### WI-TST-003: Add scripts + Jest config split

* `test:unit`, `test:int`, `test:e2e`
* optional move jest config out of package.json

### WI-TST-004: Disable scheduler in tests

* Config flag `ENABLE_SCHEDULER`
* CronModule conditional registration

### WI-TST-005: Unit tests — BillingSchedulerService

* Date guard behavior
* Eligible user selection behavior
* Failure propagation

### WI-TST-006: Unit tests — Payment webhook routing

* invoice paid path enqueues job
* unknown event types handled safely

### WI-TST-007: Unit tests — HealthInfoSyncProcessor

* selects only `healthInfoCompleted = NO`
* updates only on external success
* batch paging safe

### WI-TST-008: Unit tests — MailConsumer handlers

* payload validation
* transactional boundary behavior (unit-level)
* failure propagation

### WI-TST-009: Integration tests — migrations sanity + mail transaction

* run migrations on clean DB
* mail consumer rollback on failure

### WI-TST-010: HTTP integration tests — order + support

* validation pipe enforcement
* error shape stable

### WI-TST-011: CI — add integration job with MSSQL + Redis services

* ensure stable waits
* run integration suite

### WI-TST-012: Coverage ratchet enforcement

* start low, raise weekly
* module-specific thresholds later

---

## 15) Common pitfalls & how to avoid them (repo-specific)

1. **Cron running during tests**

   * must be disabled with env flag
2. **Multiple TypeORM DataSources**

   * mail consumer uses its own pool; integration tests must detect leaks and partial commits
3. **DB cleanup order**

   * MSSQL + FKs means delete order matters; centralize cleanup SQL
4. **Bull processors executing unexpectedly**

   * prefer mocking queue `.add()`; only run processors end-to-end in dedicated tests
5. **Stripe/mailer external calls**

   * must be mocked; never hit real Stripe/Postmark/Mailgun

---

## 16) Definition-of-ready checklist (before coding tests)

* [ ] docker-compose.test.yml starts MSSQL and Redis successfully
* [ ] `.env.test.example` exists and documents required env vars
* [ ] scheduler disabled in tests
* [ ] integration tests can run migrations reliably
* [ ] you know which providers to override for external clients

---

## 17) Definition-of-done checklist (after implementing this runbook)

* [ ] `yarn test:unit` passes locally without docker
* [ ] `yarn test:int` passes locally with docker MSSQL+Redis
* [ ] CI runs unit + integration on PRs
* [ ] At least 10–20 meaningful unit tests exist for core flows
* [ ] At least 3 meaningful integration tests exist (migrations + mail tx + one HTTP flow)
* [ ] E2E tests exist (optional) and are deterministic
* [ ] Coverage threshold enforced with a ratchet plan

````

# Implementation Appendix (Detailed, Step-by-step)

This appendix contains concrete implementation steps, code skeletons, and CI examples tailored to vitract-kit-api-v1.

---

## A) Step-by-step: Add test docker compose

### A1) Create `docker-compose.test.yml`
Use the example in the runbook. Prefer stable ports that don’t conflict:
- MSSQL: `14339 -> 1433`
- Redis: `16379 -> 6379`

### A2) Create `.env.test.example`
```env
NODE_ENV=test
ENABLE_SCHEDULER=false

DATABASE_HOST=127.0.0.1
DATABASE_PORT=14339
DATABASE_USER=sa
DATABASE_PASSWORD=YourStrong(!)Password
DATABASE_NAME=vitract_kit_test

REDIS_HOST=127.0.0.1
REDIS_PORT=16379
REDIS_PASSWORD=
````

### A3) Add docs to README (or `docs/testing.md`)

Document:

* start services
* run unit tests
* run integration tests
* run e2e tests

---

## B) Step-by-step: Disable scheduler in tests

### B1) Add config flag usage

Where you define cron module: `src/cron/cron.module.ts`

Current: it imports `ScheduleModule.forRoot()` always.

Change: register it conditionally.

Example approach:

```ts
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { ScheduleService } from './cron.service';

@Module({
  imports: [
    ConfigModule,
    // Only enable scheduler when flag true
    ...(process.env.ENABLE_SCHEDULER === 'true' ? [ScheduleModule.forRoot()] : []),
  ],
  providers: [ScheduleService],
})
export class CronModule {}
```

> Alternative (cleaner): use ConfigService, but keep it minimal for now.

### B2) Ensure `.env.test` sets `ENABLE_SCHEDULER=false`

And ensure your test setup file enforces it too.

---

## C) Step-by-step: Test helper layer

### C1) `test/helpers/jest.setup.ts`

```ts
process.env.NODE_ENV = 'test';
process.env.ENABLE_SCHEDULER = 'false';
jest.setTimeout(60000);
```

### C2) `test/helpers/test-app.ts` (full skeleton)

```ts
import { Test } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { AppModule } from '../../src/app.module';
import { AllExceptionsFilter } from '../../src/common/filters/all-exceptions.filter';

type Override = { token: any; useValue: any };

export async function createTestApp(opts?: {
  overrides?: Override[];
  globalPrefix?: string;
}): Promise<INestApplication> {
  // Ensure cron doesn't run
  process.env.ENABLE_SCHEDULER = 'false';

  const builder = Test.createTestingModule({ imports: [AppModule] });

  for (const o of opts?.overrides ?? []) {
    builder.overrideProvider(o.token).useValue(o.useValue);
  }

  const moduleRef = await builder.compile();
  const app = moduleRef.createNestApplication();

  // Mirror production where it matters
  app.setGlobalPrefix(opts?.globalPrefix ?? 'api/v1');
  app.useGlobalPipes(new ValidationPipe({ transform: true }));
  app.useGlobalFilters(new AllExceptionsFilter());

  await app.init();
  return app;
}
```

> If your `AllExceptionsFilter` depends on DI, instantiate it via moduleRef:
> `app.useGlobalFilters(moduleRef.get(AllExceptionsFilter))`

### C3) `test/helpers/db-config.ts`

Create a dedicated function to build a DataSource config using env vars.
This avoids accidentally using prod config.

```ts
import type { DataSourceOptions } from 'typeorm';

export function dbConfigForTest(): DataSourceOptions {
  return {
    type: 'mssql',
    host: process.env.DATABASE_HOST,
    port: Number(process.env.DATABASE_PORT),
    username: process.env.DATABASE_USER,
    password: process.env.DATABASE_PASSWORD,
    database: process.env.DATABASE_NAME,

    // IMPORTANT: point to entities/migrations the same way prod does in tests
    // If you use ts-jest, you can reference TS paths
    entities: [__dirname + '/../../src/**/*.entity.ts'],
    migrations: [__dirname + '/../../src/common/migrations/*.ts'],

    synchronize: false,
    options: { encrypt: false, trustServerCertificate: true },
  };
}
```

> You may need to mirror your existing `DbConfig` exactly, but ensure it points to TS files during tests.

### C4) `test/helpers/db.ts` (migrations + cleanup)

```ts
import { DataSource } from 'typeorm';
import { dbConfigForTest } from './db-config';

let ds: DataSource | null = null;

export async function getTestDataSource(): Promise<DataSource> {
  if (!ds) {
    ds = new DataSource(dbConfigForTest());
    await ds.initialize();
  }
  return ds;
}

export async function migrateTestDb(): Promise<void> {
  const dataSource = await getTestDataSource();
  await dataSource.runMigrations();
}

// Centralize cleanup. Start with tables you KNOW you touch.
// Expand as you add integration tests.
export async function cleanupTestDb(): Promise<void> {
  const dataSource = await getTestDataSource();

  // Example pattern; replace with your real tables and FK order.
  // Prefer DELETE, not TRUNCATE, due to FKs and MSSQL constraints.
  await dataSource.query(`DELETE FROM support_messages`);
  await dataSource.query(`DELETE FROM support_tickets`);
  await dataSource.query(`DELETE FROM orders`);
  await dataSource.query(`DELETE FROM kits`);
  await dataSource.query(`DELETE FROM users`);
}

export async function closeTestDb(): Promise<void> {
  if (ds) {
    await ds.destroy();
    ds = null;
  }
}
```

### C5) `test/helpers/fixtures.ts`

```ts
import { DataSource } from 'typeorm';

export async function seedUser(ds: DataSource, overrides?: Partial<any>) {
  // Prefer repository insert if you have entities imported.
  // Or use ds.query for speed.
  const user = { /* minimal columns */ ...overrides };
  // return created record
  return user;
}
```

---

## D) Step-by-step: Queue mocking strategy

### D1) Prefer “assert enqueue” over “run processor” (for most tests)

Why:

* stable
* fast
* avoids concurrency flakiness

### D2) How to override QueueService or queues

If your code uses `QueueService` as an abstraction (you listed it), override it in tests.

Example:

```ts
const mockQueueService = {
  addHealthInfoSyncJob: jest.fn(),
  addReconcileProcessingJob: jest.fn(),
  addSendMailJob: jest.fn(),
};

const app = await createTestApp({
  overrides: [{ token: QueueService, useValue: mockQueueService }],
});
```

If your code injects Bull queues directly via `@InjectQueue('mail')`, you can override those providers too.

---

## E) First test suite (exact priorities + what to assert)

### E1) Unit: Payment webhook routing (invoice paid)

**File:** `src/payment/services/payment-webhook.ts` (or similar)

Tests:

1. When event type is “invoice.paid” (or equivalent), it enqueues `process-invoice-payment`
2. When event type is unknown, it returns 200 (or safe response) and does not enqueue
3. It validates signature (either mocked or verified using test secret)

Assertions:

* correct job name
* correct payload contains invoice id/customer id
* errors throw (so webhook can retry or at least logs)

### E2) Unit: BillingSchedulerService date guards

Tests:

* outside date window → no charges
* inside window with eligible user list → statement creation called
* if Stripe throws → error is propagated/logged properly

Assertions:

* service selects correct users
* it doesn’t double-process in the same run (if you have dedupe keys)

### E3) Unit: HealthInfoSyncProcessor

Tests:

* selects only kits with `healthInfoCompleted = NO`
* if client.exists returns true → updates kit
* if client.exists throws → does not update, error propagates

Assertions:

* update called expected times
* batch query called with correct limits
* no unbounded loops

### E4) Unit: MailConsumer handlers

Tests:

* required fields missing → throw
* valid payload → calls mail service
* provider throw → job fails (throw), not swallow

---

## F) Integration tests (first two are mandatory)

### F1) Integration: migrations sanity

**File:** `test/integration/db-migrations.int-spec.ts`

Pseudo:

```ts
import { migrateTestDb, closeTestDb } from '../helpers/db';

describe('DB migrations', () => {
  it('applies migrations cleanly', async () => {
    await migrateTestDb();
  });

  afterAll(async () => {
    await closeTestDb();
  });
});
```

### F2) Integration: mail consumer transaction rollback

* Setup DB with a support ticket
* Call handler with payload
* Force provider error
* Assert DB did not partially update

---

## G) CI: GitHub Actions example (integration job)

Example snippet to add to your workflow:

```yaml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn typecheck
      - run: yarn lint:check
      - run: yarn format:check
      - run: yarn test:unit

  integration:
    runs-on: ubuntu-latest
    services:
      mssql:
        image: mcr.microsoft.com/mssql/server:2022-latest
        env:
          ACCEPT_EULA: Y
          SA_PASSWORD: YourStrong(!)Password
          MSSQL_PID: Express
        ports:
          - 1433:1433
        options: >-
          --health-cmd "/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourStrong(!)Password' -Q 'SELECT 1' || exit 1"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 20
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 30

    env:
      NODE_ENV: test
      ENABLE_SCHEDULER: "false"
      DATABASE_HOST: 127.0.0.1
      DATABASE_PORT: 1433
      DATABASE_USER: sa
      DATABASE_PASSWORD: YourStrong(!)Password
      DATABASE_NAME: vitract_kit_test
      REDIS_HOST: 127.0.0.1
      REDIS_PORT: 6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn test:int
```

> If your app requires extra env vars at bootstrap, add dummy values for test, and mock those providers.

---

## H) Worker split preparation (later, but design tests to not fight it)

When you implement worker split:

* API mode: imports controllers/services, registers queues, does **not** register processors
* Worker mode: registers processors, does **not** start HTTP

Add env flags:

* `HTTP_MODE=true/false`
* `WORKER_MODE=true/false`
* `ENABLE_SCHEDULER=true/false`

Testing becomes:

* Unit tests unchanged
* Integration tests can target API mode or worker mode separately
* E2E tests become cleaner (API process only enqueues)

---

## I) Troubleshooting guide

### I1) MSSQL connection failures

* Verify ports in `.env.test`
* Ensure password meets MSSQL complexity rules
* Ensure `trustServerCertificate` in test mode

### I2) Migrations not found

* Ensure migrations paths point to TS in test (if using ts-jest)
* If you only have compiled JS migrations, run integration tests against `dist/` artifacts (less ideal)

### I3) Flaky tests due to cron/jobs

* Confirm `ENABLE_SCHEDULER=false`
* Ensure no processors start consuming jobs automatically during HTTP integration tests
* Prefer mocking queue processing

### I4) Slow cleanup between tests

* Centralize cleanup
* Delete only touched tables first
* Expand cleanup as coverage grows

---

## J) Final deliverable checklist (the “ship it” list)

* [ ] docker-compose.test.yml
* [ ] `.env.test.example`
* [ ] `test/helpers/*` created
* [ ] `test:unit`, `test:int`, `test:e2e` scripts
* [ ] scheduler disabled in tests
* [ ] 15+ meaningful unit tests across billing/payment/health/mail
* [ ] 3–5 integration tests (migrations + mail tx + 1–2 HTTP flows)
* [ ] CI runs unit + integration on PR
* [ ] initial coverage thresholds enforced + ratchet plan documented

```

If you want, I can also generate an **Augment execution prompt** that tells it exactly which files to touch (≤12–15) to implement **Phase 1 foundations** with minimal drift (docker-compose.test.yml + helpers + scripts + scheduler flag).
```
