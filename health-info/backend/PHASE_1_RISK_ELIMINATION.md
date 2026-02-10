# Phase 1: Risk Elimination

## Goal

**Eliminate all critical data loss, data corruption, and broken API behavior** from the Health Information backend system.

## Why This Phase Exists

The current system has **active, ongoing risks** that cause:
- **Data loss**: All questionnaire responses expire after 24 hours (Redis TTL)
- **Broken functionality**: REST GET endpoint always returns empty (critical bug)
- **Data corruption**: Concurrent writes can overwrite each other (race conditions)
- **Compliance violations**: Medical data has no audit trail (HIPAA/GDPR)
- **Silent failures**: No monitoring means failures go undetected

**These are not theoretical risks - they are happening right now.**

---

## Included Issues (Mapped to Audit)

| Audit ID | Issue | Severity | Why Phase 1 |
|----------|-------|----------|-------------|
| **C1** | No persistent storage | CRITICAL | **Active data loss** - All data expires after 24h |
| **C2** | REST GET always returns empty | CRITICAL | **Broken API** - Users cannot retrieve their data |
| **C3** | No HIPAA/GDPR compliance | CRITICAL | **Legal liability** - Medical data requires audit trail |
| **C4** | Race conditions in concurrent writes | CRITICAL | **Data corruption** - Last write wins, no locking |
| **H3** | No validation that submitted=true prevents edits | HIGH | **Data integrity** - Submitted forms can be modified |
| **M1** | No monitoring/alerting | MEDIUM | **Blind spots** - Failures go undetected |

---

## Tasks

### Task 1.1: Fix REST GET Bug (Immediate)

**Description**: Remove the `resetResponse()` call that causes REST GET to always return empty data.

**Why Phase 1**: This is a **critical bug** that breaks the REST API. Users cannot retrieve their questionnaire responses via REST.

**Files Affected**:
- `src/health-info/services/get-response.ts` (line 14)

**Implementation**:
```typescript
// BEFORE (BROKEN):
async execute(kitId: string): Promise<QuestionnaireResponseInterface> {
  await this.cache.resetResponse(kitId);  // ❌ DELETES DATA
  return this.cache.getResponse(kitId);   // Returns empty
}

// AFTER (FIXED):
async execute(kitId: string): Promise<QuestionnaireResponseInterface> {
  return this.cache.getResponse(kitId);   // ✅ Returns actual data
}
```

**Acceptance Criteria**:
- ✅ REST GET `/health-info/:kitId` returns actual cached data
- ✅ Integration test added: write response → read via REST → verify data matches
- ✅ No regressions in SSE/WebSocket reads

**Rollout**:
- **Risk**: LOW - Simple bug fix, no side effects
- **Deployment**: Can deploy immediately
- **Rollback**: Revert single line change

---

### Task 1.2: Create Database Schema (Foundation)

**Description**: Create SQL Server tables to persist questionnaire responses, answers, agreements, and audit logs.

**Why Phase 1**: **Database is prerequisite** for eliminating data loss risk. Must exist before write-through pattern.

**Files Affected**:
- `src/health-info/entity/questionnaire-response.entity.ts` (NEW)
- `src/health-info/entity/questionnaire-answer.entity.ts` (NEW)
- `src/health-info/entity/questionnaire-agreement.entity.ts` (NEW)
- `src/health-info/entity/questionnaire-audit-log.entity.ts` (NEW)
- `src/migrations/YYYYMMDDHHMMSS-create-questionnaire-tables.ts` (NEW)

**Schema Design**:

```sql
-- questionnaire_responses (parent table)
CREATE TABLE questionnaire_responses (
  id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
  kit_id VARCHAR(50) NOT NULL UNIQUE,
  order_id UNIQUEIDENTIFIER NOT NULL,
  submitted BIT NOT NULL DEFAULT 0,
  submitted_at DATETIME2 NULL,
  version INT NOT NULL DEFAULT 1,  -- For optimistic locking (Task 1.4)
  created_at DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
  updated_at DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
  INDEX IX_order_id (order_id),
  INDEX IX_submitted (submitted, updated_at)
);

-- questionnaire_answers (child table)
CREATE TABLE questionnaire_answers (
  id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
  response_id UNIQUEIDENTIFIER NOT NULL,
  category_id INT NOT NULL,
  question_id VARCHAR(100) NOT NULL,
  answer_value NVARCHAR(MAX) NOT NULL,  -- JSON for complex answers
  completed BIT NOT NULL DEFAULT 0,
  created_at DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
  updated_at DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
  FOREIGN KEY (response_id) REFERENCES questionnaire_responses(id) ON DELETE CASCADE,
  UNIQUE (response_id, category_id, question_id),
  INDEX IX_response_category (response_id, category_id)
);

-- questionnaire_agreements (child table)
CREATE TABLE questionnaire_agreements (
  id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
  response_id UNIQUEIDENTIFIER NOT NULL,
  agreement_type VARCHAR(50) NOT NULL,  -- 'terms' or 'policy'
  accepted BIT NOT NULL,
  accepted_at DATETIME2 NULL,
  created_at DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
  FOREIGN KEY (response_id) REFERENCES questionnaire_responses(id) ON DELETE CASCADE,
  UNIQUE (response_id, agreement_type)
);

-- questionnaire_audit_log (compliance requirement)
CREATE TABLE questionnaire_audit_log (
  id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
  response_id UNIQUEIDENTIFIER NOT NULL,
  action VARCHAR(50) NOT NULL,  -- 'CREATE', 'UPDATE_ANSWER', 'SUBMIT', etc.
  user_id VARCHAR(100) NULL,
  changes NVARCHAR(MAX) NULL,  -- JSON diff
  ip_address VARCHAR(45) NULL,
  user_agent VARCHAR(500) NULL,
  created_at DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
  FOREIGN KEY (response_id) REFERENCES questionnaire_responses(id) ON DELETE CASCADE,
  INDEX IX_response_created (response_id, created_at)
);
```

**Acceptance Criteria**:
- ✅ TypeORM entities created with proper relationships
- ✅ Migration script generated and tested in dev environment
- ✅ Foreign key constraints enforce referential integrity
- ✅ Indexes created for common query patterns
- ✅ Rollback migration script tested

**Rollout**:
- **Risk**: LOW - Schema creation only, no data migration
- **Deployment**: Run migration in staging → verify → production
- **Rollback**: Run down migration (drops tables)

---

### Task 1.3: Implement Write-Through Cache Pattern

**Description**: Update cache service to write to database FIRST, then update cache. Database becomes source of truth.

**Why Phase 1**: **Eliminates data loss risk**. Even if Redis restarts or TTL expires, data persists in database.

**Files Affected**:
- `src/database/cache/response.ts` (major refactor)
- `src/health-info/services/add-question-response.ts` (update to use new pattern)
- `src/health-info/services/set-submitted.ts` (update to use new pattern)
- `src/health-info/services/set-agreement.ts` (update to use new pattern)

**Implementation Pattern**:

```typescript
// NEW: Write-through pattern
async addQuestionResponse(dto: AddQuestionResponseDto): Promise<void> {
  // 1. Start database transaction
  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    // 2. Write to database FIRST (source of truth)
    let response = await this.responseRepo.findOne({
      where: { kitId: dto.kitId },
      relations: ['answers', 'agreements']
    });

    if (!response) {
      response = this.responseRepo.create({
        kitId: dto.kitId,
        orderId: dto.orderId,  // Must be provided
        submitted: false,
        version: 1
      });
      await queryRunner.manager.save(response);
    }

    // 3. Upsert answer
    const answer = await this.answerRepo.findOne({
      where: {
        responseId: response.id,
        categoryId: dto.categoryId,
        questionId: dto.questionResponse.questionId
      }
    });

    if (answer) {
      answer.answerValue = JSON.stringify(dto.questionResponse.answer);
      answer.completed = dto.completed;
      answer.updatedAt = new Date();
    } else {
      const newAnswer = this.answerRepo.create({
        responseId: response.id,
        categoryId: dto.categoryId,
        questionId: dto.questionResponse.questionId,
        answerValue: JSON.stringify(dto.questionResponse.answer),
        completed: dto.completed
      });
      await queryRunner.manager.save(newAnswer);
    }

    // 4. Create audit log entry (compliance)
    const auditLog = this.auditRepo.create({
      responseId: response.id,
      action: 'UPDATE_ANSWER',
      userId: dto.userId,
      changes: JSON.stringify({
        categoryId: dto.categoryId,
        questionId: dto.questionResponse.questionId,
        answer: dto.questionResponse.answer
      }),
      ipAddress: dto.ipAddress,  // Pass from controller
      userAgent: dto.userAgent   // Pass from controller
    });
    await queryRunner.manager.save(auditLog);

    // 5. Commit transaction
    await queryRunner.commitTransaction();

    // 6. Update cache (performance layer)
    // If cache write fails, log error but don't fail request
    try {
      const cacheData = await this.buildCacheData(response.id);
      await this.cacheManager.set(
        `questionnaire-response-${dto.kitId}`,
        cacheData,
        60 * 60  // 1 hour TTL (reduced from 24h)
      );
    } catch (cacheError) {
      this.logger.error('Cache write failed, data safe in DB', cacheError);
    }

  } catch (error) {
    // 7. Rollback on any error
    await queryRunner.rollbackTransaction();
    throw error;
  } finally {
    await queryRunner.release();
  }
}
```

**Acceptance Criteria**:
- ✅ All writes go to database first, then cache
- ✅ Database transaction ensures atomicity
- ✅ Audit log created for every write operation
- ✅ Cache write failures don't fail the request
- ✅ Integration tests verify data persists after Redis restart
- ✅ Performance tests show <200ms p95 write latency

**Rollout**:
- **Risk**: MEDIUM - Changes core write path
- **Feature Flag**: `ENABLE_DB_WRITE_THROUGH` (default: false)
- **Deployment**:
  1. Deploy with flag OFF
  2. Enable for 10% of traffic
  3. Monitor error rates and latency
  4. Gradually increase to 100%
- **Rollback**: Disable feature flag, revert to cache-only writes

---

### Task 1.4: Add Optimistic Locking (Race Condition Fix)

**Description**: Use version field to detect and prevent concurrent write conflicts.

**Why Phase 1**: **Prevents data corruption**. Without locking, concurrent writes can overwrite each other silently.

**Files Affected**:
- `src/database/cache/response.ts` (add version check)
- `src/health-info/dto/add-question-response.dto.ts` (add version field)
- `src/health-info/health-info.controller.ts` (return version in responses)
- `src/health-info/health-info-sse.controller.ts` (return version in responses)
- `src/health-info/health-info.gateway.ts` (return version in responses)

**Implementation**:

```typescript
async addQuestionResponse(dto: AddQuestionResponseDto): Promise<number> {
  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    // 1. Fetch current version
    const response = await this.responseRepo.findOne({
      where: { kitId: dto.kitId }
    });

    if (!response) {
      // New response, no version conflict possible
      // ... create logic from Task 1.3
      return 1;
    }

    // 2. Check version match (optimistic lock)
    if (dto.version && dto.version !== response.version) {
      throw new ConflictException(
        `Version conflict: expected ${dto.version}, current is ${response.version}. ` +
        `Please refresh and retry.`
      );
    }

    // 3. Perform update
    // ... update logic from Task 1.3

    // 4. Increment version
    response.version += 1;
    response.updatedAt = new Date();
    await queryRunner.manager.save(response);

    await queryRunner.commitTransaction();

    // 5. Return new version to client
    return response.version;

  } catch (error) {
    await queryRunner.rollbackTransaction();
    throw error;
  } finally {
    await queryRunner.release();
  }
}
```

**Client Handling**:
```typescript
// Client receives 409 Conflict
// 1. Fetch latest version: GET /health-info/:kitId
// 2. Merge changes (or prompt user)
// 3. Retry with new version
```

**Acceptance Criteria**:
- ✅ Concurrent writes return 409 Conflict with clear error message
- ✅ Version field included in all read responses
- ✅ Version field accepted (optional) in all write requests
- ✅ Integration test: simulate concurrent writes → verify one succeeds, one gets 409
- ✅ Documentation updated with retry guidance

**Rollout**:
- **Risk**: MEDIUM - Changes API contract (adds version field)
- **Backward Compatibility**: Version field is optional in requests (for now)
- **Deployment**: Deploy backend → update client SDKs → make version required
- **Rollback**: Remove version check, keep field for future use

---

### Task 1.5: Add Submitted Status Validation

**Description**: Reject all write operations when `submitted=true` to prevent modification of submitted questionnaires.

**Why Phase 1**: **Data integrity violation**. Submitted forms should be immutable, but currently can be modified.

**Files Affected**:
- `src/health-info/services/add-question-response.ts` (add validation)
- `src/health-info/services/set-agreement.ts` (add validation)
- `src/database/cache/response.ts` (add validation helper)

**Implementation**:

```typescript
async addQuestionResponse(dto: AddQuestionResponseDto): Promise<void> {
  // 1. Check if already submitted
  const response = await this.responseRepo.findOne({
    where: { kitId: dto.kitId }
  });

  if (response?.submitted) {
    throw new BadRequestException(
      `Cannot modify questionnaire ${dto.kitId}: already submitted at ${response.submittedAt}`
    );
  }

  // 2. Proceed with write (Task 1.3 logic)
  // ...
}

async setSubmitted(dto: SetSubmittedDto): Promise<void> {
  // Allow setting submitted=false (for admin corrections)
  // But log it prominently in audit trail

  if (!dto.submitted) {
    this.logger.warn(`ADMIN ACTION: Unmarking questionnaire ${dto.kitId} as submitted`);
  }

  // Proceed with update
  // ...
}
```

**Acceptance Criteria**:
- ✅ Write operations return 400 Bad Request when submitted=true
- ✅ Error message includes submission timestamp
- ✅ Setting submitted=false is allowed (admin override) but logged
- ✅ Integration test: submit → attempt write → verify rejection
- ✅ Audit log captures all submission status changes

**Rollout**:
- **Risk**: LOW - Enforces expected behavior
- **Deployment**: Can deploy immediately
- **Rollback**: Remove validation check

---

### Task 1.6: Add Basic Monitoring & Alerting

**Description**: Implement metrics, structured logging, and alerts for critical failures.

**Why Phase 1**: **Observability blind spot**. Without monitoring, data loss and failures go undetected.

**Files Affected**:
- `src/health-info/health-info.module.ts` (add Prometheus module)
- `src/database/cache/response.ts` (add metrics)
- `src/health-info/services/*.ts` (add structured logging)
- `src/common/interceptors/logging.interceptor.ts` (NEW - request logging)

**Metrics to Track**:

```typescript
// Write operations
healthinfo_write_total{operation="add_response|set_submitted|set_agreement", status="success|failure"}
healthinfo_write_duration_seconds{operation="...", percentile="p50|p95|p99"}

// Database operations
healthinfo_db_write_total{status="success|failure"}
healthinfo_db_write_duration_seconds{percentile="p50|p95|p99"}

// Cache operations
healthinfo_cache_hit_total{operation="get|set"}
healthinfo_cache_miss_total
healthinfo_cache_error_total

// Version conflicts
healthinfo_version_conflict_total

// Validation failures
healthinfo_submitted_write_rejected_total
```

**Structured Logging**:

```typescript
this.logger.log({
  message: 'Questionnaire response added',
  kitId: dto.kitId,
  categoryId: dto.categoryId,
  questionId: dto.questionResponse.questionId,
  userId: dto.userId,
  version: newVersion,
  duration: Date.now() - startTime,
  requestId: context.requestId  // From interceptor
});
```

**Alerts**:

```yaml
# Critical alerts
- alert: HealthInfoDatabaseWriteFailureRate
  expr: rate(healthinfo_db_write_total{status="failure"}[5m]) > 0.01
  severity: critical
  message: "Database write failure rate >1% for health info"

- alert: HealthInfoCacheErrorRate
  expr: rate(healthinfo_cache_error_total[5m]) > 0.05
  severity: warning
  message: "Cache error rate >5% for health info"

- alert: HealthInfoHighVersionConflicts
  expr: rate(healthinfo_version_conflict_total[5m]) > 10
  severity: warning
  message: "High rate of version conflicts (>10/min)"
```

**Acceptance Criteria**:
- ✅ Prometheus metrics endpoint exposes all defined metrics
- ✅ Grafana dashboard created with key metrics
- ✅ Structured logs include request ID, user ID, kit ID
- ✅ PII (answer values) redacted from logs
- ✅ Alerts configured in monitoring system
- ✅ Alert runbook created for on-call engineers

**Rollout**:
- **Risk**: LOW - Observability only, no behavior change
- **Deployment**: Deploy monitoring infrastructure → deploy code changes
- **Rollback**: N/A (monitoring can stay even if code reverts)

---

## Exit Criteria

Phase 1 is complete when **ALL** of the following are true:

### Functional Requirements
- ✅ REST GET `/health-info/:kitId` returns actual data (not empty)
- ✅ All questionnaire responses persist to SQL Server database
- ✅ Write-through cache pattern implemented and enabled for 100% of traffic
- ✅ Submitted questionnaires cannot be modified (validation enforced)
- ✅ Concurrent writes return 409 Conflict (optimistic locking working)
- ✅ All write operations create audit log entries

### Testing Requirements
- ✅ Integration tests pass for all critical paths:
  - Write response → read via REST/SSE/WS → verify data matches
  - Submit questionnaire → attempt write → verify rejection
  - Concurrent writes → verify conflict detection
  - Redis restart → verify data recoverable from database
- ✅ Load test shows <200ms p95 write latency under normal load
- ✅ Zero data loss in 24-hour soak test

### Operational Requirements
- ✅ Monitoring dashboard shows all key metrics
- ✅ Alerts firing correctly in test scenarios
- ✅ Runbook created for common failure scenarios
- ✅ Database backup/restore tested
- ✅ Rollback procedures documented and tested

### Compliance Requirements
- ✅ Audit log captures: who, what, when for all changes
- ✅ PII redacted from application logs
- ✅ Data retention policy documented
- ✅ HIPAA compliance checklist reviewed (if applicable)

---

## Rollback Procedures

### If Task 1.3 (Write-Through) Causes Issues

1. **Immediate**: Disable feature flag `ENABLE_DB_WRITE_THROUGH`
2. **Verify**: System reverts to cache-only writes
3. **Investigate**: Check logs for error patterns
4. **Fix**: Address root cause
5. **Re-enable**: Gradually roll out again

### If Database Performance Degrades

1. **Check**: Database connection pool exhaustion
2. **Scale**: Increase connection pool size or database resources
3. **Optimize**: Add missing indexes (check slow query log)
4. **Fallback**: Temporarily disable write-through if critical

### If Version Conflicts Spike

1. **Investigate**: Are multiple users editing same kit?
2. **Expected**: Some conflicts are normal in concurrent scenarios
3. **Unexpected**: If >10% of writes conflict, investigate client retry logic
4. **Mitigation**: Increase client-side debouncing

---

## Dependencies

### Prerequisites
- SQL Server database accessible from application
- Redis instance running (existing)
- Prometheus/Grafana monitoring infrastructure
- TypeORM configured and working

### External Coordination
- **Frontend team**: Notify of version field addition (optional for now)
- **DevOps team**: Database migration approval
- **Compliance team**: Review audit log schema

---

## Estimated Timeline

| Task | Duration | Dependencies |
|------|----------|--------------|
| 1.1: Fix REST GET bug | 2 hours | None |
| 1.2: Create database schema | 1 day | None |
| 1.3: Write-through cache | 2 days | Task 1.2 |
| 1.4: Optimistic locking | 1 day | Task 1.3 |
| 1.5: Submitted validation | 4 hours | Task 1.3 |
| 1.6: Monitoring | 1 day | None (parallel) |
| **Testing & Validation** | 1 day | All tasks |
| **TOTAL** | **5-6 days** | |

---

**Phase 1 is MANDATORY. Do not proceed to Phase 2 until all exit criteria are met.**


