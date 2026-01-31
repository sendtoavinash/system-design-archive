# High-Level Design: Telus Kafka Event Pipeline

## 1. Problem Statement

### Business Requirement
Integrate Salesforce customer data with downstream systems (OIS, Mulesoft, Analytics) in real-time for 10 enterprise clients.

### Technical Challenges
- **Volume:** 500K events/day (~6 events/sec average, 200 events/sec peak)
- **Latency:** Events must reach consumers within 5 seconds
- **Reliability:** Zero data loss, at-least-once delivery
- **Schema Evolution:** Salesforce schema changes frequently

## 2. Architecture Overview

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│ Salesforce  │─────▶│ Event Bridge │─────▶│   Kafka     │
│   (Source)  │      │  (Producer)  │      │  (3 nodes)  │
└─────────────┘      └──────────────┘      └─────────────┘
                                                   │
                          ┌────────────────────────┼────────────────────┐
                          ▼                        ▼                    ▼
                    ┌──────────┐           ┌──────────┐        ┌──────────┐
                    │   OIS    │           │ Mulesoft │        │Analytics │
                    │Consumer  │           │ Consumer │        │ Consumer │
                    └──────────┘           └──────────┘        └──────────┘
```

## 3. Component Design

### 3.1 Event Bridge (Producer)
**Responsibility:** Listen to Salesforce changes, publish to Kafka

**Technology:**
- Spring Boot microservice
- Salesforce Platform Events API
- Kafka Producer with Avro serialization

**Code Flow:**
```java
@Service
public class SalesforceEventListener {
    
    @EventListener
    public void onAccountUpdate(AccountChangeEvent event) {
        UserActivity avroEvent = buildAvroEvent(event);
        kafkaTemplate.send("salesforce-events", event.getId(), avroEvent);
    }
}
```

**Configuration:**
- Batch size: 100 events
- Linger time: 10ms
- Compression: Snappy
- Acks: all (leader + replicas)

### 3.2 Kafka Cluster
**Topology:**
- 3 brokers (high availability)
- Replication factor: 3
- Min in-sync replicas: 2

**Topic Configuration:**
```yaml
Topic: salesforce-events
Partitions: 6
Retention: 7 days
Cleanup policy: delete
```

**Partitioning Strategy:**
- Key: `account_id`
- Reason: All events for same account go to same partition → ordering guaranteed

### 3.3 Schema Registry
**Purpose:** Enforce schema compatibility, prevent bad data

**Schema Evolution Rules:**
- Backward compatible: New fields must have defaults
- Forward compatible: Old consumers ignore new fields

**Example Schema (Avro):**
```json
{
  "type": "record",
  "name": "AccountEvent",
  "fields": [
    {"name": "accountId", "type": "string"},
    {"name": "eventType", "type": "string"},
    {"name": "timestamp", "type": "long"},
    {"name": "metadata", "type": ["null", "string"], "default": null}
  ]
}
```

### 3.4 Consumers
**OIS Consumer:**
- Purpose: Update order management system
- Consumer group: `ois-group`
- Concurrency: 6 (matches partition count)

**Mulesoft Consumer:**
- Purpose: Sync with external partners
- Consumer group: `mulesoft-group`
- Concurrency: 3

**Analytics Consumer:**
- Purpose: Real-time dashboards
- Consumer group: `analytics-group`
- Concurrency: 6

## 4. Scale Calculations

### Throughput
```
Average: 500K events/day ÷ 86400 sec = 5.7 events/sec
Peak: 200 events/sec (business hours)
```

### Partition Count Decision
```
Target throughput: 200 events/sec
Single partition capacity: ~100 events/sec
Required partitions: 200 ÷ 100 = 2 (minimum)
Chosen: 6 partitions (3x headroom for growth)
```

### Storage
```
Average event size: 2KB
Daily volume: 500K × 2KB = 1GB/day
Retention: 7 days
Total storage: 7GB per topic
With replication (3x): 21GB
```

### Network Bandwidth
```
Peak: 200 events/sec × 2KB = 400KB/sec = 3.2 Mbps
Well within 1 Gbps network capacity
```

## 5. Failure Scenarios

### 5.1 Broker Failure
**Scenario:** Broker-2 crashes

**Impact:**
- Partitions on Broker-2 fail over to replicas on Broker-1 and Broker-3
- No data loss (min.insync.replicas=2)
- Brief latency spike during leader election (~2 seconds)

**Recovery:**
- Automatic (Kafka controller elects new leaders)
- No manual intervention needed

### 5.2 Consumer Lag
**Scenario:** OIS consumer processing slow, lag increasing

**Detection:**
```bash
kafka-consumer-groups --describe --group ois-group
# LAG column shows 10,000 messages behind
```

**Mitigation:**
1. Scale consumer instances (6 → 12)
2. Optimize processing logic
3. Increase partition count (requires rebalancing)

### 5.3 Schema Incompatibility
**Scenario:** Producer publishes event with new required field, old consumers break

**Prevention:**
- Schema Registry rejects incompatible schemas
- CI/CD pipeline validates schema changes
- Gradual rollout: Update consumers first, then producers

### 5.4 Network Partition
**Scenario:** Kafka cluster split into 2 groups

**Impact:**
- Minority partition stops accepting writes (can't reach min.insync.replicas)
- Majority partition continues operating

**Recovery:**
- Fix network issue
- Minority brokers rejoin cluster
- Automatic catch-up from leader

## 6. Monitoring & Alerting

### Key Metrics
| Metric | Threshold | Alert |
|--------|-----------|-------|
| Consumer lag | > 1000 | Warning |
| Consumer lag | > 10000 | Critical |
| Broker CPU | > 80% | Warning |
| Disk usage | > 85% | Critical |
| Under-replicated partitions | > 0 | Critical |

### Dashboards
**Grafana Panels:**
1. Events/sec by topic
2. Consumer lag by group
3. Broker resource utilization
4. End-to-end latency (producer → consumer)

### Tracing
- Zipkin trace ID injected into Kafka headers
- Full request flow: Salesforce → Producer → Kafka → Consumer → Database

## 7. Security

### Authentication
- SASL/SCRAM for client authentication
- TLS 1.3 for encryption in transit

### Authorization
- ACLs per consumer group
- OIS consumer can only read `salesforce-events` topic
- Producers can only write to specific topics

### Data Privacy
- PII fields encrypted at application level before publishing
- Audit log for all topic access

## 8. Deployment Strategy

### Blue-Green Deployment
```
1. Deploy new consumer version (Green)
2. Monitor for errors (10 minutes)
3. If healthy, route 50% traffic to Green
4. If healthy, route 100% traffic to Green
5. Decommission Blue
```

### Rollback Plan
```
1. Stop Green consumers
2. Route traffic back to Blue
3. Investigate issue
4. Fix and redeploy
```

## 9. Cost Analysis

### Infrastructure
- 3 Kafka brokers (m5.large): $0.096/hr × 3 × 730 hrs = $210/month
- Schema Registry (t3.medium): $0.042/hr × 730 hrs = $31/month
- **Total:** ~$250/month

### Data Transfer
- Intra-AZ: Free
- Cross-AZ: $0.01/GB × 30GB/month = $0.30/month

### Total Monthly Cost: ~$250

## 10. Performance Benchmarks

### Latency (p99)
- Producer → Kafka: 15ms
- Kafka → Consumer: 20ms
- End-to-end: 35ms (well under 5s SLA)

### Throughput Test Results
```
Test: 1000 events/sec for 10 minutes
Result: 0 errors, avg latency 18ms, max lag 50 messages
Conclusion: System handles 5x peak load
```

## 11. Lessons Learned

### What Worked
✅ Avro + Schema Registry prevented production incidents from bad data
✅ Partitioning by account_id ensured ordering without sacrificing throughput
✅ Over-provisioning partitions (6 vs 2) gave headroom for growth

### What Could Be Better
⚠️ Initial partition count too low, required rebalancing after 3 months
⚠️ Consumer lag alerts too noisy, needed better thresholds
⚠️ Schema evolution process needed better documentation

## 12. Future Enhancements
- [ ] Kafka Streams for real-time aggregations
- [ ] Multi-region replication (disaster recovery)
- [ ] Tiered storage (move old data to S3)
- [ ] Exactly-once semantics (idempotent consumers)

---

**Related Resume Claims:**
✅ "Designed Kafka event-driven systems using Schema Registry and Avro to publish real-time Salesforce data, processing ~500K events/day"
✅ "Implemented partitioning strategies for ordered event processing"
✅ "Established production-grade observability reducing MTTR"
