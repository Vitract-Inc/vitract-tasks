# Quick Start Guide - Health Information Backend Implementation

## 🚨 Start Here

**If you only read one thing, read this:**

The Health Information backend is **actively losing data** (24-hour Redis TTL, no database) and has a **broken REST GET endpoint** (always returns empty). 

**Phase 1 is MANDATORY and URGENT.**

---

## Day 1: Get Started

### Morning (2-4 hours)

1. **Read the audit** (if you haven't already):
   - `assessments/HEALTH_INFO_BACKEND_AUDIT.md`
   - Focus on "Critical Severity" section

2. **Read Phase 1 plan**:
   - `PHASE_1_RISK_ELIMINATION.md`
   - Understand all 6 tasks

3. **Set up your environment**:
   ```bash
   # Ensure you have access to:
   - SQL Server database (dev/staging/prod)
   - Redis instance (existing)
   - Monitoring infrastructure (Prometheus/Grafana)
   ```

4. **Create a feature branch**:
   ```bash
   git checkout -b feature/health-info-phase-1-risk-elimination
   ```

### Afternoon (4 hours)

5. **Task 1.1: Fix REST GET bug** (QUICK WIN - 2 hours):
   - File: `src/health-info/services/get-response.ts`
   - Remove line 14: `await this.cache.resetResponse(kitId);`
   - Add integration test
   - Create PR, get review, merge
   - **Deploy immediately** (low risk, high impact)

6. **Task 1.2: Start database schema** (2 hours):
   - Create TypeORM entities (4 files)
   - Generate migration script
   - Review with team

---

## Day 2-3: Database Foundation

### Task 1.2: Complete Database Schema

**Files to create**:
```
src/health-info/entity/
├── questionnaire-response.entity.ts
├── questionnaire-answer.entity.ts
├── questionnaire-agreement.entity.ts
└── questionnaire-audit-log.entity.ts

src/migrations/
└── YYYYMMDDHHMMSS-create-questionnaire-tables.ts
```

**Steps**:
1. Create entities (see Phase 1 doc for schema)
2. Generate migration: `npm run migration:generate -- -n CreateQuestionnaireTables`
3. Test in dev: `npm run migration:run`
4. Verify tables created in SQL Server
5. Test rollback: `npm run migration:revert`
6. Commit and push

### Task 1.6: Set Up Monitoring (Parallel)

**While waiting for DB review**:
1. Add Prometheus metrics to cache service
2. Create Grafana dashboard
3. Configure alerts
4. Test in dev environment

---

## Day 4-5: Write-Through Cache

### Task 1.3: Implement Write-Through Pattern

**This is the BIG task** - take your time, test thoroughly.

**Files to modify**:
- `src/database/cache/response.ts` (major refactor)
- `src/health-info/services/add-question-response.ts`
- `src/health-info/services/set-submitted.ts`
- `src/health-info/services/set-agreement.ts`

**Implementation checklist**:
- [ ] Database transaction wrapper
- [ ] Write to DB first (source of truth)
- [ ] Update cache second (performance layer)
- [ ] Audit log entry for every write
- [ ] Error handling (rollback on failure)
- [ ] Feature flag: `ENABLE_DB_WRITE_THROUGH`
- [ ] Integration tests (write → restart Redis → read → verify)
- [ ] Load tests (p95 latency <200ms)

**Deployment strategy**:
1. Deploy with flag OFF
2. Enable for 10% traffic
3. Monitor for 24 hours
4. Increase to 50%, then 100%

---

## Day 6: Locking & Validation

### Task 1.4: Add Optimistic Locking

**Files to modify**:
- `src/database/cache/response.ts` (add version check)
- `src/health-info/dto/*.dto.ts` (add version field)
- All controllers (return version in responses)

**Test scenario**:
```typescript
// Simulate concurrent writes
const promise1 = addResponse({ kitId: 'KIT123', version: 5, ... });
const promise2 = addResponse({ kitId: 'KIT123', version: 5, ... });

// One should succeed (version 6), one should get 409 Conflict
```

### Task 1.5: Add Submitted Validation

**Files to modify**:
- `src/health-info/services/add-question-response.ts`
- `src/health-info/services/set-agreement.ts`

**Test scenario**:
```typescript
// Submit questionnaire
await setSubmitted({ kitId: 'KIT123', submitted: true });

// Try to modify (should fail)
await addResponse({ kitId: 'KIT123', ... });
// Expected: 400 Bad Request
```

---

## Day 7: Testing & Validation

### Phase 1 Exit Criteria Checklist

**Functional**:
- [ ] REST GET returns actual data (not empty)
- [ ] All writes persist to database
- [ ] Write-through cache enabled for 100% traffic
- [ ] Submitted questionnaires cannot be modified
- [ ] Concurrent writes return 409 Conflict
- [ ] All writes create audit log entries

**Testing**:
- [ ] Integration tests pass
- [ ] Load test: p95 write latency <200ms
- [ ] Soak test: 24 hours with no data loss
- [ ] Redis restart test: data recoverable from DB

**Operational**:
- [ ] Monitoring dashboard shows metrics
- [ ] Alerts firing correctly in test
- [ ] Runbook created
- [ ] Database backup tested

**If all checked**: ✅ **Phase 1 Complete!**

---

## Week 2: Phase 2 (Standardization)

### Overview
- Unify SSE/WebSocket to use service layer
- Eliminate code duplication
- Standardize error responses

**Start with**: `PHASE_2_STANDARDIZATION.md`

**Key tasks**:
1. Refactor SSE controller (1 day)
2. Refactor WebSocket gateway (1 day)
3. Create bulk response service (0.5 day)
4. Standardize errors (1 day)
5. Document external API (0.5 day)

---

## Week 3: Phase 3 (Maintainability)

### Overview
- Add idempotency, rate limiting, PII redaction
- Add request tracing
- Make limits configurable
- Add API versioning

**Start with**: `PHASE_3_MAINTAINABILITY.md`

**Key tasks**:
1. Idempotency middleware (1 day)
2. Rate limiting (0.5 day)
3. PII redaction (0.5 day)
4. Request tracing (0.5 day)
5. Configurable limits (0.5 day)
6. API versioning (1 day)

---

## Week 4+: Phase 4 (Optional)

**Only if product team requests specific features.**

See: `PHASE_4_OPTIONAL_ENHANCEMENTS.md`

---

## Common Questions

### Q: What if I get stuck?

**A**: 
1. Check the detailed task description in the phase document
2. Review the audit for context
3. Ask the team for help
4. Refer to the rollback procedures if needed

### Q: Can I work on tasks in parallel?

**A**:
- **Phase 1**: Tasks 1.1 and 1.6 can be done in parallel with others
- **Phase 2**: Most tasks are sequential (need unified service layer first)
- **Phase 3**: Tasks 3.2-3.6 can be done in parallel

### Q: What if tests fail?

**A**:
1. Don't proceed to next task
2. Debug and fix
3. Re-run tests
4. Only move forward when all tests pass

### Q: What if performance degrades?

**A**:
1. Check monitoring dashboard
2. Identify bottleneck (DB? Cache? Network?)
3. Optimize (add indexes, increase connection pool, etc.)
4. If critical: disable feature flag and investigate

### Q: What if we need to rollback?

**A**:
- Each task has rollback procedures in the phase document
- Feature flags allow instant rollback without deployment
- Database migrations have down scripts
- Always test rollback in staging first

---

## Success Metrics

### After Phase 1
- **Data loss incidents**: 0
- **REST GET success rate**: 100%
- **Database write success rate**: >99.9%
- **Audit log coverage**: 100%

### After Phase 2
- **Code paths**: 3 controllers → 1 service layer
- **Business logic bypasses**: 0
- **Error format consistency**: 100%

### After Phase 3
- **Idempotency hit rate**: >5% (retries being handled)
- **Rate limit hit rate**: <1% (not too aggressive)
- **PII leakage**: 0 instances
- **Request ID coverage**: 100%

---

## Resources

### Documentation
- **Audit**: `assessments/HEALTH_INFO_BACKEND_AUDIT.md`
- **Phase 1**: `PHASE_1_RISK_ELIMINATION.md`
- **Phase 2**: `PHASE_2_STANDARDIZATION.md`
- **Phase 3**: `PHASE_3_MAINTAINABILITY.md`
- **Phase 4**: `PHASE_4_OPTIONAL_ENHANCEMENTS.md`
- **Summary**: `IMPLEMENTATION_SUMMARY.md`

### Code References
- **Cache Service**: `src/database/cache/response.ts`
- **Controllers**: `src/health-info/health-info*.controller.ts`
- **Services**: `src/health-info/services/*.ts`
- **Entities**: `src/health-info/entity/*.entity.ts`

### Tools
- **TypeORM**: Database migrations and entities
- **NestJS**: Framework documentation
- **Prometheus**: Metrics and monitoring
- **Grafana**: Dashboards and visualization

---

## Team Contacts

- **Backend Lead**: [Name] - Architecture questions
- **DevOps**: [Name] - Infrastructure, database access
- **QA**: [Name] - Testing strategy
- **Product**: [Name] - Requirements clarification
- **Compliance**: [Name] - Audit log, PII redaction review

---

## Next Steps

1. **Today**: Read audit and Phase 1 plan
2. **Tomorrow**: Fix REST GET bug (Task 1.1)
3. **This week**: Complete Phase 1
4. **Next week**: Start Phase 2
5. **Week 3**: Complete Phase 3
6. **Week 4+**: Optional enhancements (if requested)

---

**Remember**: Phase 1 is MANDATORY. Every day delayed is a day of active data loss risk.

**Let's get started! 🚀**

