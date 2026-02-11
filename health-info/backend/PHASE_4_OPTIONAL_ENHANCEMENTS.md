# Phase 4: Optional Enhancements

## ⚠️ THIS PHASE IS OPTIONAL ⚠️

**This phase contains nice-to-have features that are NOT required for system safety, correctness, or compliance.**

Implement these features **only if**:
- All previous phases (1-3) are complete
- Product team explicitly requests them
- Resources are available
- There are no higher-priority initiatives

---

## Goal

**Add advanced collaboration features and performance optimizations** to enhance user experience.

## Why This Phase Exists

After Phases 1-3 ensure the system is **safe, correct, and maintainable**, this phase adds:
- **Better multi-user collaboration**: Advanced SSE coordination, presence indicators
- **Better performance**: Cache warming, read replicas, compression
- **Better UX**: Versioning helpers, conflict resolution UI support
- **Better developer experience**: Enhanced debugging tools, performance profiling

**These are conveniences, not requirements.**

---

## Included Features

| Feature | Benefit | Complexity | Priority |
|---------|---------|------------|----------|
| Advanced SSE room management | Better multi-user coordination | MEDIUM | OPTIONAL |
| Cache warming | Faster first-load performance | LOW | OPTIONAL |
| Conflict resolution helpers | Better UX for version conflicts | MEDIUM | OPTIONAL |
| Read replicas | Improved read scalability | HIGH | OPTIONAL |
| Response compression | Reduced bandwidth | LOW | OPTIONAL |
| Performance profiling | Better optimization insights | LOW | OPTIONAL |

---

## Tasks

### Task 4.1: Advanced SSE Room Management (OPTIONAL)

**Description**: Add presence indicators, typing indicators, and cursor position sharing for real-time collaboration.

**Why Optional**: Current SSE implementation supports basic multi-user scenarios. Advanced features are nice-to-have.

**Files Affected**:
- `src/health-info/health-info-sse.bus.ts` (enhance)
- `src/health-info/health-info-sse.controller.ts` (add endpoints)
- `src/health-info/dto/presence.dto.ts` (NEW)

**Features**:

1. **User Presence**:
```typescript
// Broadcast when user joins/leaves
{
  event: 'user_presence',
  data: {
    userId: 'user-123',
    status: 'active' | 'idle' | 'disconnected',
    lastSeen: '2026-02-10T14:30:00Z',
    currentQuestion: 'Q5'  // Optional: what they're viewing
  }
}
```

2. **Typing Indicators**:
```typescript
// Broadcast when user is typing
POST /v1/sse/health-info/typing
{
  kitId: 'KIT12345',
  userId: 'user-123',
  questionId: 'Q5',
  isTyping: true
}

// Broadcast to others
{
  event: 'user_typing',
  data: {
    userId: 'user-123',
    questionId: 'Q5',
    isTyping: true
  }
}
```

3. **Cursor Position Sharing**:
```typescript
// Broadcast cursor position for collaborative editing
POST /v1/sse/health-info/cursor
{
  kitId: 'KIT12345',
  userId: 'user-123',
  questionId: 'Q5',
  position: { line: 2, column: 15 }
}
```

**Acceptance Criteria**:
- ✅ Presence events broadcast to all room members
- ✅ Typing indicators debounced (max 1 event per second)
- ✅ Cursor positions tracked per user
- ✅ Idle detection (no activity for 5 minutes)
- ✅ Cleanup on disconnect
- ✅ Performance: <10ms broadcast latency

**Rollout**:
- **Risk**: LOW - Additive features
- **Feature Flag**: `ENABLE_ADVANCED_SSE_FEATURES`
- **Deployment**: Deploy with flag OFF → enable for beta users → general availability

---

### Task 4.2: Cache Warming (OPTIONAL)

**Description**: Pre-populate cache on kit registration to avoid cold start latency.

**Why Optional**: Current cache-on-demand works fine. Warming is a performance optimization.

**Files Affected**:
- `src/kit/service/enqueue-health-info-dispatch.ts` (trigger warming)
- `src/health-info/services/warm-cache.ts` (NEW)
- `src/queues/processors/cache-warming.processor.ts` (NEW)

**Implementation**:

```typescript
// src/health-info/services/warm-cache.ts
@Injectable()
export class WarmCacheService {
  constructor(
    private readonly responseRepo: Repository<QuestionnaireResponse>,
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    private logger: Logger
  ) {}

  async warmCache(kitId: string): Promise<void> {
    const startTime = Date.now();

    // 1. Fetch from database
    const response = await this.responseRepo.findOne({
      where: { kitId },
      relations: ['answers', 'agreements']
    });

    if (!response) {
      this.logger.debug(`No data to warm for kit ${kitId}`);
      return;
    }

    // 2. Build cache structure
    const cacheData = this.buildCacheData(response);

    // 3. Write to cache
    await this.cacheManager.set(
      `questionnaire-response-${kitId}`,
      cacheData,
      60 * 60  // 1 hour TTL
    );

    this.logger.log({
      message: 'Cache warmed',
      kitId,
      duration: Date.now() - startTime
    });
  }
}

// Trigger on kit registration
async execute(kitId: string, orderId: string): Promise<void> {
  // ... existing dispatch logic

  // Queue cache warming (low priority)
  await this.cacheWarmingQueue.add('warm-cache', {
    kitId,
    orderId
  }, {
    priority: 1,  // Low priority
    delay: 5000   // Wait 5 seconds (let dispatch email send first)
  });
}
```

**Acceptance Criteria**:
- ✅ Cache warmed on kit registration
- ✅ Warming happens asynchronously (doesn't block registration)
- ✅ Failed warming logged but doesn't fail registration
- ✅ Metrics track warming success rate
- ✅ Performance: First read after warming <50ms p95

**Rollout**:
- **Risk**: LOW - Background process, no user-facing impact
- **Feature Flag**: `ENABLE_CACHE_WARMING`
- **Deployment**: Deploy and monitor cache hit rates

---

### Task 4.3: Conflict Resolution Helpers (OPTIONAL)

**Description**: Add endpoints to help clients resolve version conflicts gracefully.

**Why Optional**: Clients can already handle 409 conflicts by refetching. This makes it easier.

**Files Affected**:
- `src/health-info/health-info.controller.ts` (add endpoints)
- `src/health-info/services/merge-responses.ts` (NEW)
- `src/health-info/dto/merge-request.dto.ts` (NEW)

**Features**:

1. **Get Diff Endpoint**:
```typescript
// Compare two versions
GET /v1/health-info/:kitId/diff?fromVersion=5&toVersion=7

Response:
{
  fromVersion: 5,
  toVersion: 7,
  changes: [
    {
      type: 'answer_updated',
      categoryId: 1,
      questionId: 'Q5',
      oldValue: 'Yes',
      newValue: 'No',
      changedBy: 'user-456',
      changedAt: '2026-02-10T14:30:00Z'
    },
    {
      type: 'answer_added',
      categoryId: 2,
      questionId: 'Q8',
      value: 'New answer',
      changedBy: 'user-789',
      changedAt: '2026-02-10T14:31:00Z'
    }
  ]
}
```

2. **Three-Way Merge Endpoint**:
```typescript
// Merge client changes with server state
POST /v1/health-info/:kitId/merge
{
  baseVersion: 5,           // Version client started from
  clientChanges: [...],     // What client changed
  serverVersion: 7          // Current server version
}

Response:
{
  merged: true,
  newVersion: 8,
  conflicts: [],            // Empty if auto-merged
  result: { ... }           // Merged questionnaire
}

// Or if conflicts exist:
{
  merged: false,
  conflicts: [
    {
      questionId: 'Q5',
      clientValue: 'Yes',
      serverValue: 'No',
      resolution: 'manual'  // Client must choose
    }
  ]
}
```

**Acceptance Criteria**:
- ✅ Diff endpoint returns all changes between versions
- ✅ Merge endpoint auto-resolves non-conflicting changes
- ✅ Conflicts clearly identified
- ✅ Audit log tracks merge operations
- ✅ Client SDK updated with merge helpers

**Rollout**:
- **Risk**: LOW - New endpoints, existing flows unchanged
- **Deployment**: Deploy and document for client teams

---

### Task 4.4: Read Replicas (OPTIONAL)

**Description**: Route read operations to database replicas for improved scalability.

**Why Optional**: Current read performance is acceptable. Replicas needed only at high scale.

**Files Affected**:
- `src/database/database.module.ts` (configure replicas)
- `src/health-info/services/get-response.ts` (use read replica)
- `ormconfig.ts` (add replica configuration)

**Implementation**:

```typescript
// ormconfig.ts
{
  type: 'mssql',
  replication: {
    master: {
      host: process.env.DB_HOST,
      port: 1433,
      username: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
    },
    slaves: [
      {
        host: process.env.DB_REPLICA_1_HOST,
        port: 1433,
        username: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME,
      },
      {
        host: process.env.DB_REPLICA_2_HOST,
        port: 1433,
        username: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME,
      }
    ]
  }
}

// TypeORM automatically routes:
// - SELECT queries → replicas (round-robin)
// - INSERT/UPDATE/DELETE → master
```

**Acceptance Criteria**:
- ✅ Read queries routed to replicas
- ✅ Write queries routed to master
- ✅ Replica lag monitored (<1 second)
- ✅ Automatic failover if replica unavailable
- ✅ Performance: Read latency reduced by 30%

**Rollout**:
- **Risk**: MEDIUM - Infrastructure change
- **Prerequisites**: Database replicas provisioned and synced
- **Deployment**: Configure replicas → test in staging → production
- **Rollback**: Remove replication config (route all to master)

---

### Task 4.5: Response Compression (OPTIONAL)

**Description**: Compress large questionnaire responses in cache and API responses.

**Why Optional**: Current response sizes are manageable. Compression is an optimization.

**Files Affected**:
- `src/common/interceptors/compression.interceptor.ts` (NEW)
- `src/database/cache/response.ts` (compress before caching)
- `package.json` (add compression library)

**Implementation**:

```typescript
// Install: npm install compression

// src/main.ts
import * as compression from 'compression';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Enable HTTP compression
  app.use(compression({
    threshold: 1024,  // Only compress responses >1KB
    level: 6          // Compression level (1-9)
  }));
  
  await app.listen(3000);
}

// Cache compression (for large responses)
async setCache(kitId: string, data: any): Promise<void> {
  const serialized = JSON.stringify(data);
  
  // Compress if >10KB
  const toCache = serialized.length > 10240
    ? await gzip(serialized)
    : serialized;
  
  await this.cacheManager.set(
    `questionnaire-response-${kitId}`,
    toCache,
    60 * 60
  );
}
```

**Acceptance Criteria**:
- ✅ HTTP responses compressed (gzip/brotli)
- ✅ Large cache entries compressed
- ✅ Compression transparent to clients
- ✅ Metrics track compression ratio
- ✅ Performance: Bandwidth reduced by 60%

**Rollout**:
- **Risk**: LOW - Standard HTTP feature
- **Deployment**: Deploy and monitor bandwidth metrics

---

### Task 4.6: Performance Profiling (OPTIONAL)

**Description**: Add detailed performance profiling for optimization insights.

**Why Optional**: Current performance is acceptable. Profiling helps find optimization opportunities.

**Files Affected**:
- `src/common/interceptors/performance.interceptor.ts` (NEW)
- `src/health-info/health-info.module.ts` (register interceptor)

**Implementation**:

```typescript
// src/common/interceptors/performance.interceptor.ts
@Injectable()
export class PerformanceInterceptor implements NestInterceptor {
  constructor(private logger: Logger) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const startTime = Date.now();
    const marks: { [key: string]: number } = {};

    // Add timing helper to request
    request.mark = (label: string) => {
      marks[label] = Date.now() - startTime;
    };

    return next.handle().pipe(
      tap(() => {
        const totalDuration = Date.now() - startTime;
        
        this.logger.log({
          message: 'Request performance',
          path: request.path,
          method: request.method,
          totalDuration,
          marks,
          requestId: request.requestId
        });
      })
    );
  }
}

// Usage in service
async execute(dto: AddQuestionResponseDto, req: any): Promise<void> {
  req.mark('validation_start');
  // ... validation logic
  req.mark('validation_end');
  
  req.mark('db_write_start');
  // ... database write
  req.mark('db_write_end');
  
  req.mark('cache_write_start');
  // ... cache write
  req.mark('cache_write_end');
}

// Log output:
{
  message: 'Request performance',
  path: '/v1/health-info/response',
  method: 'POST',
  totalDuration: 145,
  marks: {
    validation_start: 2,
    validation_end: 5,
    db_write_start: 6,
    db_write_end: 120,
    cache_write_start: 121,
    cache_write_end: 143
  }
}
```

**Acceptance Criteria**:
- ✅ Performance marks logged for all operations
- ✅ Slow requests (>500ms) highlighted
- ✅ Bottlenecks identified
- ✅ Dashboard shows performance breakdown
- ✅ No performance overhead from profiling (<1%)

**Rollout**:
- **Risk**: LOW - Logging only
- **Feature Flag**: `ENABLE_PERFORMANCE_PROFILING`
- **Deployment**: Enable in staging → analyze → optimize → production

---

## Exit Criteria (If Implemented)

Phase 4 is complete when **ALL IMPLEMENTED FEATURES** meet:

### Feature Requirements
- ✅ All implemented features working as designed
- ✅ No regressions in core functionality
- ✅ Feature flags allow disabling if needed

### Testing Requirements
- ✅ Unit tests for all new features
- ✅ Integration tests for collaboration features
- ✅ Performance tests show improvements

### Documentation Requirements
- ✅ Feature documentation published
- ✅ Client SDK updated (if applicable)
- ✅ Runbooks for new features

---

## Dependencies

### Prerequisites
- **Phase 3 complete** (all core features stable)
- Product approval for specific features
- Resources allocated

### External Coordination
- **Frontend team**: Coordinate on collaboration features
- **DevOps team**: Provision replicas if needed
- **Product team**: Prioritize features

---

## Estimated Timeline

| Task | Duration | Priority |
|------|----------|----------|
| 4.1: Advanced SSE | 2 days | OPTIONAL |
| 4.2: Cache warming | 1 day | OPTIONAL |
| 4.3: Conflict helpers | 2 days | OPTIONAL |
| 4.4: Read replicas | 2 days | OPTIONAL |
| 4.5: Compression | 0.5 day | OPTIONAL |
| 4.6: Performance profiling | 0.5 day | OPTIONAL |
| **Testing & Validation** | 1 day | If implemented |
| **TOTAL** | **2-9 days** | Depends on scope |

---

## ⚠️ IMPORTANT REMINDERS ⚠️

1. **This phase is OPTIONAL** - Do not implement unless explicitly requested
2. **Phases 1-3 are MANDATORY** - Complete them first
3. **Pick features selectively** - Don't implement all at once
4. **Measure impact** - Ensure features provide real value
5. **Can be deferred indefinitely** - System is complete after Phase 3

---

**Only proceed with Phase 4 if Phases 1-3 are complete AND product team requests specific features.**

