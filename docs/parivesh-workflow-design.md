# Design Doc: National Environmental Clearance Workflow (PARIVESH)

## 1. Requirement
Automate the approval process for environmental proposals involving 4 levels of government hierarchy.

## 2. Architecture Choice: Flowable Engine

### Decision: Why not hardcode `if-else` logic?
- The approval hierarchy changes based on state laws. Hardcoding would require redeployment for every policy change.
- **Flowable (BPMN)** allows dynamic workflow updates without downtime.

### Alternatives Considered
| Approach | Pros | Cons | Decision |
|----------|------|------|----------|
| Hardcoded if-else | Simple | Requires redeployment | ❌ Rejected |
| State Machine | Type-safe | Complex versioning | ❌ Rejected |
| Flowable BPMN | Dynamic, visual | Learning curve | ✅ **Chosen** |

## 3. Scale Estimation

### Volume
- **500+ proposals/day**
- **10,000 active workflows** at any time
- **4 approval levels** per proposal

### Storage
- **Metadata:** PostgreSQL (workflow state, approvals)
- **Documents:** AWS S3 (Cold storage moved to Glacier after 1 year)

### Throughput Calculation
```
500 proposals/day × 4 approval steps = 2000 state transitions/day
Average: 1.4 transitions/minute
Peak: 10 transitions/minute (morning hours)
```

## 4. Workflow Design

### Approval Hierarchy
```
Proposal Submitted
    ↓
Level 1: District Officer (2 days SLA)
    ↓
Level 2: State Committee (5 days SLA)
    ↓
Level 3: Regional Office (7 days SLA)
    ↓
Level 4: Central Authority (10 days SLA)
    ↓
Certificate Generated
```

### Role Resolution Logic
```java
public String resolveApprover(Proposal proposal, int level) {
    switch(level) {
        case 1: return getDistrictOfficer(proposal.getDistrict());
        case 2: return getStateCommittee(proposal.getState());
        case 3: return getRegionalOffice(proposal.getRegion());
        case 4: return "CENTRAL_AUTHORITY";
    }
}
```

### Dynamic Routing
Some proposals skip levels based on project type:
- **Low Risk:** Levels 1, 2 only
- **Medium Risk:** Levels 1, 2, 3
- **High Risk:** All 4 levels

**BPMN Gateway:**
```xml
<exclusiveGateway id="riskGateway">
  <sequenceFlow targetRef="level3" condition="${risk == 'HIGH'}"/>
  <sequenceFlow targetRef="certificate" condition="${risk == 'LOW'}"/>
</exclusiveGateway>
```

## 5. Bottleneck Resolution

### Issue: PDF generation blocking approval API
**Symptom:**
- Approval API taking 5-8 seconds
- Users waiting for certificate generation
- Timeout errors during peak hours

**Root Cause:**
PDF generation (2-3 seconds) was synchronous in the approval flow.

**Fix: Decoupled PDF Generation**
```
Before:
User → Approve API → Generate PDF → Return Response (8s)

After:
User → Approve API → Publish Kafka Event → Return "Accepted" (200ms)
                           ↓
                    PDF Worker → Generate PDF → Upload S3 → Email User
```

**Impact:**
- API response time: 8s → 200ms (96% improvement)
- User experience: Immediate feedback + email notification
- Scalability: PDF workers scale independently

## 6. Failure Handling

### Scenario 1: Approver unavailable
**Problem:** Officer on leave, workflow stuck
**Solution:** Auto-escalation after SLA breach
```
If no action in 2 days → Escalate to supervisor
If no action in 4 days → Escalate to regional head
```

### Scenario 2: Workflow version mismatch
**Problem:** Workflow updated mid-execution
**Solution:** Flowable migration strategy
```java
@Bean
public ProcessMigrationService migrationService() {
    return new ProcessMigrationService()
        .migrateProcessInstance(oldVersion, newVersion)
        .preserveVariables();
}
```

### Scenario 3: Database failure
**Problem:** PostgreSQL down, workflows can't progress
**Solution:**
- Read replica for queries
- Circuit breaker on writes
- Retry with exponential backoff

## 7. Data Model

### Workflow Variables
```json
{
  "proposalId": "ENV-2024-001",
  "applicantId": "USER-123",
  "projectType": "MINING",
  "riskLevel": "HIGH",
  "currentLevel": 2,
  "approvals": [
    {"level": 1, "officer": "OFF-456", "status": "APPROVED", "timestamp": "2024-01-15"}
  ]
}
```

### Database Schema
```sql
CREATE TABLE proposals (
    id VARCHAR(50) PRIMARY KEY,
    workflow_instance_id VARCHAR(100),
    status VARCHAR(20),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE approvals (
    id SERIAL PRIMARY KEY,
    proposal_id VARCHAR(50),
    level INT,
    approver_id VARCHAR(50),
    decision VARCHAR(20),
    comments TEXT,
    timestamp TIMESTAMP
);
```

## 8. Performance Optimization

### Before Optimization
- 500 proposals/day
- Average approval time: 25 days
- Manual intervention: 30% of cases

### After Optimization
- 500 proposals/day
- Average approval time: 20 days (20% reduction)
- Manual intervention: 10% of cases

### Key Improvements
1. **Parallel Approvals:** Multiple officers can review simultaneously
2. **Auto-Approval:** Low-risk proposals auto-approved at Level 1
3. **SLA Monitoring:** Alerts sent 1 day before deadline

## 9. Monitoring & Alerts

### Metrics Tracked
- Workflow completion rate
- Average time per level
- SLA breach count
- Error rate

### Dashboards
```
Grafana Dashboard:
- Active workflows by status
- Approval time distribution
- Bottleneck identification (which level is slowest)
```

## 10. Security Considerations

### Access Control
- Officers can only approve proposals in their jurisdiction
- Audit log for every action
- Digital signatures for final certificates

### Data Privacy
- PII encrypted at rest (AES-256)
- Sensitive documents in S3 with server-side encryption
- Access logs retained for 7 years (compliance)

## 11. Lessons Learned

### What Worked
✅ Flowable's visual designer helped non-technical stakeholders understand the flow
✅ Async PDF generation eliminated timeout issues
✅ SLA monitoring reduced approval delays

### What Could Be Better
⚠️ Workflow versioning was complex - needed better migration tooling
⚠️ Initial learning curve for BPMN syntax
⚠️ Testing workflows required dedicated test environment

## 12. Future Enhancements
- [ ] AI-based risk assessment (auto-classify proposals)
- [ ] Mobile app for officers (approve on-the-go)
- [ ] Blockchain for immutable audit trail
- [ ] Multi-language support (Hindi, regional languages)

---

**Related Resume Claim:**
✅ "Implemented dynamic, role-based workflows supporting 500+ daily proposals, reducing approval cycle time by 20%"
