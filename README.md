# System Design Archive

High-Level (HLD) and Low-Level (LLD) design documents for distributed systems built over 6 years of backend engineering.

## 📚 Documents

### 1. [Telus Kafka Event Pipeline](docs/telus-kafka-hld.md)
**System:** Real-time event streaming for Salesforce integration
**Scale:** 500K events/day, 6 events/sec average, 200 events/sec peak
**Key Decisions:**
- Kafka partitioning strategy (account_id as key)
- Schema Registry for evolution
- 3-node cluster with replication factor 3

**Resume Claim:** ✅ "Designed Kafka event-driven systems processing ~500K events/day"

---

### 2. [PARIVESH Workflow Design](docs/parivesh-workflow-design.md)
**System:** National environmental clearance platform
**Scale:** 500+ proposals/day, 4-level approval hierarchy
**Key Decisions:**
- Flowable BPMN engine for dynamic workflows
- Async PDF generation (8s → 200ms API response)
- SLA monitoring with auto-escalation

**Resume Claim:** ✅ "Implemented dynamic workflows reducing approval cycle time by 20%"

---

### 3. [VAHAN Migration Strategy](docs/vahan-migration-strategy.md)
**System:** National vehicle registration system
**Scale:** 10K daily users, 50+ RTOs nationwide
**Key Decisions:**
- Strangler Fig pattern for gradual migration
- Dual-write for data consistency
- Phased rollout (9 months)

**Resume Claim:** ✅ "Migrated legacy systems improving performance by 25%"

---

## 🎯 Design Principles

### 1. Scale Estimation
Every design includes:
- Throughput calculations (requests/sec)
- Storage requirements (GB/day)
- Network bandwidth (Mbps)

**Example:**
```
500K events/day ÷ 86400 sec = 5.7 events/sec average
Peak: 200 events/sec
Partition count: 200 ÷ 100 = 2 (minimum), chose 6 for headroom
```

### 2. Failure Scenarios
Every design addresses:
- Component failures (broker crash, DB down)
- Network partitions
- Data inconsistencies
- Rollback strategies

### 3. Trade-offs
Every decision documented with:
- Alternatives considered
- Pros/cons table
- Why chosen approach won

**Example:**
| Approach | Pros | Cons | Decision |
|----------|------|------|----------|
| Hardcoded logic | Simple | Requires redeployment | ❌ |
| Flowable BPMN | Dynamic | Learning curve | ✅ |

## 📊 Common Patterns

### Event-Driven Architecture
- **Used in:** Telus Kafka, PARIVESH (PDF generation)
- **Benefits:** Decoupling, scalability, async processing
- **Trade-offs:** Eventual consistency, debugging complexity

### Strangler Fig Pattern
- **Used in:** VAHAN migration
- **Benefits:** Low-risk, gradual rollout
- **Trade-offs:** Longer timeline, dual-write complexity

### Cache-Aside Pattern
- **Used in:** Telus (Redis caching)
- **Benefits:** Reduced DB load (30%), lower latency
- **Trade-offs:** Stale data, cache invalidation complexity

## 🔍 How to Use These Docs

### For Interviews
1. **System Design Round:** Reference these as examples of real systems you've built
2. **Deep Dive:** Be ready to explain any decision (why Kafka over RabbitMQ?)
3. **Scale Questions:** Use the math (500K/day = 5.7/sec)

### For Resume Verification
Each doc maps to specific resume bullets. Interviewers can verify:
- ✅ You understand the scale (not just buzzwords)
- ✅ You made real trade-off decisions
- ✅ You handled production failures

### For Learning
These docs show:
- How to structure HLD documents
- What level of detail to include
- How to present trade-offs

## 🛠 Tools & Technologies

### Distributed Systems
- Kafka, RabbitMQ
- Redis, PostgreSQL
- Flowable BPMN

### Observability
- Prometheus, Grafana
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Zipkin (distributed tracing)

### Deployment
- Docker, Kubernetes
- Jenkins CI/CD
- Blue-Green deployments

## 📈 Impact Summary

| System | Scale | Performance Improvement | Business Impact |
|--------|-------|------------------------|-----------------|
| Telus Kafka | 500K events/day | 35ms p99 latency | Real-time data sync |
| PARIVESH | 500 proposals/day | 20% faster approvals | Reduced govt. delays |
| VAHAN | 10K users/day | 25% latency reduction | Nationwide rollout |

## 🎓 Interview Preparation

### Common Questions & Answers

**Q: How did you choose partition count for Kafka?**
A: Calculated peak throughput (200 events/sec), divided by single partition capacity (100 events/sec), added 3x headroom → 6 partitions.

**Q: How do you handle schema evolution?**
A: Schema Registry with backward compatibility rules. New fields must have defaults. Consumers updated before producers.

**Q: What happens if a Kafka broker crashes?**
A: Replicas on other brokers take over (replication factor 3, min.insync.replicas 2). Brief latency spike during leader election (~2s), no data loss.

**Q: How did you reduce approval time by 20%?**
A: 1) Async PDF generation (8s → 200ms), 2) Parallel approvals, 3) Auto-approval for low-risk cases, 4) SLA monitoring with alerts.

**Q: Why Strangler pattern over Big Bang migration?**
A: Lower risk. Can rollback individual modules. Gradual user training. Dual-write ensures data consistency during transition.

## 📚 Further Reading

### Books
- "Designing Data-Intensive Applications" - Martin Kleppmann
- "Building Microservices" - Sam Newman
- "Site Reliability Engineering" - Google

### Resources
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [Kafka Documentation](https://kafka.apache.org/documentation/)

---

**Note:** These are real production systems. Sensitive details (client names, exact numbers) have been generalized for confidentiality.
