# Phase 2: Standardization & Correctness

## Goal

**Establish industry-standard patterns and architectural consistency** across all three client interfaces (REST, SSE, WebSocket).

## Why This Phase Exists

After Phase 1 eliminates critical risks, the system still has **architectural inconsistencies** that cause:
- **Business logic bypasses**: SSE/WebSocket controllers call cache directly, skipping service layer
- **Inconsistent behavior**: Same operation behaves differently depending on entry point
- **Maintenance burden**: Changes must be duplicated across 3 controllers
- **Future risk**: New business logic won't apply uniformly

**This phase ensures correctness and professional code structure.**

---

## Included Issues (Mapped to Audit)

| Audit ID | Issue | Severity | Why Phase 2 |
|----------|-------|----------|-------------|
| **H2** | Inconsistent service layer usage | HIGH | Business logic bypassed in SSE/WS |
| **H4** | External API relationship unclear | HIGH | Potential data inconsistency |
| **L3** | Inconsistent error responses | LOW | Poor client experience |

---

## Tasks

### Task 2.1: Unify Service Layer for SSE Controller

**Description**: Refactor SSE controller to use service layer instead of calling cache directly.

**Why Phase 2**: Ensures **business logic consistency**. Currently REST uses services, but SSE bypasses them.

**Files Affected**:
- `src/health-info/health-info-sse.controller.ts` (refactor all write endpoints)
- `src/health-info/services/add-question-response.ts` (already exists from Phase 1)
- `src/health-info/services/set-submitted.ts` (already exists from Phase 1)
- `src/health-info/services/set-agreement.ts` (already exists from Phase 1)
- `src/health-info/services/add-bulk-responses.ts` (NEW - extract from controller)
- `src/health-info/services/reset-response.ts` (NEW - extract from controller)

**Current State (INCONSISTENT)**:
```typescript
// SSE Controller - BYPASSES SERVICE LAYER ❌
@Post('add_question_response')
async addQuestionResponse(@Body() dto: AddQuestionResponseDto) {
  // Direct cache call - no validation, no audit, no business logic
  await this.cache.addQuestionResponse(dto);
  
  this.sseBus.sendOthers(dto.kitId, dto.userId, {
    event: 'response_updated',
    data: await this.cache.getResponse(dto.kitId)
  });
  
  return { ok: true };
}
```

**Target State (CONSISTENT)**:
```typescript
// SSE Controller - USES SERVICE LAYER ✅
@Post('add_question_response')
async addQuestionResponse(@Body() dto: AddQuestionResponseDto, @Req() req: Request) {
  // Enrich DTO with request context
  const enrichedDto = {
    ...dto,
    ipAddress: req.ip,
    userAgent: req.headers['user-agent'],
    requestId: req['requestId']  // From logging interceptor
  };

  // Use service layer (includes validation, audit, DB write)
  const newVersion = await this.addQuestionResponseService.execute(enrichedDto);
  
  // Broadcast to SSE clients
  const updatedData = await this.getResponseService.execute(dto.kitId);
  this.sseBus.sendOthers(dto.kitId, dto.userId, {
    event: 'response_updated',
    data: { ...updatedData, version: newVersion }
  });
  
  return { ok: true, version: newVersion };
}
```

**Acceptance Criteria**:
- ✅ All SSE write endpoints use service layer
- ✅ No direct cache calls from SSE controller
- ✅ Broadcast behavior preserved (SSE clients still receive updates)
- ✅ Request context (IP, user agent) passed to services
- ✅ Integration tests verify SSE writes create audit logs
- ✅ No performance regression (p95 latency within 10% of baseline)

**Rollout**:
- **Risk**: MEDIUM - Changes SSE write path
- **Deployment**: Deploy and monitor SSE error rates
- **Rollback**: Revert to direct cache calls

---

### Task 2.2: Unify Service Layer for WebSocket Gateway

**Description**: Refactor WebSocket gateway to use service layer instead of calling cache directly.

**Why Phase 2**: Same reason as Task 2.1 - ensures **business logic consistency**.

**Files Affected**:
- `src/health-info/health-info.gateway.ts` (refactor all write handlers)

**Current State (INCONSISTENT)**:
```typescript
// WebSocket Gateway - BYPASSES SERVICE LAYER ❌
@SubscribeMessage('add_question_response')
async handleAddQuestionResponse(
  @MessageBody() dto: AddQuestionResponseDto,
  @ConnectedSocket() client: Socket
) {
  await this.cache.addQuestionResponse(dto);
  
  const response = await this.cache.getResponse(dto.kitId);
  this.server.to(dto.kitId).emit('response_updated', response);
}
```

**Target State (CONSISTENT)**:
```typescript
// WebSocket Gateway - USES SERVICE LAYER ✅
@SubscribeMessage('add_question_response')
async handleAddQuestionResponse(
  @MessageBody() dto: AddQuestionResponseDto,
  @ConnectedSocket() client: Socket
) {
  // Enrich DTO with socket context
  const enrichedDto = {
    ...dto,
    ipAddress: client.handshake.address,
    userAgent: client.handshake.headers['user-agent'],
    requestId: client.id
  };

  // Use service layer
  const newVersion = await this.addQuestionResponseService.execute(enrichedDto);
  
  // Broadcast to room
  const updatedData = await this.getResponseService.execute(dto.kitId);
  this.server.to(dto.kitId).emit('response_updated', {
    ...updatedData,
    version: newVersion
  });
}
```

**Acceptance Criteria**:
- ✅ All WebSocket write handlers use service layer
- ✅ No direct cache calls from gateway
- ✅ Room broadcast behavior preserved
- ✅ Socket context (IP, user agent) passed to services
- ✅ Integration tests verify WebSocket writes create audit logs
- ✅ No performance regression

**Rollout**:
- **Risk**: MEDIUM - Changes WebSocket write path
- **Note**: WebSocket is not actively used (SSE is primary), lower risk
- **Deployment**: Deploy and monitor (if any WebSocket clients exist)
- **Rollback**: Revert to direct cache calls

---

### Task 2.3: Create Bulk Response Service

**Description**: Extract bulk response logic from controllers into dedicated service.

**Why Phase 2**: **Code reuse** and **consistency**. Currently duplicated in SSE and WebSocket.

**Files Affected**:
- `src/health-info/services/add-bulk-responses.ts` (NEW)
- `src/health-info/health-info-sse.controller.ts` (use new service)
- `src/health-info/health-info.gateway.ts` (use new service)

**Implementation**:
```typescript
// src/health-info/services/add-bulk-responses.ts
@Injectable()
export class AddBulkResponsesService {
  constructor(
    private readonly addQuestionResponseService: AddQuestionResponseService,
    private readonly logger: Logger
  ) {}

  async execute(dto: AddBulkResponsesDto): Promise<number> {
    const startTime = Date.now();
    let finalVersion: number;

    // Process responses sequentially to maintain version consistency
    for (const response of dto.responses) {
      const enrichedDto = {
        kitId: dto.kitId,
        categoryId: response.categoryId,
        questionResponse: response.questionResponse,
        completed: response.completed,
        userId: dto.userId,
        ipAddress: dto.ipAddress,
        userAgent: dto.userAgent,
        requestId: dto.requestId,
        version: finalVersion  // Use latest version for optimistic locking
      };

      finalVersion = await this.addQuestionResponseService.execute(enrichedDto);
    }

    this.logger.log({
      message: 'Bulk responses added',
      kitId: dto.kitId,
      count: dto.responses.length,
      finalVersion,
      duration: Date.now() - startTime
    });

    return finalVersion;
  }
}
```

**Acceptance Criteria**:
- ✅ Bulk response service created
- ✅ SSE and WebSocket use same service
- ✅ Responses processed sequentially (maintains version consistency)
- ✅ Single audit log entry per response (not one for bulk)
- ✅ Transaction rollback on any failure (all-or-nothing)
- ✅ Unit tests for bulk service

**Rollout**:
- **Risk**: LOW - New service, existing behavior preserved
- **Deployment**: Deploy and verify bulk operations work
- **Rollback**: Revert to inline logic

---

### Task 2.4: Standardize Error Response Format

**Description**: Create consistent error response structure across REST, SSE, and WebSocket.

**Why Phase 2**: **Better client experience**. Currently errors have different shapes depending on interface.

**Files Affected**:
- `src/common/filters/http-exception.filter.ts` (enhance for REST)
- `src/common/filters/sse-exception.filter.ts` (NEW - for SSE)
- `src/common/filters/ws-exception.filter.ts` (enhance for WebSocket)
- `src/health-info/health-info.module.ts` (register filters)

**Standard Error Format**:
```typescript
interface StandardErrorResponse {
  success: false;
  error: {
    code: string;           // Machine-readable: 'VERSION_CONFLICT', 'ALREADY_SUBMITTED'
    message: string;        // Human-readable: 'Version conflict: expected 5, current is 7'
    statusCode: number;     // HTTP status: 400, 404, 409, 500
    timestamp: string;      // ISO 8601: '2026-02-10T14:30:00Z'
    path: string;           // Request path: '/health-info/response'
    requestId: string;      // Trace ID: 'req-abc-123'
    details?: any;          // Optional context
  };
}
```

**Examples**:
```typescript
// Version conflict (409)
{
  success: false,
  error: {
    code: 'VERSION_CONFLICT',
    message: 'Version conflict: expected 5, current is 7. Please refresh and retry.',
    statusCode: 409,
    timestamp: '2026-02-10T14:30:00Z',
    path: '/health-info/response',
    requestId: 'req-abc-123',
    details: { expectedVersion: 5, currentVersion: 7 }
  }
}

// Already submitted (400)
{
  success: false,
  error: {
    code: 'QUESTIONNAIRE_ALREADY_SUBMITTED',
    message: 'Cannot modify questionnaire KIT12345: already submitted at 2026-02-09T10:00:00Z',
    statusCode: 400,
    timestamp: '2026-02-10T14:30:00Z',
    path: '/health-info/response',
    requestId: 'req-abc-123',
    details: { kitId: 'KIT12345', submittedAt: '2026-02-09T10:00:00Z' }
  }
}
```

**Acceptance Criteria**:
- ✅ All three interfaces return same error structure
- ✅ Error codes documented in API specification
- ✅ Request ID included in all errors
- ✅ PII excluded from error messages
- ✅ Client SDK updated with error code constants
- ✅ Integration tests verify error format

**Rollout**:
- **Risk**: LOW - Error handling only
- **Backward Compatibility**: Clients should handle new format gracefully
- **Deployment**: Deploy and monitor client error handling
- **Rollback**: Revert to original error filters

---

### Task 2.5: Document External API Relationship

**Description**: Clarify relationship with VitractHealthInfoClient and implement sync strategy if needed.

**Why Phase 2**: **Prevents future inconsistencies**. Currently unclear if external API should be synced.

**Files Affected**:
- `docs/EXTERNAL_API_INTEGRATION.md` (NEW - documentation)
- `src/integrations/health-info.client.ts` (add comments)
- `src/health-info/services/sync-external-api.ts` (NEW - if sync needed)

**Investigation Questions**:
1. What is the external Vitract API?
   - Legacy system being replaced?
   - Authoritative source for questionnaire templates?
   - Read-only reference?

2. Should we sync data?
   - One-way (us → external)?
   - One-way (external → us)?
   - Two-way?
   - No sync (independent systems)?

3. What happens on conflict?
   - External has data we don't?
   - We have data external doesn't?
   - Data differs?

**Decision Tree**:
```
Is external API authoritative for questionnaire responses?
├─ YES → Implement read-from-external fallback
│         (if cache miss, try external before returning empty)
│
└─ NO → Is external API authoritative for questionnaire templates?
   ├─ YES → Fetch templates from external, store responses locally
   │         (current behavior, document it)
   │
   └─ NO → External API is read-only reference
            (document, no sync needed)
```

**Acceptance Criteria**:
- ✅ External API relationship documented
- ✅ Sync strategy decided and documented
- ✅ If sync needed: implementation complete and tested
- ✅ If no sync: document why and add monitoring for drift
- ✅ Team alignment on external API role

**Rollout**:
- **Risk**: Depends on decision (LOW if no sync, MEDIUM if sync)
- **Deployment**: If sync implemented, deploy with feature flag
- **Rollback**: Disable sync, revert to current behavior

---

## Exit Criteria

Phase 2 is complete when **ALL** of the following are true:

### Architectural Requirements
- ✅ All three interfaces (REST/SSE/WebSocket) use unified service layer
- ✅ No direct cache calls from controllers/gateways
- ✅ Business logic centralized in service layer
- ✅ Code duplication eliminated

### API Requirements
- ✅ Error responses standardized across all interfaces
- ✅ Error codes documented
- ✅ Version field included in all responses
- ✅ Request ID propagated through all layers

### Documentation Requirements
- ✅ External API relationship documented
- ✅ API contracts documented (OpenAPI/AsyncAPI)
- ✅ Service layer responsibilities documented
- ✅ Error handling guide for clients

### Testing Requirements
- ✅ Integration tests verify all interfaces use same code path
- ✅ Unit tests for all new services
- ✅ Error format tests for all error scenarios
- ✅ No regressions in SSE/WebSocket behavior

---

## Dependencies

### Prerequisites
- **Phase 1 complete** (database, write-through cache, monitoring)
- Service layer exists and is tested

### External Coordination
- **Frontend team**: Notify of standardized error format
- **Product team**: Clarify external API relationship

---

## Estimated Timeline

| Task | Duration | Dependencies |
|------|----------|--------------|
| 2.1: Unify SSE service layer | 1 day | Phase 1 |
| 2.2: Unify WebSocket service layer | 1 day | Phase 1 |
| 2.3: Bulk response service | 0.5 day | Tasks 2.1, 2.2 |
| 2.4: Standardize errors | 1 day | None (parallel) |
| 2.5: Document external API | 0.5 day | Product input |
| **Testing & Validation** | 1 day | All tasks |
| **TOTAL** | **4-5 days** | |

---

**Phase 2 should only begin after Phase 1 exit criteria are met.**

