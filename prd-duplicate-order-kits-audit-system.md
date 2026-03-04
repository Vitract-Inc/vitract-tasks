# PRD: Duplicate Order Kits Audit System

**Version:** 1.0  
**Date:** 2026-03-04  
**Status:** Draft  
**Author:** System Architecture Team

---

## 1. Executive Summary

This PRD outlines the design and implementation of a comprehensive audit system for tracking duplicate order kits from a backup database. The system will create two separate audit tables with their entities, establish a dedicated backup database connection, and provide both automated migration-based and standalone script-based cleanup mechanisms.

### Goals
- Create audit trail for all duplicate order kits from backup database
- Isolate special duplicate kits for specific order IDs
- Enable safe, reversible cleanup operations
- Maintain backward compatibility with existing systems
- Support both migration-driven and standalone execution modes

### Quick Start Summary

**What gets created:**
1. Two new audit tables: `duplicate_order_kits` and `special_duplicate_order_kits`
2. Two TypeORM entities with full audit trail support
3. Separate backup database configuration (isolated from main DB)
4. Six migration files (table creation + operation instructions)
5. Import and cleanup services with dry-run support
6. CLI commands for standalone execution
7. Background job processors for async operations

**How to use:**
```bash
# 1. Run migrations to create tables
yarn migration:run

# 2. Import duplicates from backup
yarn start:cli import:duplicate-order-kits --all
yarn start:cli import:duplicate-order-kits --special

# 3. Preview cleanup (dry-run)
yarn start:cli cleanup:duplicate-order-kits --special --dry-run

# 4. Execute cleanup
yarn start:cli cleanup:duplicate-order-kits --special
yarn start:cli cleanup:duplicate-order-kits --all --confirm  # DESTRUCTIVE
```

**Key Features:**
- ✅ Backward compatible - no breaking changes
- ✅ Standalone execution - works without migrations
- ✅ Dry-run mode - preview before cleanup
- ✅ Audit trail - full history of all operations
- ✅ Transaction safety - automatic rollback on errors
- ✅ Background jobs - async processing for large datasets

---

## 2. Background & Context

### Current State
- The system uses SQL Server (MSSQL) as the primary database
- TypeORM manages entities and migrations
- Migration files are located in `src/common/migrations/`
- Background jobs use Bull queues with Redis
- Database configuration is centralized in `src/config/db.ts`

### Problem Statement
Duplicate order kits exist in a backup database that need to be:
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
Stores all duplicate order kits from backup database.

```sql
CREATE TABLE duplicate_order_kits (
  id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
  createdAt DATETIME NOT NULL DEFAULT GETUTCDATE(),
  updatedAt DATETIME NOT NULL DEFAULT GETUTCDATE(),
  
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
Stores duplicate kits specific to the two special order IDs.

```sql
CREATE TABLE special_duplicate_order_kits (
  id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
  createdAt DATETIME NOT NULL DEFAULT GETUTCDATE(),
  updatedAt DATETIME NOT NULL DEFAULT GETUTCDATE(),
  
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
  specialOrderReason NVARCHAR(500), -- Why this order is special
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
import { BaseEntity } from 'src/common';
import { Entity, Column, Index } from 'typeorm';

@Entity('duplicate_order_kits')
@Index('idx_duplicate_order_kits_order_id', ['orderId'])
@Index('idx_duplicate_order_kits_kit_id', ['kitId'])
@Index('idx_duplicate_order_kits_cleanup_status', ['cleanupStatus'])
export class DuplicateOrderKit extends BaseEntity {
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
import { BaseEntity } from 'src/common';
import { Entity, Column, Index } from 'typeorm';

@Entity('special_duplicate_order_kits')
@Index('idx_special_duplicate_order_kits_order_id', ['orderId'])
@Index('idx_special_duplicate_order_kits_kit_id', ['kitId'])
@Index('idx_special_duplicate_order_kits_cleanup_status', ['cleanupStatus'])
export class SpecialDuplicateOrderKit extends BaseEntity {
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
import { NODE_ENV } from './keys';
import * as dotenv from 'dotenv';

dotenv.config();

const {
  BACKUP_DATABASE_HOST,
  BACKUP_DATABASE_PORT,
  BACKUP_DATABASE_USER,
  BACKUP_DATABASE_PASSWORD,
  BACKUP_DATABASE_NAME,
} = process.env;

class BackupDbConfig {
  public getTypeOrmConfig(): TypeOrmModuleOptions {
    if (!BACKUP_DATABASE_HOST || !BACKUP_DATABASE_NAME) {
      throw new Error('Backup database configuration is incomplete');
    }

    const port = Number(BACKUP_DATABASE_PORT) || 1433;

    return {
      type: 'mssql',
      host: BACKUP_DATABASE_HOST,
      username: BACKUP_DATABASE_USER,
      password: BACKUP_DATABASE_PASSWORD,
      database: BACKUP_DATABASE_NAME,
      port,
      synchronize: false,
      // Only include audit entities for backup DB
      entities: [
        'dist/audit/entity/duplicate-order-kit.entity.js',
        'dist/audit/entity/special-duplicate-order-kit.entity.js',
      ],
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

const backupDbConfig = new BackupDbConfig();
export { backupDbConfig };
```

#### Environment Variables
Add to `.env`:

```bash
# Backup Database Configuration
BACKUP_DATABASE_HOST=backup-db-server.example.com
BACKUP_DATABASE_PORT=1433
BACKUP_DATABASE_USER=backup_user
BACKUP_DATABASE_PASSWORD=secure_password
BACKUP_DATABASE_NAME=vitract_backup
```

---

## 4. Migration Strategy

### 4.1 Migration Files Structure

All migrations will be created as separate files in `src/common/migrations/`:

1. **Migration 1**: Create `duplicate_order_kits` table
2. **Migration 2**: Create `special_duplicate_order_kits` table
3. **Migration 3**: Import all duplicates from backup (runs background job)
4. **Migration 4**: Import special duplicates from backup (runs background job)
5. **Migration 5**: Cleanup special order duplicates (runs background job)
6. **Migration 6**: Cleanup all duplicates (DESTRUCTIVE - runs background job)

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
        "id" uniqueidentifier NOT NULL
          CONSTRAINT "DF_duplicate_order_kits_id" DEFAULT NEWSEQUENTIALID(),
        "createdAt" datetime NOT NULL
          CONSTRAINT "DF_duplicate_order_kits_createdAt" DEFAULT GETUTCDATE(),
        "updatedAt" datetime NOT NULL
          CONSTRAINT "DF_duplicate_order_kits_updatedAt" DEFAULT GETUTCDATE(),
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

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP INDEX "idx_duplicate_order_kits_cleanup_status"
      ON "duplicate_order_kits"`);
    await queryRunner.query(`DROP INDEX "idx_duplicate_order_kits_kit_id"
      ON "duplicate_order_kits"`);
    await queryRunner.query(`DROP INDEX "idx_duplicate_order_kits_order_id"
      ON "duplicate_order_kits"`);
    await queryRunner.query(`DROP TABLE "duplicate_order_kits"`);
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
        "id" uniqueidentifier NOT NULL
          CONSTRAINT "DF_special_duplicate_order_kits_id" DEFAULT NEWSEQUENTIALID(),
        "createdAt" datetime NOT NULL
          CONSTRAINT "DF_special_duplicate_order_kits_createdAt" DEFAULT GETUTCDATE(),
        "updatedAt" datetime NOT NULL
          CONSTRAINT "DF_special_duplicate_order_kits_updatedAt" DEFAULT GETUTCDATE(),
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

    await queryRunner.query(`
      CREATE INDEX "idx_special_duplicate_order_kits_order_id"
      ON "special_duplicate_order_kits" ("orderId")
    `);

    await queryRunner.query(`
      CREATE INDEX "idx_special_duplicate_order_kits_kit_id"
      ON "special_duplicate_order_kits" ("kitId")
    `);

    await queryRunner.query(`
      CREATE INDEX "idx_special_duplicate_order_kits_cleanup_status"
      ON "special_duplicate_order_kits" ("cleanupStatus")
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP INDEX "idx_special_duplicate_order_kits_cleanup_status"
      ON "special_duplicate_order_kits"`);
    await queryRunner.query(`DROP INDEX "idx_special_duplicate_order_kits_kit_id"
      ON "special_duplicate_order_kits"`);
    await queryRunner.query(`DROP INDEX "idx_special_duplicate_order_kits_order_id"
      ON "special_duplicate_order_kits"`);
    await queryRunner.query(`DROP TABLE "special_duplicate_order_kits"`);
  }
}
```

### 4.4 Migration 3: Import All Duplicates (Background Job)

Location: `src/common/migrations/1772000000003-import-all-duplicate-order-kits.ts`

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class ImportAllDuplicateOrderKits1772000000003
  implements MigrationInterface
{
  name = 'ImportAllDuplicateOrderKits1772000000003';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // This migration triggers a background job to import duplicates
    // The actual import logic is in a service that can be run independently

    console.log('');
    console.log('='.repeat(80));
    console.log('MIGRATION: Import All Duplicate Order Kits');
    console.log('='.repeat(80));
    console.log('');
    console.log('This migration has created the necessary tables.');
    console.log('');
    console.log('To import duplicate order kits from the backup database, run:');
    console.log('');
    console.log('  yarn start:cli import:duplicate-order-kits --all');
    console.log('');
    console.log('Or to run as a background job:');
    console.log('');
    console.log('  yarn start:cli import:duplicate-order-kits --all --background');
    console.log('');
    console.log('='.repeat(80));
    console.log('');
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // Rollback: Delete all imported records
    await queryRunner.query(`DELETE FROM "duplicate_order_kits"`);
  }
}
```

### 4.5 Migration 4: Import Special Duplicates (Background Job)

Location: `src/common/migrations/1772000000004-import-special-duplicate-order-kits.ts`

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class ImportSpecialDuplicateOrderKits1772000000004
  implements MigrationInterface
{
  name = 'ImportSpecialDuplicateOrderKits1772000000004';

  public async up(queryRunner: QueryRunner): Promise<void> {
    console.log('');
    console.log('='.repeat(80));
    console.log('MIGRATION: Import Special Duplicate Order Kits');
    console.log('='.repeat(80));
    console.log('');
    console.log('To import special duplicate order kits for specific orders, run:');
    console.log('');
    console.log('  yarn start:cli import:duplicate-order-kits --special');
    console.log('');
    console.log('This will import duplicates for orders:');
    console.log('  - 7D01E25F-961C-F011-8B3D-6045BD8068CA');
    console.log('  - 750A2CC5-0A22-F011-8B3D-6045BD8068CA');
    console.log('');
    console.log('='.repeat(80));
    console.log('');
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DELETE FROM "special_duplicate_order_kits"`);
  }
}
```

### 4.6 Migration 5: Cleanup Special Order Duplicates

Location: `src/common/migrations/1772000000005-cleanup-special-order-duplicates.ts`

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class CleanupSpecialOrderDuplicates1772000000005
  implements MigrationInterface
{
  name = 'CleanupSpecialOrderDuplicates1772000000005';

  public async up(queryRunner: QueryRunner): Promise<void> {
    console.log('');
    console.log('='.repeat(80));
    console.log('MIGRATION: Cleanup Special Order Duplicates');
    console.log('='.repeat(80));
    console.log('');
    console.log('⚠️  WARNING: This operation will clean up duplicate order kits');
    console.log('for special orders. This is a DESTRUCTIVE operation.');
    console.log('');
    console.log('To proceed with cleanup, run:');
    console.log('');
    console.log('  yarn start:cli cleanup:duplicate-order-kits --special --dry-run');
    console.log('');
    console.log('Review the dry-run results, then execute:');
    console.log('');
    console.log('  yarn start:cli cleanup:duplicate-order-kits --special');
    console.log('');
    console.log('='.repeat(80));
    console.log('');
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // Rollback: Reset cleanup status
    await queryRunner.query(`
      UPDATE "special_duplicate_order_kits"
      SET "cleanupStatus" = 'pending',
          "cleanedAt" = NULL,
          "cleanupJobId" = NULL
      WHERE "cleanupStatus" = 'cleaned'
    `);
  }
}
```

### 4.7 Migration 6: Cleanup All Duplicates (DESTRUCTIVE)

Location: `src/common/migrations/1772000000006-cleanup-all-duplicates.ts`

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class CleanupAllDuplicates1772000000006 implements MigrationInterface {
  name = 'CleanupAllDuplicates1772000000006';

  public async up(queryRunner: QueryRunner): Promise<void> {
    console.log('');
    console.log('='.repeat(80));
    console.log('MIGRATION: Cleanup All Duplicate Order Kits');
    console.log('='.repeat(80));
    console.log('');
    console.log('⚠️  CRITICAL WARNING: This is a DESTRUCTIVE operation!');
    console.log('This will clean up ALL duplicate order kits from the system.');
    console.log('');
    console.log('ALWAYS run a dry-run first:');
    console.log('');
    console.log('  yarn start:cli cleanup:duplicate-order-kits --all --dry-run');
    console.log('');
    console.log('After reviewing the dry-run results and backing up data:');
    console.log('');
    console.log('  yarn start:cli cleanup:duplicate-order-kits --all --confirm');
    console.log('');
    console.log('='.repeat(80));
    console.log('');
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // Rollback: Reset cleanup status for all records
    await queryRunner.query(`
      UPDATE "duplicate_order_kits"
      SET "cleanupStatus" = 'pending',
          "cleanedAt" = NULL,
          "cleanupJobId" = NULL
      WHERE "cleanupStatus" = 'cleaned'
    `);
  }
}
```

---

## 5. Service Layer Implementation

### 5.1 Import Service

Location: `src/audit/services/import-duplicate-order-kits.service.ts`

**Responsibilities:**
- Connect to backup database
- Query duplicate order kits
- Import into appropriate audit tables
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

### 5.3 Queue Job Types

Location: `src/queues/types/queue.types.ts`

Add new job types:

```typescript
export enum JobTypes {
  // ... existing types
  IMPORT_DUPLICATE_ORDER_KITS = 'import-duplicate-order-kits',
  CLEANUP_DUPLICATE_ORDER_KITS = 'cleanup-duplicate-order-kits',
}

export interface ImportDuplicateOrderKitsJobData {
  scope: 'all' | 'special';
  batchSize?: number;
  startFrom?: number;
}

export interface CleanupDuplicateOrderKitsJobData {
  scope: 'all' | 'special';
  dryRun: boolean;
  confirmationToken?: string;
}
```

### 5.4 Queue Processor

Location: `src/queues/processors/duplicate-order-kits.processor.ts`

```typescript
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';
import { Injectable, Logger } from '@nestjs/common';
import { QueueNames, JobTypes } from '../types/queue.types';
import { ImportDuplicateOrderKitsService } from 'src/audit/services/import-duplicate-order-kits.service';
import { CleanupDuplicateOrderKitsService } from 'src/audit/services/cleanup-duplicate-order-kits.service';

@Injectable()
@Processor(QueueNames.AUDIT)
export class DuplicateOrderKitsProcessor {
  private readonly logger = new Logger(DuplicateOrderKitsProcessor.name);

  constructor(
    private readonly importService: ImportDuplicateOrderKitsService,
    private readonly cleanupService: CleanupDuplicateOrderKitsService,
  ) {}

  @Process({ name: JobTypes.IMPORT_DUPLICATE_ORDER_KITS, concurrency: 1 })
  async handleImport(job: Job<ImportDuplicateOrderKitsJobData>): Promise<any> {
    const { scope, batchSize } = job.data;

    this.logger.log(`Starting import of ${scope} duplicate order kits`);

    if (scope === 'all') {
      return await this.importService.importAll({ batchSize });
    } else {
      return await this.importService.importSpecial({ batchSize });
    }
  }

  @Process({ name: JobTypes.CLEANUP_DUPLICATE_ORDER_KITS, concurrency: 1 })
  async handleCleanup(job: Job<CleanupDuplicateOrderKitsJobData>): Promise<any> {
    const { scope, dryRun, confirmationToken } = job.data;

    this.logger.log(`Starting cleanup of ${scope} duplicates (dryRun: ${dryRun})`);

    if (dryRun) {
      return await this.cleanupService.dryRun(scope);
    }

    if (scope === 'all' && !confirmationToken) {
      throw new Error('Confirmation token required for cleanup all operation');
    }

    if (scope === 'special') {
      return await this.cleanupService.cleanupSpecial({ dryRun: false });
    } else {
      return await this.cleanupService.cleanupAll({
        dryRun: false,
        confirmationToken
      });
    }
  }
}
```

---

## 6. CLI Commands (Standalone Execution)

### 6.1 Import Command

Location: `src/cli/commands/import-duplicate-order-kits.command.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { Command, CommandRunner, Option } from 'nest-commander';
import { ImportDuplicateOrderKitsService } from 'src/audit/services/import-duplicate-order-kits.service';

interface ImportCommandOptions {
  all?: boolean;
  special?: boolean;
  background?: boolean;
  batchSize?: number;
}

@Injectable()
@Command({
  name: 'import:duplicate-order-kits',
  description: 'Import duplicate order kits from backup database',
})
export class ImportDuplicateOrderKitsCommand extends CommandRunner {
  constructor(
    private readonly importService: ImportDuplicateOrderKitsService,
  ) {
    super();
  }

  async run(
    passedParams: string[],
    options: ImportCommandOptions,
  ): Promise<void> {
    const scope = options.all ? 'all' : options.special ? 'special' : null;

    if (!scope) {
      console.error('Error: Must specify --all or --special');
      process.exit(1);
    }

    console.log(`Importing ${scope} duplicate order kits...`);

    if (options.background) {
      // Queue the job
      console.log('Job queued for background processing');
    } else {
      // Run synchronously
      const result = scope === 'all'
        ? await this.importService.importAll({ batchSize: options.batchSize })
        : await this.importService.importSpecial({ batchSize: options.batchSize });

      console.log('Import completed:', result);
    }
  }

  @Option({
    flags: '--all',
    description: 'Import all duplicate order kits',
  })
  parseAll(): boolean {
    return true;
  }

  @Option({
    flags: '--special',
    description: 'Import only special order duplicates',
  })
  parseSpecial(): boolean {
    return true;
  }

  @Option({
    flags: '--background',
    description: 'Run as background job',
  })
  parseBackground(): boolean {
    return true;
  }

  @Option({
    flags: '--batch-size <size>',
    description: 'Batch size for import (default: 1000)',
  })
  parseBatchSize(val: string): number {
    return parseInt(val, 10);
  }
}
```

### 6.2 Cleanup Command

Location: `src/cli/commands/cleanup-duplicate-order-kits.command.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { Command, CommandRunner, Option } from 'nest-commander';
import { CleanupDuplicateOrderKitsService } from 'src/audit/services/cleanup-duplicate-order-kits.service';

interface CleanupCommandOptions {
  all?: boolean;
  special?: boolean;
  dryRun?: boolean;
  confirm?: boolean;
}

@Injectable()
@Command({
  name: 'cleanup:duplicate-order-kits',
  description: 'Cleanup duplicate order kits (DESTRUCTIVE)',
})
export class CleanupDuplicateOrderKitsCommand extends CommandRunner {
  constructor(
    private readonly cleanupService: CleanupDuplicateOrderKitsService,
  ) {
    super();
  }

  async run(
    passedParams: string[],
    options: CleanupCommandOptions,
  ): Promise<void> {
    const scope = options.all ? 'all' : options.special ? 'special' : null;

    if (!scope) {
      console.error('Error: Must specify --all or --special');
      process.exit(1);
    }

    if (options.dryRun) {
      console.log(`Running dry-run for ${scope} cleanup...`);
      const result = await this.cleanupService.dryRun(scope);
      console.log('Dry-run results:', result);
      return;
    }

    if (scope === 'all' && !options.confirm) {
      console.error('');
      console.error('⚠️  ERROR: --confirm flag required for --all cleanup');
      console.error('This is a DESTRUCTIVE operation that will delete data.');
      console.error('');
      console.error('Run with --dry-run first to preview changes.');
      console.error('');
      process.exit(1);
    }

    console.log(`⚠️  Starting ${scope} cleanup (DESTRUCTIVE)...`);

    const result = scope === 'special'
      ? await this.cleanupService.cleanupSpecial({ dryRun: false })
      : await this.cleanupService.cleanupAll({
          dryRun: false,
          confirmationToken: 'CONFIRMED'
        });

    console.log('Cleanup completed:', result);
  }

  @Option({
    flags: '--all',
    description: 'Cleanup all duplicate order kits',
  })
  parseAll(): boolean {
    return true;
  }

  @Option({
    flags: '--special',
    description: 'Cleanup only special order duplicates',
  })
  parseSpecial(): boolean {
    return true;
  }

  @Option({
    flags: '--dry-run',
    description: 'Preview cleanup without making changes',
  })
  parseDryRun(): boolean {
    return true;
  }

  @Option({
    flags: '--confirm',
    description: 'Confirm destructive cleanup operation',
  })
  parseConfirm(): boolean {
    return true;
  }
}
```

---

## 7. Module Structure

### 7.1 Audit Module

Location: `src/audit/audit.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { DuplicateOrderKit } from './entity/duplicate-order-kit.entity';
import { SpecialDuplicateOrderKit } from './entity/special-duplicate-order-kit.entity';
import { ImportDuplicateOrderKitsService } from './services/import-duplicate-order-kits.service';
import { CleanupDuplicateOrderKitsService } from './services/cleanup-duplicate-order-kits.service';
import { AuditController } from './audit.controller';

@Module({
  imports: [
    TypeOrmModule.forFeature([
      DuplicateOrderKit,
      SpecialDuplicateOrderKit,
    ]),
  ],
  controllers: [AuditController],
  providers: [
    ImportDuplicateOrderKitsService,
    CleanupDuplicateOrderKitsService,
  ],
  exports: [
    ImportDuplicateOrderKitsService,
    CleanupDuplicateOrderKitsService,
  ],
})
export class AuditModule {}
```

### 7.2 Update App Module

Location: `src/app.module.ts`

Add the AuditModule to imports:

```typescript
import { AuditModule } from './audit/audit.module';

@Module({
  imports: [
    // ... existing imports
    AuditModule,
  ],
  // ...
})
export class AppModule {}
```

### 7.3 Update CLI Command Module

Location: `src/cli/command.module.ts`

```typescript
import { ImportDuplicateOrderKitsCommand } from './commands/import-duplicate-order-kits.command';
import { CleanupDuplicateOrderKitsCommand } from './commands/cleanup-duplicate-order-kits.command';
import { AuditModule } from 'src/audit/audit.module';

@Module({
  imports: [
    // ... existing imports
    AuditModule,
  ],
  providers: [
    // ... existing providers
    ImportDuplicateOrderKitsCommand,
    CleanupDuplicateOrderKitsCommand,
  ],
})
export class CommandModule {}
```

---

## 8. Backward Compatibility

### 8.1 Design Principles

1. **Non-Breaking Changes**: All new tables and entities are additive
2. **Isolated Database Connection**: Backup DB config is separate and optional
3. **Graceful Degradation**: System continues to work if backup DB is unavailable
4. **Migration Safety**: All migrations are reversible with `down()` methods
5. **Feature Flags**: Can be controlled via environment variables

### 8.2 Environment Variable Guards

```typescript
// In services that use backup DB
if (!process.env.BACKUP_DATABASE_HOST) {
  this.logger.warn('Backup database not configured, skipping import');
  return { skipped: true, reason: 'No backup database configured' };
}
```

### 8.3 Rollback Strategy

Each migration includes a `down()` method that:
- Drops created tables
- Deletes imported data
- Resets cleanup status
- Preserves data integrity

---

## 9. Execution Modes

### 9.1 Migration-Driven Execution

**Use Case**: Automated deployment pipelines

```bash
# Run all migrations including audit setup
yarn migration:run
```

**Behavior**:
- Creates tables automatically
- Prints instructions for manual data import
- Does NOT automatically import or cleanup data
- Safe for production deployments

### 9.2 Standalone CLI Execution

**Use Case**: Manual operations, testing, one-off tasks

```bash
# Import all duplicates
yarn start:cli import:duplicate-order-kits --all

# Import special duplicates only
yarn start:cli import:duplicate-order-kits --special

# Dry-run cleanup
yarn start:cli cleanup:duplicate-order-kits --special --dry-run

# Execute cleanup
yarn start:cli cleanup:duplicate-order-kits --special
```

**Behavior**:
- Can run independently of migrations
- Provides immediate feedback
- Supports dry-run mode
- Requires explicit confirmation for destructive operations

### 9.3 Background Job Execution

**Use Case**: Large datasets, long-running operations

```bash
# Queue import job
yarn start:cli import:duplicate-order-kits --all --background

# Monitor via Bull Board
# Visit: http://localhost:3000/v1/admin/queues
```

**Behavior**:
- Runs asynchronously
- Provides progress tracking
- Handles failures with retries
- Logs detailed execution history

---

## 10. Safety Mechanisms

### 10.1 Dry-Run Mode

All cleanup operations support dry-run:

```typescript
interface DryRunResult {
  scope: 'all' | 'special';
  totalRecords: number;
  recordsToClean: number;
  estimatedDuration: string;
  affectedOrders: string[];
  warnings: string[];
}
```

### 10.2 Confirmation Requirements

Destructive operations require explicit confirmation:

```typescript
// For cleanup all
if (scope === 'all' && !confirmationToken) {
  throw new Error('Confirmation required for cleanup all');
}
```

### 10.3 Audit Trail

Every operation is logged:

```typescript
{
  cleanupStatus: 'cleaned',
  cleanedAt: new Date(),
  cleanupJobId: 'job-12345',
  notes: 'Cleaned via CLI command on 2026-03-04'
}
```

### 10.4 Transaction Safety

All database operations use transactions:

```typescript
await this.dataSource.transaction(async (manager) => {
  // Import or cleanup operations
  // Automatically rolled back on error
});
```

---

## 11. Monitoring & Observability

### 11.1 Logging

All services use NestJS Logger:

```typescript
this.logger.log('Starting import of duplicate order kits');
this.logger.warn('Backup database connection slow');
this.logger.error('Failed to import batch', error);
```

### 11.2 Metrics

Track key metrics:
- Import duration
- Records processed
- Cleanup count
- Error rate
- Job queue depth

### 11.3 Bull Board Integration

Monitor background jobs at:
```
http://localhost:3000/v1/admin/queues
```

---

## 12. Testing Strategy

### 12.1 Unit Tests

**Entity Tests**:
- Validate entity structure
- Test constraints (e.g., special order ID check)
- Verify default values

**Service Tests**:
- Mock backup database connection
- Test batch processing logic
- Verify error handling
- Test dry-run mode

### 12.2 Integration Tests

**Migration Tests**:
- Test up/down migrations
- Verify table creation
- Test rollback scenarios

**Import Tests**:
- Test with sample backup data
- Verify data transformation
- Test batch processing
- Verify transaction rollback on error

**Cleanup Tests**:
- Test dry-run accuracy
- Verify cleanup logic
- Test audit trail updates
- Verify data integrity after cleanup

### 12.3 E2E Tests

**Full Workflow**:
1. Run migrations
2. Import duplicates
3. Run dry-run cleanup
4. Execute cleanup
5. Verify results

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
- [ ] Create import queue processor
- [ ] Create migration files (3-4)
- [ ] Write integration tests for import

### Phase 3: Cleanup System (Week 3)
- [ ] Implement cleanup service
- [ ] Create cleanup CLI command
- [ ] Create cleanup queue processor
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
# 1. Set environment variables
export BACKUP_DATABASE_HOST=backup-server.example.com
export BACKUP_DATABASE_PORT=1433
export BACKUP_DATABASE_USER=backup_user
export BACKUP_DATABASE_PASSWORD=secure_password
export BACKUP_DATABASE_NAME=vitract_backup

# 2. Run migrations
yarn migration:run

# 3. Verify tables created
# Check database for duplicate_order_kits and special_duplicate_order_kits tables
```

### 14.2 Import Duplicates

```bash
# Import all duplicates (recommended first)
yarn start:cli import:duplicate-order-kits --all

# Or import as background job for large datasets
yarn start:cli import:duplicate-order-kits --all --background

# Import special duplicates
yarn start:cli import:duplicate-order-kits --special
```

### 14.3 Cleanup Operations

```bash
# ALWAYS run dry-run first
yarn start:cli cleanup:duplicate-order-kits --special --dry-run

# Review dry-run results, then execute
yarn start:cli cleanup:duplicate-order-kits --special

# For cleanup all (DESTRUCTIVE)
yarn start:cli cleanup:duplicate-order-kits --all --dry-run
# Review carefully, then:
yarn start:cli cleanup:duplicate-order-kits --all --confirm
```

### 14.4 Monitoring

```bash
# Check import status
SELECT
  COUNT(*) as total,
  cleanupStatus,
  COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() as percentage
FROM duplicate_order_kits
GROUP BY cleanupStatus;

# Check special duplicates
SELECT * FROM special_duplicate_order_kits
WHERE cleanupStatus = 'pending';

# Monitor background jobs
# Visit: http://localhost:3000/v1/admin/queues
```

### 14.5 Troubleshooting

**Issue**: Import fails with connection timeout

```bash
# Solution: Increase connection timeout in backup-db.ts
# Or run with smaller batch size
yarn start:cli import:duplicate-order-kits --all --batch-size 500
```

**Issue**: Cleanup dry-run shows unexpected results

```bash
# Solution: Review the records manually
SELECT * FROM duplicate_order_kits
WHERE cleanupStatus = 'pending'
LIMIT 100;

# Verify against source data in backup DB
```

**Issue**: Migration rollback needed

```bash
# Rollback last migration
yarn migration:revert

# Rollback specific migration
# Edit migration file and run down() method manually
```

---

## 15. Security Considerations

### 15.1 Database Access

- Backup database credentials stored in environment variables
- Use read-only credentials for backup DB connection
- Implement connection pooling limits
- Enable SSL/TLS for backup DB connections

### 15.2 Data Privacy

- Audit tables contain PII (user IDs, kit IDs)
- Apply same access controls as main database
- Consider data retention policies
- Implement audit log for who accessed cleanup commands

### 15.3 Operation Authorization

- Cleanup operations require admin privileges
- Log all cleanup operations with user context
- Implement approval workflow for production cleanup
- Require confirmation tokens for destructive operations

---

## 16. Performance Considerations

### 16.1 Batch Processing

- Default batch size: 1000 records
- Configurable via CLI: `--batch-size <n>`
- Process in transactions to prevent partial imports
- Use bulk insert operations

### 16.2 Indexing Strategy

Indexes created for optimal query performance:
- `orderId` - For filtering by order
- `kitId` - For kit lookups
- `cleanupStatus` - For cleanup queries

### 16.3 Connection Pooling

Backup database connection pool:
- Max connections: 5
- Min connections: 0
- Idle timeout: 30 seconds
- Connection timeout: 60 seconds

### 16.4 Large Dataset Handling

For datasets > 100K records:
- Use background job execution
- Implement progress tracking
- Consider chunking by date ranges
- Monitor memory usage

---

## 17. Success Criteria

### 17.1 Functional Requirements

- ✅ Two audit tables created successfully
- ✅ Backup database connection isolated and working
- ✅ Import all duplicates from backup
- ✅ Import special duplicates for specific orders
- ✅ Cleanup special order duplicates
- ✅ Cleanup all duplicates (destructive)
- ✅ All operations reversible via migrations

### 17.2 Non-Functional Requirements

- ✅ Backward compatible with existing system
- ✅ Can run standalone without migrations
- ✅ Dry-run mode for all cleanup operations
- ✅ Comprehensive audit trail
- ✅ Performance: Process 10K records in < 5 minutes
- ✅ Error handling with automatic rollback
- ✅ Monitoring via Bull Board

### 17.3 Documentation Requirements

- ✅ PRD document (this document)
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
   - Cron job for periodic duplicate detection
   - Automatic import from backup on schedule
   - Email notifications for cleanup recommendations

3. **Advanced Analytics**
   - Duplicate pattern analysis
   - Root cause identification
   - Trend reporting over time

4. **Multi-Database Support**
   - Support multiple backup databases
   - Parallel import from multiple sources
   - Consolidated reporting

5. **Data Archival**
   - Archive cleaned duplicates to cold storage
   - Implement retention policies
   - Automated cleanup of old audit records

---

## 19. Risks & Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Data loss during cleanup | High | Low | Dry-run mode, confirmation required, audit trail |
| Backup DB connection failure | Medium | Medium | Graceful degradation, retry logic, timeout handling |
| Performance degradation | Medium | Low | Batch processing, background jobs, connection pooling |
| Migration conflicts | Low | Low | Timestamped migrations, thorough testing |
| Incorrect duplicate identification | High | Low | Manual review process, dry-run validation |

---

## 20. Appendix

### 20.1 Special Order IDs Reference

```typescript
export const SPECIAL_ORDER_IDS = [
  '7D01E25F-961C-F011-8B3D-6045BD8068CA',
  '750A2CC5-0A22-F011-8B3D-6045BD8068CA',
] as const;
```

### 20.2 Cleanup Status Enum

```typescript
export enum CleanupStatus {
  PENDING = 'pending',
  CLEANED = 'cleaned',
  SKIPPED = 'skipped',
  FAILED = 'failed',
}
```

### 20.3 Sample Queries

**Find all pending duplicates:**
```sql
SELECT * FROM duplicate_order_kits
WHERE cleanupStatus = 'pending'
ORDER BY createdAt DESC;
```

**Count duplicates by order:**
```sql
SELECT orderId, COUNT(*) as duplicate_count
FROM duplicate_order_kits
GROUP BY orderId
HAVING COUNT(*) > 1
ORDER BY duplicate_count DESC;
```

**Audit trail for cleaned records:**
```sql
SELECT
  orderId,
  kitId,
  cleanedAt,
  cleanupJobId,
  notes
FROM duplicate_order_kits
WHERE cleanupStatus = 'cleaned'
ORDER BY cleanedAt DESC;
```

---

## 21. Glossary

- **Audit Table**: A database table that stores historical records for tracking and compliance
- **Backup Database**: A separate database containing backup/historical data
- **Dry-Run**: A test execution that simulates an operation without making actual changes
- **Destructive Operation**: An operation that permanently deletes or modifies data
- **Migration**: A versioned database schema change
- **Background Job**: An asynchronous task processed by a queue worker
- **Batch Processing**: Processing data in chunks rather than all at once
- **Idempotent**: An operation that produces the same result regardless of how many times it's executed

---

## 22. References

- [TypeORM Documentation](https://typeorm.io/)
- [NestJS Documentation](https://docs.nestjs.com/)
- [Bull Queue Documentation](https://github.com/OptimalBits/bull)
- [SQL Server Best Practices](https://docs.microsoft.com/en-us/sql/relational-databases/)
- [Vitract Platform Database Schema](../src/docs/platform/database-schema.md)

---

**Document Version History**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-04 | System Architecture Team | Initial PRD creation |

---

**Approval Signatures**

- [ ] Technical Lead: _________________ Date: _______
- [ ] Product Manager: _________________ Date: _______
- [ ] Database Administrator: _________________ Date: _______
- [ ] Security Officer: _________________ Date: _______


