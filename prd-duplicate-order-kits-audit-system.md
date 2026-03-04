# PRD: Duplicate Order Kits Audit System

**Version:** 1.1  
**Date:** 2026-03-04  
**Status:** Draft  
**Author:** System Architecture Team

---

## 1. Executive Summary

This PRD outlines the design and implementation of a comprehensive audit system for tracking duplicate order kits sourced from a backup database. The system creates two separate audit tables in the main database, establishes a dedicated read-only backup database connection for reading backup `order-kits`, and provides CLI-based import and cleanup mechanisms.

### Goals
- Create audit trail for all duplicate order kits sourced from backup data
- Isolate special duplicate kits for specific order IDs
- Enable safe, reversible cleanup operations
- Maintain backward compatibility with existing systems
- Support migration-assisted setup with standalone CLI execution

### Quick Start Summary

**What gets created:**
1. Two new audit tables in the main DB: `duplicate_order_kits` and `special_duplicate_order_kits`
2. Two TypeORM entities with full audit-trail support
3. Separate backup database configuration used only to read backup `order-kits`
4. Six migration files (table creation + operation instructions)
5. Import and cleanup services with dry-run support
6. CLI commands for standalone execution

**How to use:**
```bash
# 1. Run migrations to create tables
yarn migration:run

# 2. Import duplicates from backup
yarn start:cli import:duplicate-order-kits --all
yarn start:cli import:duplicate-order-kits --special

# 3. Preview cleanup (dry-run)
yarn start:cli cleanup:duplicate-order-kits --special --dry-run
yarn start:cli cleanup:duplicate-order-kits --all --dry-run

# 4. Execute cleanup
yarn start:cli cleanup:duplicate-order-kits --special
yarn start:cli cleanup:duplicate-order-kits --all --confirm  # DESTRUCTIVE
```

**Key Features:**
- ✅ Backward compatible - no breaking changes
- ✅ Standalone execution - works without automatic orchestration
- ✅ Dry-run mode - preview before cleanup
- ✅ Audit trail - full history of all operations
- ✅ Transaction safety - automatic rollback on errors
- ✅ Mirror-copy semantics - imported source fields preserve original values, including `id`

---

## 2. Background & Context

### Current State
- The system uses SQL Server (MSSQL) as the primary database
- TypeORM manages entities and migrations
- Migration files are located in `src/common/migrations/`
- Database configuration is centralized in `src/config/db.ts`
- CLI command patterns already exist in the codebase

### Problem Statement
Duplicate order kits exist in a backup database and need to be:
1. Audited and tracked in the main database
2. Categorized (general duplicates vs. special order-specific duplicates)
3. Cleaned up in a controlled, reversible manner
4. Processed without affecting production operations

### Special Order IDs
- `7D01E25F-961C-F011-8B3D-6045BD8068CA`
- `750A2CC5-0A22-F011-8B3D-6045BD8068CA`

---

## 3. Technical Architecture

### 3.1 Database Schema

#### Table 1: `duplicate_order_kits`
Stores main-database mirror copies of duplicate order-kit rows sourced from the backup `order-kits` table.

```sql
CREATE TABLE duplicate_order_kits (
  id UNIQUEIDENTIFIER PRIMARY KEY, -- copied from backup row
  createdAt DATETIME NOT NULL,     -- copied from backup row
  updatedAt DATETIME NOT NULL,     -- copied from backup row

  -- Original data from backup
  orderId UNIQUEIDENTIFIER NOT NULL,
  kitId VARCHAR(255) NOT NULL,
  registrationStatus VARCHAR(50),
  registeredBy VARCHAR(50),
  registeredByUserId UNIQUEIDENTIFIER,

  -- Audit metadata
  backupSourceTimestamp DATETIME,
  importedAt DATETIME NOT NULL DEFAULT GETUTCDATE(),
  cleanupStatus VARCHAR(50) DEFAULT 'pending', -- pending, cleaned, skipped
  cleanedAt DATETIME,
  cleanupJobId VARCHAR(255),
  notes NVARCHAR(MAX),

  INDEX idx_duplicate_order_kits_order_id (orderId),
  INDEX idx_duplicate_order_kits_kit_id (kitId),
  INDEX idx_duplicate_order_kits_cleanup_status (cleanupStatus)
);
```

#### Table 2: `special_duplicate_order_kits`
Stores mirror copies of duplicate kits specific to the two special order IDs.

```sql
CREATE TABLE special_duplicate_order_kits (
  id UNIQUEIDENTIFIER PRIMARY KEY, -- copied from backup row
  createdAt DATETIME NOT NULL,     -- copied from backup row
  updatedAt DATETIME NOT NULL,     -- copied from backup row

  -- Original data from backup
  orderId UNIQUEIDENTIFIER NOT NULL,
  kitId VARCHAR(255) NOT NULL,
  registrationStatus VARCHAR(50),
  registeredBy VARCHAR(50),
  registeredByUserId UNIQUEIDENTIFIER,

  -- Audit metadata
  backupSourceTimestamp DATETIME,
  importedAt DATETIME NOT NULL DEFAULT GETUTCDATE(),
  cleanupStatus VARCHAR(50) DEFAULT 'pending',
  cleanedAt DATETIME,
  cleanupJobId VARCHAR(255),
  specialOrderReason NVARCHAR(500),
  notes NVARCHAR(MAX),

  INDEX idx_special_duplicate_order_kits_order_id (orderId),
  INDEX idx_special_duplicate_order_kits_kit_id (kitId),
  INDEX idx_special_duplicate_order_kits_cleanup_status (cleanupStatus),

  CONSTRAINT chk_special_order_ids CHECK (
    orderId IN (
      '7D01E25F-961C-F011-8B3D-6045BD8068CA',
      '750A2CC5-0A22-F011-8B3D-6045BD8068CA'
    )
  )
);
```

### 3.2 Entity Design

#### Entity 1: `DuplicateOrderKit`
Location: `src/audit/entity/duplicate-order-kit.entity.ts`

```typescript
import { Entity, Column, Index, PrimaryColumn } from 'typeorm';

@Entity('duplicate_order_kits')
@Index('idx_duplicate_order_kits_order_id', ['orderId'])
@Index('idx_duplicate_order_kits_kit_id', ['kitId'])
@Index('idx_duplicate_order_kits_cleanup_status', ['cleanupStatus'])
export class DuplicateOrderKit {
  @PrimaryColumn({ type: 'uuid' })
  id: string; // mirror of backup row ID

  @Column({ type: 'datetime' })
  createdAt: Date; // mirror of backup row value

  @Column({ type: 'datetime' })
  updatedAt: Date; // mirror of backup row value

  @Column({ type: 'uuid' })
  orderId: string;

  @Column({ type: 'varchar', length: 255 })
  kitId: string;

  @Column({ type: 'varchar', length: 50, nullable: true })
  registrationStatus: string | null;

  @Column({ type: 'varchar', length: 50, nullable: true })
  registeredBy: string | null;

  @Column({ type: 'uuid', nullable: true })
  registeredByUserId: string | null;

  @Column({ type: 'datetime', nullable: true })
  backupSourceTimestamp: Date | null;

  @Column({ type: 'datetime', default: () => 'GETUTCDATE()' })
  importedAt: Date;

  @Column({ type: 'varchar', length: 50, default: 'pending' })
  cleanupStatus: string;

  @Column({ type: 'datetime', nullable: true })
  cleanedAt: Date | null;

  @Column({ type: 'varchar', length: 255, nullable: true })
  cleanupJobId: string | null;

  @Column({ type: 'nvarchar', length: 'max', nullable: true })
  notes: string | null;
}
```

#### Entity 2: `SpecialDuplicateOrderKit`
Location: `src/audit/entity/special-duplicate-order-kit.entity.ts`

```typescript
import { Entity, Column, Index, PrimaryColumn } from 'typeorm';

@Entity('special_duplicate_order_kits')
@Index('idx_special_duplicate_order_kits_order_id', ['orderId'])
@Index('idx_special_duplicate_order_kits_kit_id', ['kitId'])
@Index('idx_special_duplicate_order_kits_cleanup_status', ['cleanupStatus'])
export class SpecialDuplicateOrderKit {
  @PrimaryColumn({ type: 'uuid' })
  id: string; // mirror of backup row ID

  @Column({ type: 'datetime' })
  createdAt: Date; // mirror of backup row value

  @Column({ type: 'datetime' })
  updatedAt: Date; // mirror of backup row value

  @Column({ type: 'uuid' })
  orderId: string;

  @Column({ type: 'varchar', length: 255 })
  kitId: string;

  @Column({ type: 'varchar', length: 50, nullable: true })
  registrationStatus: string | null;

  @Column({ type: 'varchar', length: 50, nullable: true })
  registeredBy: string | null;

  @Column({ type: 'uuid', nullable: true })
  registeredByUserId: string | null;

  @Column({ type: 'datetime', nullable: true })
  backupSourceTimestamp: Date | null;

  @Column({ type: 'datetime', default: () => 'GETUTCDATE()' })
  importedAt: Date;

  @Column({ type: 'varchar', length: 50, default: 'pending' })
  cleanupStatus: string;

  @Column({ type: 'datetime', nullable: true })
  cleanedAt: Date | null;

  @Column({ type: 'varchar', length: 255, nullable: true })
  cleanupJobId: string | null;

  @Column({ type: 'nvarchar', length: 500, nullable: true })
  specialOrderReason: string | null;

  @Column({ type: 'nvarchar', length: 'max', nullable: true })
  notes: string | null;
}
```

### 3.3 Backup Database Configuration

#### New Configuration Class
Location: `src/config/backup-db.ts`

```typescript
import { TypeOrmModuleOptions } from '@nestjs/typeorm';
import {
  BACKUP_DATABASE_URL,
  NODE_ENV,
} from './keys';
import splitPostgresConnectionString from 'src/common/utils/get-db-configs';

const parsedBackupDatabase = BACKUP_DATABASE_URL
  ? splitPostgresConnectionString(BACKUP_DATABASE_URL)
  : null;

class BackupDbConfig {
  public getTypeOrmConfig(): TypeOrmModuleOptions {
    if (!BACKUP_DATABASE_URL || !parsedBackupDatabase) {
      throw new Error('Backup database configuration is incomplete');
    }

    const port = Number(parsedBackupDatabase.port) || 1433;

    return {
      type: 'mssql',
      host: parsedBackupDatabase.host,
      username: parsedBackupDatabase.username,
      password: parsedBackupDatabase.password,
      database: parsedBackupDatabase.database,
      port,
      synchronize: false,
      options: {
        encrypt: true,
        trustServerCertificate: NODE_ENV !== 'production',
        enableArithAbort: true,
      },
      extra: {
        encrypt: true,
        trustServerCertificate: NODE_ENV !== 'production',
        enableArithAbort: true,
        connectTimeout: 60_000,
        requestTimeout: 60_000,
        pool: {
          max: 5,
          min: 0,
          idleTimeoutMillis: 30_000,
        },
      },
      retryAttempts: 3,
      retryDelay: 2000,
    };
  }
}
```

#### Environment Variables
Add to `.env`:

```bash
# Backup Database Configuration
BACKUP_DATABASE_URL=mssql://backup_user:secure_password@backup-db-server.example.com:1433/vitract_backup
```

Important rules:
- Backup DB credentials must be read-only
- The backup DB connection exists only to read backup `order-kits`
- Audit entities and audit tables exist only in the main database

---

## 4. Migration Strategy

### 4.1 Migration Files Structure

All migrations will be created as separate files in `src/common/migrations/`:

1. **Migration 1**: Create `duplicate_order_kits` table
2. **Migration 2**: Create `special_duplicate_order_kits` table
3. **Migration 3**: Import all duplicates from backup (prints CLI instructions)
4. **Migration 4**: Import special duplicates from backup (prints CLI instructions)
5. **Migration 5**: Cleanup special order duplicates (prints CLI instructions)
6. **Migration 6**: Cleanup all duplicates (DESTRUCTIVE - prints CLI instructions)

### 4.2 Migration 1: Create `duplicate_order_kits` Table

Location: `src/common/migrations/1772000000001-create-duplicate-order-kits-table.ts`

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class CreateDuplicateOrderKitsTable1772000000001
  implements MigrationInterface
{
  name = 'CreateDuplicateOrderKitsTable1772000000001';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TABLE "duplicate_order_kits" (
        "id" uniqueidentifier NOT NULL,
        "createdAt" datetime NOT NULL,
        "updatedAt" datetime NOT NULL,
        "orderId" uniqueidentifier NOT NULL,
        "kitId" varchar(255) NOT NULL,
        "registrationStatus" varchar(50),
        "registeredBy" varchar(50),
        "registeredByUserId" uniqueidentifier,
        "backupSourceTimestamp" datetime,
        "importedAt" datetime NOT NULL
          CONSTRAINT "DF_duplicate_order_kits_importedAt" DEFAULT GETUTCDATE(),
        "cleanupStatus" varchar(50) NOT NULL
          CONSTRAINT "DF_duplicate_order_kits_cleanupStatus" DEFAULT 'pending',
        "cleanedAt" datetime,
        "cleanupJobId" varchar(255),
        "notes" nvarchar(max),
        CONSTRAINT "PK_duplicate_order_kits" PRIMARY KEY ("id")
      )
    `);

    await queryRunner.query(`
      CREATE INDEX "idx_duplicate_order_kits_order_id"
      ON "duplicate_order_kits" ("orderId")
    `);

    await queryRunner.query(`
      CREATE INDEX "idx_duplicate_order_kits_kit_id"
      ON "duplicate_order_kits" ("kitId")
    `);

    await queryRunner.query(`
      CREATE INDEX "idx_duplicate_order_kits_cleanup_status"
      ON "duplicate_order_kits" ("cleanupStatus")
    `);
  }
}
```

### 4.3 Migration 2: Create `special_duplicate_order_kits` Table

Location: `src/common/migrations/1772000000002-create-special-duplicate-order-kits-table.ts`

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class CreateSpecialDuplicateOrderKitsTable1772000000002
  implements MigrationInterface
{
  name = 'CreateSpecialDuplicateOrderKitsTable1772000000002';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TABLE "special_duplicate_order_kits" (
        "id" uniqueidentifier NOT NULL,
        "createdAt" datetime NOT NULL,
        "updatedAt" datetime NOT NULL,
        "orderId" uniqueidentifier NOT NULL,
        "kitId" varchar(255) NOT NULL,
        "registrationStatus" varchar(50),
        "registeredBy" varchar(50),
        "registeredByUserId" uniqueidentifier,
        "backupSourceTimestamp" datetime,
        "importedAt" datetime NOT NULL
          CONSTRAINT "DF_special_duplicate_order_kits_importedAt" DEFAULT GETUTCDATE(),
        "cleanupStatus" varchar(50) NOT NULL
          CONSTRAINT "DF_special_duplicate_order_kits_cleanupStatus" DEFAULT 'pending',
        "cleanedAt" datetime,
        "cleanupJobId" varchar(255),
        "specialOrderReason" nvarchar(500),
        "notes" nvarchar(max),
        CONSTRAINT "PK_special_duplicate_order_kits" PRIMARY KEY ("id"),
        CONSTRAINT "CHK_special_order_ids" CHECK (
          "orderId" IN (
            '7D01E25F-961C-F011-8B3D-6045BD8068CA',
            '750A2CC5-0A22-F011-8B3D-6045BD8068CA'
          )
        )
      )
    `);
  }
}
```

### 4.4 Migration 3: Import All Duplicates

Location: `src/common/migrations/1772000000003-import-all-duplicate-order-kits.ts`

This migration does not import data automatically. It prints the correct operator command:

```bash
yarn start:cli import:duplicate-order-kits --all
```

### 4.5 Migration 4: Import Special Duplicates

Location: `src/common/migrations/1772000000004-import-special-duplicate-order-kits.ts`

This migration prints:

```bash
yarn start:cli import:duplicate-order-kits --special
```

### 4.6 Migration 5: Cleanup Special Order Duplicates

Location: `src/common/migrations/1772000000005-cleanup-special-order-duplicates.ts`

This migration prints:

```bash
yarn start:cli cleanup:duplicate-order-kits --special --dry-run
yarn start:cli cleanup:duplicate-order-kits --special
```

### 4.7 Migration 6: Cleanup All Duplicates (DESTRUCTIVE)

Location: `src/common/migrations/1772000000006-cleanup-all-duplicates.ts`

This migration prints:

```bash
yarn start:cli cleanup:duplicate-order-kits --all --dry-run
yarn start:cli cleanup:duplicate-order-kits --all --confirm
```

---

## 5. Service Layer Implementation

### 5.1 Import Service

Location: `src/audit/services/import-duplicate-order-kits.service.ts`

**Responsibilities:**
- Connect to backup database
- Read duplicate source rows from backup `order-kits`
- Import mirror copies into the main database audit tables
- Handle batch processing for large datasets
- Provide progress reporting

**Key Methods:**
```typescript
class ImportDuplicateOrderKitsService {
  async importAll(options: ImportOptions): Promise<ImportResult>
  async importSpecial(options: ImportOptions): Promise<ImportResult>
  private async connectToBackupDb(): Promise<DataSource>
  private async batchImport(records: any[], batchSize: number): Promise<void>
}
```

### 5.2 Cleanup Service

Location: `src/audit/services/cleanup-duplicate-order-kits.service.ts`

**Responsibilities:**
- Identify duplicate records to clean
- Perform dry-run analysis
- Execute cleanup operations
- Update audit trail
- Generate cleanup reports

**Key Methods:**
```typescript
class CleanupDuplicateOrderKitsService {
  async cleanupSpecial(options: CleanupOptions): Promise<CleanupResult>
  async cleanupAll(options: CleanupOptions): Promise<CleanupResult>
  async dryRun(scope: 'special' | 'all'): Promise<DryRunResult>
  private async markAsCleaned(ids: string[], jobId: string): Promise<void>
}
```

### 5.3 Execution Rules

- Import and cleanup operations are CLI-only
- No queue job types are required for this feature
- No queue processor is required for this feature
- Long-running operations rely on batch size, transaction control, and structured CLI output

---

## 6. CLI Commands (Standalone Execution)

### 6.1 Import Command

Location: `src/cli/commands/import-duplicate-order-kits.command.ts`

```typescript
interface ImportCommandOptions {
  all?: boolean;
  special?: boolean;
  batchSize?: number;
}
```

Rules:
- Must specify exactly one of `--all` or `--special`
- Supports `--batch-size <size>`
- Does not support `--background`

### 6.2 Cleanup Command

Location: `src/cli/commands/cleanup-duplicate-order-kits.command.ts`

```typescript
interface CleanupCommandOptions {
  all?: boolean;
  special?: boolean;
  dryRun?: boolean;
  confirm?: boolean;
}
```

Rules:
- Must specify exactly one of `--all` or `--special`
- `--dry-run` previews mutation
- `--confirm` is required for `--all`

---

## 7. Module Structure

### 7.1 Audit Module

Location: `src/audit/audit.module.ts`

Responsibilities:
- Register audit entities against the main DB
- Export services for CLI execution

### 7.2 Update App Module

The application should register the audit module without changing existing business flows.

### 7.3 Update CLI Command Module

CLI command registration is required for:
- `import:duplicate-order-kits`
- `cleanup:duplicate-order-kits`

No queue processor module changes are required for this feature.

---

## 8. Backward Compatibility

### 8.1 Design Principles

1. **Non-Breaking Additions Only**: New tables, entities, and commands are additive
2. **No Existing Code Changes**: Current API and business logic remain untouched
3. **Graceful Degradation**: The application continues to work when backup DB is not configured, unless backup-specific CLI commands are invoked

### 8.2 Environment Variable Guards

```typescript
if (!BACKUP_DATABASE_URL) {
  return { skipped: true, reason: 'No backup database configured' };
}
```

### 8.3 Rollback Strategy

- Table creation migrations can be reverted
- Cleanup metadata can be reset where safe
- Operators must review dry-run output before destructive runs

---

## 9. Execution Modes

### 9.1 Migration-Driven Execution

```bash
yarn migration:run
```

This creates schema and prints operator instructions.

### 9.2 Standalone CLI Execution

```bash
yarn start:cli import:duplicate-order-kits --all
yarn start:cli import:duplicate-order-kits --special
yarn start:cli cleanup:duplicate-order-kits --special --dry-run
yarn start:cli cleanup:duplicate-order-kits --special
```

There is no background-job execution mode for this feature.

---

## 10. Safety Mechanisms

### 10.1 Dry-Run Mode

- All cleanup paths must support dry-run preview

### 10.2 Confirmation Requirements

- `cleanup --all` requires `--confirm`

### 10.3 Audit Trail

- Imported audit rows persist in main DB
- Cleanup actions update audit metadata

### 10.4 Transaction Safety

- Batch imports should be transactional where practical
- Partial failures must be explicit and observable

---

## 11. Monitoring & Observability

### 11.1 Logging

Structured logs should include:
- Command name
- Scope
- Batch size
- Duration
- Processed/imported/cleaned/skipped counts

### 11.2 Metrics

Useful operator signals:
- CLI run duration
- Batch throughput
- Import counts
- Cleanup candidate counts
- Cleanup final counts

### 11.3 Operational Visibility

This feature does not depend on Bull Board or queue monitoring.
Operational visibility comes from CLI output, application logs, and direct database queries.

---

## 12. Testing Strategy

### 12.1 Unit Tests

- Backup database configuration parsing
- Import service mapping logic
- Cleanup confirmation behavior
- CLI parsing and validation

### 12.2 Integration Tests

- Import into `duplicate_order_kits`
- Import into `special_duplicate_order_kits`
- Mirror-copy preservation for source fields, including `id`
- Dry-run and destructive cleanup flows
- Migration registration and execution

### 12.3 E2E Tests

- Production-like CLI execution against MSSQL

---

## 13. Implementation Phases

### Phase 1: Foundation (Week 1)
- [ ] Create entity files
- [ ] Create backup DB configuration
- [ ] Create migration files (1-2)
- [ ] Update environment variables
- [ ] Write unit tests for entities

### Phase 2: Import System (Week 2)
- [ ] Implement import service
- [ ] Create import CLI command
- [ ] Create migration files (3-4)
- [ ] Write integration tests for import

### Phase 3: Cleanup System (Week 3)
- [ ] Implement cleanup service
- [ ] Create cleanup CLI command
- [ ] Create migration files (5-6)
- [ ] Write integration tests for cleanup

### Phase 4: Testing & Documentation (Week 4)
- [ ] Complete E2E tests
- [ ] Performance testing with large datasets
- [ ] Update API documentation
- [ ] Create runbook for operations
- [ ] Conduct code review

### Phase 5: Deployment (Week 5)
- [ ] Deploy to staging environment
- [ ] Run test imports
- [ ] Validate dry-run functionality
- [ ] Deploy to production
- [ ] Monitor initial operations

---

## 14. Operational Runbook

### 14.1 Initial Setup

```bash
export BACKUP_DATABASE_URL=mssql://backup-user:secure-password@backup-server.example.com:1433/vitract_backup
yarn migration:run
```

### 14.2 Import Duplicates

```bash
yarn start:cli import:duplicate-order-kits --all
yarn start:cli import:duplicate-order-kits --special
```

### 14.3 Cleanup Operations

```bash
yarn start:cli cleanup:duplicate-order-kits --special --dry-run
yarn start:cli cleanup:duplicate-order-kits --special
yarn start:cli cleanup:duplicate-order-kits --all --dry-run
yarn start:cli cleanup:duplicate-order-kits --all --confirm
```

### 14.4 Monitoring

```sql
SELECT cleanupStatus, COUNT(*)
FROM duplicate_order_kits
GROUP BY cleanupStatus;

SELECT * FROM special_duplicate_order_kits
WHERE cleanupStatus = 'pending';
```

### 14.5 Troubleshooting

**Issue**: Import fails with connection timeout

```bash
yarn start:cli import:duplicate-order-kits --all --batch-size 500
```

**Issue**: Cleanup dry-run shows unexpected results

```sql
SELECT TOP 100 *
FROM duplicate_order_kits
WHERE cleanupStatus = 'pending';
```

**Issue**: Migration rollback needed

```bash
yarn migration:revert
```

---

## 15. Security Considerations

### 15.1 Database Access

- Backup database credentials stored in environment variables
- Use read-only credentials for backup DB connection
- Implement connection pooling limits
- Enable SSL/TLS for backup DB connections

### 15.2 Data Privacy

- Audit tables contain PII-sensitive operational identifiers
- Apply same access controls as main database
- Consider data retention policies

### 15.3 Operation Authorization

- Cleanup operations require admin/operator privileges
- Log all cleanup operations with user context where available
- Require confirmation for destructive operations

---

## 16. Performance Considerations

### 16.1 Batch Processing

- Default batch size: 1000 records
- Configurable via CLI: `--batch-size <n>`
- Process in transactions to prevent partial imports
- Use bulk insert operations where safe

### 16.2 Indexing Strategy

Indexes created for optimal query performance:
- `orderId` - for filtering by order
- `kitId` - for kit lookups
- `cleanupStatus` - for cleanup queries

### 16.3 Connection Pooling

Backup database connection pool:
- Max connections: 5
- Min connections: 0
- Idle timeout: 30 seconds
- Connection timeout: 60 seconds

### 16.4 Large Dataset Handling

For datasets > 100K records:
- Use CLI batching with explicit operator control
- Implement progress tracking
- Consider chunking by date ranges
- Monitor memory usage

---

## 17. Success Criteria

### 17.1 Functional Requirements

- ✅ Two audit tables created successfully in the main DB
- ✅ Backup database connection isolated and working
- ✅ Import all duplicates from backup `order-kits`
- ✅ Import special duplicates for specific orders
- ✅ Cleanup special order duplicates
- ✅ Cleanup all duplicates (destructive)
- ✅ All operations reversible via migrations where applicable

### 17.2 Non-Functional Requirements

- ✅ Backward compatible with existing system
- ✅ Can run standalone without migrations
- ✅ Dry-run mode for all cleanup operations
- ✅ Comprehensive audit trail
- ✅ Performance: Process 10K records in < 5 minutes
- ✅ Error handling with automatic rollback

### 17.3 Documentation Requirements

- ✅ PRD document
- ✅ API documentation for services
- ✅ Operational runbook
- ✅ Migration guide
- ✅ Troubleshooting guide

---

## 18. Future Enhancements

### 18.1 Potential Improvements

1. **Web UI for Cleanup**
   - Admin dashboard for viewing duplicates
   - Visual dry-run results
   - One-click cleanup with confirmation

2. **Automated Scheduling**
   - Cron-based CLI orchestration outside the core feature
   - Email notifications for cleanup recommendations

3. **Advanced Analytics**
   - Duplicate-rate reporting by order cohort
   - Cleanup trend analysis

4. **Multi-Source Support**
   - Support multiple backup databases
   - Configurable source table mappings

---

## 19. Risks & Mitigation

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Wrong backup source selected | High | Medium | Explicit `BACKUP_DATABASE_URL`, read-only credentials, validation queries |
| Mirror-copy drift | High | Medium | Contract tests on source fields, especially `id` |
| Unsafe destructive cleanup | High | Low | Dry-run-first flow, `--confirm`, logs |
| Performance degradation | Medium | Low | Batch processing, CLI execution, connection pooling |

---

## 20. Appendix

### 20.1 Special Order IDs Reference

- `7D01E25F-961C-F011-8B3D-6045BD8068CA`
- `750A2CC5-0A22-F011-8B3D-6045BD8068CA`

### 20.2 Cleanup Status Enum

```typescript
enum CleanupStatus {
  PENDING = 'pending',
  CLEANED = 'cleaned',
  SKIPPED = 'skipped',
}
```

### 20.3 Sample Queries

```sql
SELECT COUNT(*) FROM duplicate_order_kits;
SELECT COUNT(*) FROM special_duplicate_order_kits;

SELECT TOP 50 *
FROM duplicate_order_kits
WHERE cleanupStatus = 'pending';
```

---

## 21. Glossary

- **Mirror Copy**: A main-database audit row whose source columns exactly match the backup `order-kits` row, including `id`
- **Dry Run**: A non-mutating analysis of cleanup impact
- **Special Duplicate**: A duplicate tied to one of the two special order IDs

---

## 22. References

- Existing project CLI patterns
- TypeORM MSSQL configuration
- Internal duplicate-order-kit investigation artifacts
