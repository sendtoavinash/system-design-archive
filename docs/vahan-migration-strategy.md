# Design Doc: VAHAN 4.0 Legacy Migration Strategy

## 1. Context

### System Overview
**VAHAN** is the national vehicle registration system used by the Ministry of Road Transport & Highways (MORTH) across India.

### Migration Scope
- **From:** Monolithic J2EE application (VAHAN 3.0)
- **To:** Spring Boot microservices (VAHAN 4.0)
- **Scale:** 10,000+ daily users, 50+ RTOs (Regional Transport Offices)

## 2. Migration Strategy: Strangler Fig Pattern

### Why Strangler Pattern?
| Approach | Pros | Cons | Decision |
|----------|------|------|----------|
| Big Bang | Fast | High risk, downtime | ❌ Rejected |
| Parallel Run | Safe | 2x infrastructure cost | ❌ Rejected |
| Strangler Fig | Gradual, low risk | Longer timeline | ✅ **Chosen** |

### Pattern Overview
```
┌─────────────────────────────────────────────────┐
│              API Gateway (Routing)              │
└─────────────────────────────────────────────────┘
         │                              │
         ▼                              ▼
┌─────────────────┐          ┌─────────────────┐
│  Legacy System  │          │ New Microservice│
│   (VAHAN 3.0)   │          │   (VAHAN 4.0)   │
└─────────────────┘          └─────────────────┘
         │                              │
         └──────────────┬───────────────┘
                        ▼
                ┌───────────────┐
                │   Database    │
                └───────────────┘
```

**Phase 1:** Route 10% traffic to new service
**Phase 2:** Route 50% traffic
**Phase 3:** Route 100% traffic
**Phase 4:** Decommission legacy

## 3. Migration Phases

### Phase 1: Vehicle Registration Module (Month 1-2)
**Scope:**
- New vehicle registration
- Registration renewal
- Transfer of ownership

**Routing Rule:**
```java
if (request.path.startsWith("/api/v4/registration")) {
    route to NewService;
} else {
    route to LegacySystem;
}
```

**Success Criteria:**
- 0 data inconsistencies
- Latency < 500ms (same as legacy)
- 99.9% uptime

### Phase 2: License Management Module (Month 3-4)
**Scope:**
- Driving license issuance
- License renewal
- Endorsements

**Complexity:**
- Shared database with Phase 1
- Integration with Sarathi (license system)

### Phase 3: Challan & Enforcement (Month 5-6)
**Scope:**
- Traffic violation recording
- Fine payment
- Court integration

**Risk:**
- High transaction volume (20K/day)
- Payment gateway integration

### Phase 4: Reports & Analytics (Month 7-8)
**Scope:**
- Dashboard for RTOs
- MIS reports
- Data export

**Challenge:**
- Complex SQL queries
- Large data volumes (10 years of history)

## 4. Data Migration Strategy

### Dual-Write Pattern
During migration, writes go to both systems:

```java
@Transactional
public void registerVehicle(Vehicle vehicle) {
    // Write to new DB
    newVehicleRepository.save(vehicle);
    
    // Write to legacy DB (for backward compatibility)
    legacyVehicleRepository.save(vehicle);
    
    // Publish event for audit
    eventPublisher.publish(new VehicleRegisteredEvent(vehicle));
}
```

### Data Consistency Checks
**Nightly Reconciliation Job:**
```sql
SELECT v1.registration_number 
FROM new_db.vehicles v1
LEFT JOIN legacy_db.vehicles v2 ON v1.registration_number = v2.registration_number
WHERE v2.registration_number IS NULL;
```

**Alert if mismatch > 0.1%**

### Historical Data Migration
```
Step 1: Export legacy data (CSV)
Step 2: Transform schema (ETL pipeline)
Step 3: Load into new DB (batch insert)
Step 4: Validate checksums
```

**Timeline:** 3 months (parallel to Phase 1-3)

## 5. Rollback Plan

### Scenario: Critical bug in new service

**Immediate Rollback (< 5 minutes):**
```bash
# Update API Gateway routing
kubectl set env deployment/api-gateway ROUTE_TO_LEGACY=true

# Verify
curl http://api-gateway/health
```

**Data Rollback:**
- New service writes to both DBs → No data loss
- Legacy system continues from last known state

### Rollback Testing
- Monthly drill: Simulate failure, execute rollback
- Measure: Time to rollback, data consistency

## 6. Performance Optimization

### Before Migration (VAHAN 3.0)
- **Latency:** p95 = 800ms
- **Throughput:** 50 requests/sec
- **Database:** Single PostgreSQL instance

### After Migration (VAHAN 4.0)
- **Latency:** p95 = 600ms (25% improvement)
- **Throughput:** 200 requests/sec (4x improvement)
- **Database:** Read replicas + connection pooling

### Key Optimizations
1. **Connection Pooling:** HikariCP (max 50 connections)
2. **Caching:** Redis for frequently accessed data (vehicle details)
3. **Async Processing:** RabbitMQ for non-critical tasks (email notifications)

## 7. Risk Mitigation

### Risk 1: Data Loss During Migration
**Probability:** Medium
**Impact:** Critical
**Mitigation:**
- Dual-write to both systems
- Nightly reconciliation
- Point-in-time recovery (PITR) enabled

### Risk 2: Performance Degradation
**Probability:** Low
**Impact:** High
**Mitigation:**
- Load testing before each phase
- Auto-scaling (Kubernetes HPA)
- Circuit breakers on external calls

### Risk 3: User Resistance
**Probability:** High
**Impact:** Medium
**Mitigation:**
- Training sessions for RTO staff
- Phased rollout (pilot RTOs first)
- 24/7 support hotline

## 8. Testing Strategy

### Unit Tests
- Coverage: 85%+
- Tools: JUnit, Mockito

### Integration Tests
- Test dual-write consistency
- Test API Gateway routing
- Tools: TestContainers, WireMock

### Load Tests
- Simulate 10K concurrent users
- Tools: JMeter, Gatling
- Target: p95 < 500ms

### User Acceptance Testing (UAT)
- 5 pilot RTOs
- 2 weeks testing period
- Feedback loop with development team

## 9. Monitoring & Observability

### Metrics
| Metric | Threshold | Alert |
|--------|-----------|-------|
| Error rate | > 1% | Critical |
| Latency p95 | > 800ms | Warning |
| Database connections | > 80% | Warning |
| Dual-write mismatch | > 0.1% | Critical |

### Dashboards
**Grafana Panels:**
1. Request rate (legacy vs new)
2. Error rate comparison
3. Database query performance
4. Cache hit rate

### Logging
- Centralized: ELK Stack (Elasticsearch, Logstash, Kibana)
- Trace ID: Flows through legacy and new system
- Retention: 30 days

## 10. Success Metrics

### Technical Metrics
- ✅ 25% latency improvement
- ✅ 4x throughput increase
- ✅ 99.9% uptime maintained
- ✅ 0 data loss incidents

### Business Metrics
- ✅ 10K daily users supported
- ✅ 50+ RTOs migrated
- ✅ 95% user satisfaction (post-migration survey)

## 11. Lessons Learned

### What Worked
✅ Strangler pattern allowed gradual, low-risk migration
✅ Dual-write prevented data loss
✅ Pilot RTOs caught issues before nationwide rollout

### What Could Be Better
⚠️ Underestimated complexity of report migration (SQL queries)
⚠️ Needed more training for RTO staff
⚠️ API Gateway became bottleneck (needed optimization)

## 12. Timeline

```
Month 1-2: Vehicle Registration Module
Month 3-4: License Management Module
Month 5-6: Challan & Enforcement
Month 7-8: Reports & Analytics
Month 9: Decommission legacy system
```

**Total Duration:** 9 months

---

**Related Resume Claim:**
✅ "Migrated legacy systems to Spring Boot microservices, improving performance by 25% and supporting 10K daily users nationwide"
