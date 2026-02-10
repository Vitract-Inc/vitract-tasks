# Phase 3: Maintainability & Scalability

## Goal

**Improve code quality, observability, and performance** to ensure long-term system health and scalability.

## Why This Phase Exists

After Phase 1 eliminates risks and Phase 2 establishes consistency, the system still needs:
- **Better resilience**: Idempotency, rate limiting, retry handling
- **Better observability**: Request tracing, PII redaction, comprehensive metrics
- **Better scalability**: Configurable limits, performance optimization
- **Better maintainability**: API versioning, code cleanup

**This phase ensures the system is production-ready for scale.**

---

## Included Issues (Mapped to Audit)

| Audit ID | Issue | Severity | Why Phase 3 |
|----------|-------|----------|-------------|
| **H1** | No idempotency tokens | HIGH | Duplicate writes on network retry |
| **M2** | No rate limiting | MEDIUM | Cache flooding, DoS risk |
| **M3** | PII in logs | MEDIUM | Privacy violation |
| **M5** | Hardcoded room size limit | MEDIUM | Scalability constraint |
| **L1** | No API versioning | LOW | Breaking changes impact clients |
| **L2** | No request tracing | LOW | Difficult debugging |

---

## Tasks

### Task 3.1: Implement Idempotency Middleware

**Description**: Add support for `Idempotency-Key` header to prevent duplicate operations on network retries.

**Why Phase 3**: **Prevents duplicate writes** when clients retry failed requests. Not Phase 1 because optimistic locking (Phase 1) already prevents most duplicates.

**Files Affected**:
- `src/common/middleware/idempotency.middleware.ts` (NEW)
- `src/common/decorators/idempotent.decorator.ts` (NEW)
- `src/health-info/health-info.controller.ts` (apply decorator)
- `src/health-info/health-info-sse.controller.ts` (apply decorator)

**Implementation**:

```typescript
// src/common/middleware/idempotency.middleware.ts
@Injectable()
export class IdempotencyMiddleware implements NestMiddleware {
  constructor(
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    private logger: Logger
  ) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const idempotencyKey = req.headers['idempotency-key'] as string;
    
    // Only apply to write operations
    if (!idempotencyKey || req.method === 'GET') {
      return next();
    }

    // Check if we've seen this key before
    const cacheKey = `idempotency:${idempotencyKey}`;
    const cachedResponse = await this.cacheManager.get(cacheKey);

    if (cachedResponse) {
      this.logger.log({
        message: 'Idempotent request detected',
        idempotencyKey,
        path: req.path,
        method: req.method
      });

      // Return cached response
      return res.status(cachedResponse.statusCode).json(cachedResponse.body);
    }

    // Intercept response to cache it
    const originalJson = res.json.bind(res);
    res.json = (body: any) => {
      // Cache response for 24 hours
      this.cacheManager.set(
        cacheKey,
        { statusCode: res.statusCode, body },
        60 * 60 * 24
      ).catch(err => this.logger.error('Failed to cache idempotent response', err));

      return originalJson(body);
    };

    next();
  }
}

// src/common/decorators/idempotent.decorator.ts
export const Idempotent = () => SetMetadata('idempotent', true);
```

**Usage**:
```typescript
// In controller
@Post('response')
@Idempotent()
async addResponse(@Body() dto: AddQuestionResponseDto) {
  // If client retries with same Idempotency-Key, cached response returned
  return this.service.execute(dto);
}
```

**Acceptance Criteria**:
- ✅ Idempotency-Key header supported on all write endpoints
- ✅ Duplicate requests return cached response (same status code and body)
- ✅ Cache TTL is 24 hours
- ✅ GET requests not affected by idempotency
- ✅ Integration test: send same request twice → verify same response
- ✅ Documentation updated with idempotency guidance

**Rollout**:
- **Risk**: LOW - Additive feature, no behavior change if header not provided
- **Deployment**: Deploy and document for client teams
- **Rollback**: Remove middleware

---

### Task 3.2: Add Rate Limiting

**Description**: Implement rate limiting to prevent cache flooding and DoS attacks.

**Why Phase 3**: **Prevents abuse**. Not Phase 1 because not an active data loss risk.

**Files Affected**:
- `src/common/guards/rate-limit.guard.ts` (NEW)
- `src/health-info/health-info.module.ts` (configure limits)
- `package.json` (add @nestjs/throttler)

**Implementation**:

```typescript
// Install: npm install @nestjs/throttler

// src/health-info/health-info.module.ts
@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60,        // Time window: 60 seconds
      limit: 100,     // Max requests per window
    }),
  ],
  controllers: [HealthInfoController],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class HealthInfoModule {}

// Per-endpoint customization
@Post('response')
@Throttle(10, 60)  // Max 10 writes per minute per kit
async addResponse(@Body() dto: AddQuestionResponseDto) {
  // ...
}
```

**Rate Limit Strategy**:

| Endpoint | Limit | Window | Scope |
|----------|-------|--------|-------|
| `POST /response` | 10 | 1 min | Per kit |
| `POST /submitted` | 5 | 1 min | Per kit |
| `POST /agreement` | 5 | 1 min | Per kit |
| `POST /add_bulk_responses` | 3 | 1 min | Per kit |
| `GET /:kitId` | 100 | 1 min | Per IP |

**Response on Rate Limit**:
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again in 45 seconds.",
    "statusCode": 429,
    "timestamp": "2026-02-10T14:30:00Z",
    "retryAfter": 45
  }
}
```

**Acceptance Criteria**:
- ✅ Rate limits enforced on all write endpoints
- ✅ 429 status code returned when limit exceeded
- ✅ Retry-After header included in response
- ✅ Limits configurable via environment variables
- ✅ Metrics track rate limit hits
- ✅ Integration test: exceed limit → verify 429 response

**Rollout**:
- **Risk**: LOW - Limits set high enough to not affect normal usage
- **Deployment**: Deploy with generous limits → tighten based on metrics
- **Rollback**: Remove throttler guard

---

### Task 3.3: Implement PII Redaction in Logs

**Description**: Automatically redact sensitive data (answer values, email, names) from application logs.

**Why Phase 3**: **Privacy compliance**. Not Phase 1 because audit logs (Phase 1) already protect data at rest.

**Files Affected**:
- `src/common/interceptors/logging.interceptor.ts` (enhance)
- `src/common/utils/pii-redactor.ts` (NEW)
- `src/database/cache/response.ts` (apply redaction)

**Implementation**:

```typescript
// src/common/utils/pii-redactor.ts
export class PiiRedactor {
  private static readonly PII_FIELDS = [
    'answerValue',
    'answer',
    'email',
    'recipientEmail',
    'firstName',
    'lastName',
    'phoneNumber',
    'address',
  ];

  static redact(obj: any): any {
    if (!obj || typeof obj !== 'object') {
      return obj;
    }

    if (Array.isArray(obj)) {
      return obj.map(item => this.redact(item));
    }

    const redacted = { ...obj };
    for (const key of Object.keys(redacted)) {
      if (this.PII_FIELDS.includes(key)) {
        redacted[key] = '[REDACTED]';
      } else if (typeof redacted[key] === 'object') {
        redacted[key] = this.redact(redacted[key]);
      }
    }

    return redacted;
  }
}

// Usage in logging
this.logger.log({
  message: 'Questionnaire response added',
  kitId: dto.kitId,
  categoryId: dto.categoryId,
  questionId: dto.questionResponse.questionId,
  // ❌ answer: dto.questionResponse.answer,  // DON'T LOG RAW
  answerRedacted: PiiRedactor.redact(dto.questionResponse.answer),  // ✅ SAFE
  userId: dto.userId,
});
```

**Acceptance Criteria**:
- ✅ All PII fields redacted in application logs
- ✅ Audit logs (database) still contain full data
- ✅ Error logs don't leak PII
- ✅ Redaction applied automatically via interceptor
- ✅ Unit tests verify redaction logic
- ✅ Log review confirms no PII leakage

**Rollout**:
- **Risk**: LOW - Logging only, no functional change
- **Deployment**: Deploy and review logs
- **Rollback**: Remove redaction (not recommended)

---

### Task 3.4: Add Request Tracing

**Description**: Implement request ID propagation across all layers for distributed tracing.

**Why Phase 3**: **Better debugging**. Not Phase 1 because monitoring (Phase 1) already provides basic observability.

**Files Affected**:
- `src/common/middleware/request-id.middleware.ts` (NEW)
- `src/common/interceptors/logging.interceptor.ts` (enhance)
- `src/queues/processors/health-info-dispatch.processor.ts` (propagate to queue)

**Implementation**:

```typescript
// src/common/middleware/request-id.middleware.ts
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Use client-provided ID or generate new one
    const requestId = req.headers['x-request-id'] || uuidv4();
    
    req['requestId'] = requestId;
    res.setHeader('X-Request-Id', requestId);
    
    next();
  }
}

// Propagate to logger
this.logger.log({
  message: 'Processing request',
  requestId: req['requestId'],
  path: req.path,
  method: req.method,
});

// Propagate to queue jobs
await this.queue.add('health-info-dispatch', {
  kitId,
  orderId,
  requestId: req['requestId'],  // Include in job data
});
```

**Acceptance Criteria**:
- ✅ Request ID generated for all incoming requests
- ✅ Request ID included in all log entries
- ✅ Request ID returned in response headers
- ✅ Request ID propagated to queue jobs
- ✅ Request ID included in error responses
- ✅ Documentation on using request IDs for debugging

**Rollout**:
- **Risk**: LOW - Observability only
- **Deployment**: Deploy and verify logs include request IDs
- **Rollback**: Remove middleware

---

### Task 3.5: Make Room Size Limit Configurable

**Description**: Move hardcoded max connection limit (10) to environment variable.

**Why Phase 3**: **Scalability**. Not Phase 1 because current limit is not causing issues.

**Files Affected**:
- `src/health-info/health-info-sse.bus.ts` (use config)
- `src/health-info/health-info.gateway.ts` (use config)
- `src/config/health-info.config.ts` (NEW)
- `.env.example` (document variable)

**Implementation**:

```typescript
// src/config/health-info.config.ts
export const healthInfoConfig = registerAs('healthInfo', () => ({
  maxRoomSize: parseInt(process.env.HEALTH_INFO_MAX_ROOM_SIZE, 10) || 10,
  cacheTtl: parseInt(process.env.HEALTH_INFO_CACHE_TTL, 10) || 3600,
  rateLimitPerMinute: parseInt(process.env.HEALTH_INFO_RATE_LIMIT, 10) || 100,
}));

// src/health-info/health-info-sse.bus.ts
@Injectable()
export class HealthInfoSseBus {
  constructor(
    @Inject(healthInfoConfig.KEY)
    private config: ConfigType<typeof healthInfoConfig>
  ) {}

  attach(kitId: string, userId: string, response: Response): void {
    const room = this.rooms.get(kitId) || new Map();
    
    if (room.size >= this.config.maxRoomSize) {  // ✅ Configurable
      throw new BadRequestException(
        `Room ${kitId} is full (max ${this.config.maxRoomSize} connections)`
      );
    }
    
    // ...
  }
}
```

**Environment Variables**:
```bash
# .env.example
HEALTH_INFO_MAX_ROOM_SIZE=10          # Max SSE/WS connections per kit
HEALTH_INFO_CACHE_TTL=3600            # Cache TTL in seconds (1 hour)
HEALTH_INFO_RATE_LIMIT=100            # Max requests per minute
```

**Acceptance Criteria**:
- ✅ Room size limit configurable via environment variable
- ✅ Default value is 10 (current behavior)
- ✅ Configuration documented in .env.example
- ✅ Invalid values (negative, non-numeric) handled gracefully
- ✅ Configuration loaded at startup and logged

**Rollout**:
- **Risk**: LOW - Default behavior unchanged
- **Deployment**: Deploy with default value
- **Rollback**: Revert to hardcoded value

---

### Task 3.6: Add API Versioning

**Description**: Add `/v1/` prefix to all health info endpoints for future-proofing.

**Why Phase 3**: **Prevents breaking changes**. Not Phase 1 because not an active risk.

**Files Affected**:
- `src/health-info/health-info.controller.ts` (update path)
- `src/health-info/health-info-sse.controller.ts` (update path)
- `src/main.ts` (add global prefix)
- `docs/API_MIGRATION_GUIDE.md` (NEW)

**Implementation**:

```typescript
// src/health-info/health-info.controller.ts
@Controller('v1/health-info')  // ✅ Versioned
export class HealthInfoController {
  // Endpoints now at /v1/health-info/*
}

// src/health-info/health-info-sse.controller.ts
@Controller('v1/sse/health-info')  // ✅ Versioned
export class HealthInfoSseController {
  // Endpoints now at /v1/sse/health-info/*
}

// Backward compatibility (temporary)
@Controller('health-info')  // ❌ Deprecated
export class HealthInfoControllerLegacy {
  constructor(private readonly v1Controller: HealthInfoController) {}

  @Get(':kitId')
  @Header('X-API-Deprecated', 'true')
  @Header('X-API-Sunset', '2026-06-01')
  async getLegacy(@Param('kitId') kitId: string) {
    // Delegate to v1 controller
    return this.v1Controller.get(kitId);
  }
}
```

**Migration Guide**:
```markdown
# API Migration Guide

## Health Info API v1

All health info endpoints have been versioned. Please update your clients:

### Changes
- `/health-info/*` → `/v1/health-info/*`
- `/sse/health-info/*` → `/v1/sse/health-info/*`

### Backward Compatibility
Legacy endpoints will continue to work until **June 1, 2026**.
Responses include deprecation headers:
- `X-API-Deprecated: true`
- `X-API-Sunset: 2026-06-01`

### Migration Checklist
- [ ] Update base URL in client SDK
- [ ] Test all endpoints with new paths
- [ ] Remove legacy endpoint usage before sunset date
```

**Acceptance Criteria**:
- ✅ All endpoints available at `/v1/` prefix
- ✅ Legacy endpoints still work (backward compatibility)
- ✅ Deprecation headers included in legacy responses
- ✅ Migration guide published
- ✅ Client teams notified
- ✅ Sunset date set (6 months out)

**Rollout**:
- **Risk**: LOW - Backward compatible
- **Deployment**: Deploy v1 endpoints → notify clients → sunset legacy after 6 months
- **Rollback**: Keep both versions running

---

## Exit Criteria

Phase 3 is complete when **ALL** of the following are true:

### Resilience Requirements
- ✅ Idempotency middleware implemented and tested
- ✅ Rate limiting enforced on all write endpoints
- ✅ Duplicate requests handled gracefully

### Observability Requirements
- ✅ Request IDs propagated through all layers
- ✅ PII redacted from application logs
- ✅ Audit logs still contain full data
- ✅ Distributed tracing working

### Scalability Requirements
- ✅ Room size limit configurable
- ✅ Cache TTL configurable
- ✅ Rate limits configurable
- ✅ Configuration documented

### API Requirements
- ✅ API versioning implemented
- ✅ Backward compatibility maintained
- ✅ Migration guide published
- ✅ Deprecation timeline communicated

---

## Dependencies

### Prerequisites
- **Phase 2 complete** (unified service layer, standardized errors)
- Configuration management system in place

### External Coordination
- **Frontend team**: Notify of API versioning and migration timeline
- **DevOps team**: Update environment variables in all environments

---

## Estimated Timeline

| Task | Duration | Dependencies |
|------|----------|--------------|
| 3.1: Idempotency middleware | 1 day | Phase 2 |
| 3.2: Rate limiting | 0.5 day | None (parallel) |
| 3.3: PII redaction | 0.5 day | None (parallel) |
| 3.4: Request tracing | 0.5 day | None (parallel) |
| 3.5: Configurable limits | 0.5 day | None (parallel) |
| 3.6: API versioning | 1 day | None (parallel) |
| **Testing & Validation** | 1 day | All tasks |
| **TOTAL** | **3-4 days** | |

---

**Phase 3 should only begin after Phase 2 exit criteria are met.**

---

## Exit Criteria

Phase 3 is complete when **ALL** of the following are true:

### Resilience Requirements
- ✅ Idempotency middleware implemented and tested
- ✅ Rate limiting enforced on all write endpoints
- ✅ Duplicate requests handled gracefully
- ✅ 429 responses include retry guidance

### Observability Requirements
- ✅ Request IDs propagated through all layers
- ✅ PII redacted from application logs
- ✅ Audit logs still contain full data (not redacted)
- ✅ Distributed tracing working end-to-end
- ✅ Logs include request ID, user ID, kit ID

### Scalability Requirements
- ✅ Room size limit configurable via environment variable
- ✅ Cache TTL configurable via environment variable
- ✅ Rate limits configurable via environment variable
- ✅ All configuration documented in .env.example
- ✅ Invalid configuration values handled gracefully

### API Requirements
- ✅ API versioning implemented (`/v1/` prefix)
- ✅ Backward compatibility maintained (legacy endpoints work)
- ✅ Migration guide published and shared with client teams
- ✅ Deprecation timeline communicated (6 months)
- ✅ Sunset date set and monitored

### Testing Requirements
- ✅ Integration tests verify idempotency behavior
- ✅ Load tests verify rate limiting works under stress
- ✅ Log review confirms no PII leakage
- ✅ Request tracing tested across REST/SSE/WebSocket/Queue
- ✅ Configuration changes tested in all environments

### Documentation Requirements
- ✅ Idempotency usage documented for clients
- ✅ Rate limit values documented
- ✅ Request ID usage documented for debugging
- ✅ API migration guide published
- ✅ Configuration reference updated

---

## Rollback Procedures

### If Idempotency Causes Issues

1. **Immediate**: Disable idempotency middleware
2. **Verify**: Requests process normally without caching
3. **Investigate**: Check for cache key collisions or stale responses
4. **Fix**: Address root cause
5. **Re-enable**: Deploy fix and re-enable

### If Rate Limiting Too Aggressive

1. **Immediate**: Increase rate limits via environment variable
2. **No deployment needed**: Configuration hot-reloaded
3. **Monitor**: Verify legitimate traffic not blocked
4. **Adjust**: Fine-tune limits based on actual usage patterns

### If PII Redaction Breaks Logging

1. **Check**: Verify redaction logic doesn't break log parsing
2. **Fix**: Update redaction patterns
3. **Fallback**: Temporarily disable redaction if critical (NOT RECOMMENDED)
4. **Audit**: Review logs for any PII leakage

### If API Versioning Breaks Clients

1. **Verify**: Legacy endpoints still working
2. **Check**: Deprecation headers present
3. **Communicate**: Notify affected clients
4. **Extend**: Extend sunset date if needed

---

## Dependencies

### Prerequisites
- **Phase 2 complete** (unified service layer, standardized errors)
- Configuration management system in place (environment variables)
- Monitoring infrastructure ready (Prometheus/Grafana)

### External Coordination
- **Frontend team**: Notify of API versioning and migration timeline
- **DevOps team**: Update environment variables in all environments
- **Security team**: Review PII redaction patterns
- **Client teams**: Coordinate migration to v1 endpoints

### Infrastructure Requirements
- Redis available for idempotency cache
- Environment variable management system
- Log aggregation system (for PII audit)

---

## Estimated Timeline

| Task | Duration | Dependencies |
|------|----------|--------------|
| 3.1: Idempotency middleware | 1 day | Phase 2 |
| 3.2: Rate limiting | 0.5 day | None (parallel) |
| 3.3: PII redaction | 0.5 day | None (parallel) |
| 3.4: Request tracing | 0.5 day | None (parallel) |
| 3.5: Configurable limits | 0.5 day | None (parallel) |
| 3.6: API versioning | 1 day | None (parallel) |
| **Testing & Validation** | 1 day | All tasks |
| **TOTAL** | **3-4 days** | |

**Note**: Tasks 3.2-3.6 can be done in parallel, reducing total time.

---

## Success Metrics

### Resilience Metrics
- **Idempotency hit rate**: >5% of requests (indicates retries being handled)
- **Rate limit hit rate**: <1% of requests (limits not too aggressive)
- **Duplicate operation prevention**: 100% (no duplicate writes with same idempotency key)

### Observability Metrics
- **Request ID coverage**: 100% of logs include request ID
- **PII leakage**: 0 instances in log audit
- **Trace completeness**: 100% of requests traceable end-to-end

### Scalability Metrics
- **Configuration flexibility**: All limits adjustable without code changes
- **Environment parity**: Configuration consistent across dev/staging/prod

### API Metrics
- **Legacy endpoint usage**: Declining over time
- **v1 endpoint adoption**: Increasing over time
- **Migration completion**: 100% by sunset date

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Idempotency cache fills up | LOW | MEDIUM | Set TTL to 24h, monitor cache size |
| Rate limits block legitimate traffic | MEDIUM | HIGH | Start with generous limits, tune based on metrics |
| PII redaction misses sensitive fields | MEDIUM | HIGH | Comprehensive field list, regular audits |
| Request ID overhead | LOW | LOW | Minimal performance impact (<1ms) |
| API versioning confuses clients | MEDIUM | MEDIUM | Clear migration guide, 6-month timeline |

---

## Monitoring & Alerts

### New Alerts to Configure

```yaml
# Idempotency
- alert: HighIdempotencyHitRate
  expr: rate(idempotency_cache_hit_total[5m]) > 0.2
  severity: warning
  message: "Idempotency hit rate >20%, investigate client retry behavior"

# Rate Limiting
- alert: HighRateLimitRejectionRate
  expr: rate(rate_limit_rejected_total[5m]) > 0.05
  severity: warning
  message: "Rate limit rejection rate >5%, limits may be too aggressive"

# PII Leakage (manual audit)
- alert: PIIAuditDue
  expr: days_since_last_pii_audit > 30
  severity: info
  message: "Monthly PII log audit is due"

# Request Tracing
- alert: MissingRequestIds
  expr: rate(requests_without_id_total[5m]) > 0.01
  severity: warning
  message: "Some requests missing request IDs"

# API Versioning
- alert: LegacyEndpointUsageHigh
  expr: rate(legacy_endpoint_requests_total[1h]) > 100
  severity: info
  message: "Legacy endpoints still heavily used, remind clients to migrate"
```

### Dashboards to Create

1. **Resilience Dashboard**:
   - Idempotency hit rate over time
   - Rate limit rejections by endpoint
   - Retry patterns by client

2. **Observability Dashboard**:
   - Request ID coverage
   - Log volume by severity
   - Trace completion rate

3. **API Migration Dashboard**:
   - v1 vs legacy endpoint usage
   - Migration progress by client
   - Days until sunset

---

## Post-Phase 3 Checklist

Before declaring Phase 3 complete, verify:

- [ ] All tasks implemented and tested
- [ ] All exit criteria met
- [ ] All monitoring/alerts configured
- [ ] All documentation published
- [ ] Client teams notified of changes
- [ ] Environment variables set in all environments
- [ ] Rollback procedures tested
- [ ] Team trained on new features
- [ ] Runbooks updated
- [ ] Phase 3 retrospective completed

---

**Phase 3 should only begin after Phase 2 exit criteria are met.**
**After Phase 3, the system is production-ready. Phase 4 is optional.**

