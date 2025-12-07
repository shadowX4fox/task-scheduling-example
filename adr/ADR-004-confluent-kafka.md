# ADR-004: Selection of Confluent Kafka for Event Streaming

**Status**: ✅ Accepted
**Date**: 2024-11-25
**Authors**: Architecture Team
**Related**: ADR-001 (Quartz Scheduler), ADR-002 (AKS Deployment)

---

## Context

The Task Scheduling System requires a reliable event streaming platform to publish job lifecycle events (created, started, completed, failed) for consumption by downstream systems including audit services, notification services, and analytics platforms. Key requirements:

**Functional Requirements**:
- Publish job lifecycle events with guaranteed delivery
- Support multiple independent consumer groups with different processing patterns
- Maintain event ordering per job (partition by jobId)
- Enable event replay for audit and debugging purposes
- Support high-throughput event streaming (10,000+ events/second)
- Provide at-least-once delivery guarantees

**Non-Functional Requirements**:
- High availability (99.99% uptime)
- Low latency (p95 <50ms for event publishing)
- Event retention for audit purposes (minimum 14 days)
- Schema evolution support for event payloads
- Horizontal scalability to support growing event volumes

**Integration Requirements**:
- Native support for Java/Spring Boot applications
- Integration with Azure Kubernetes Service (AKS) deployment
- Monitoring and observability tools for event streaming metrics
- Secure communication (TLS encryption, authentication)

**Constraints**:
- Must comply with financial services audit and compliance requirements
- Event data contains sensitive financial transaction information
- Limited operational overhead (prefer managed service over self-hosted)
- Budget considerations (balance cost with enterprise features)

---

## Decision

**We will use Confluent Kafka (Confluent Cloud or Confluent Platform) as the event streaming platform for the Task Scheduling System.**

**Deployment Mode**: Confluent Cloud (managed SaaS) or self-managed Confluent Platform on AKS

**Topic Strategy**: Dedicated topic `job-lifecycle-events` with 12 partitions, replication factor 3, 14-day retention

**Integration**: Spring Kafka library with Confluent Schema Registry for event schema management

---

## Rationale

**Primary Drivers**:

1. **Enterprise-Grade Reliability and Proven Track Record**
   - Apache Kafka is the industry-standard event streaming platform with 10+ years of production use
   - Confluent is founded by the original creators of Apache Kafka and provides enterprise support
   - Used extensively in financial services for mission-critical event streaming
   - Proven at massive scale (trillions of messages per day in production environments)

2. **Guaranteed Message Delivery and Ordering**
   - At-least-once delivery semantics with configurable acknowledgment (acks=all)
   - Partition-based ordering guarantees (events for same jobId always processed in order)
   - Durable persistence with replication prevents message loss
   - Consumer offset management enables exactly-once processing semantics

3. **Event Replay and Audit Capabilities**
   - Configurable retention period (14+ days) enables event replay for audit
   - Consumer groups can rewind offsets to reprocess historical events
   - Critical for financial services compliance and troubleshooting
   - Event sourcing pattern enables full audit trail of job executions

4. **Schema Management with Confluent Schema Registry**
   - Centralized schema repository for event payload definitions
   - Schema evolution with backward/forward compatibility checks
   - Prevents breaking changes to event consumers
   - Supports Avro, JSON Schema, and Protobuf formats

5. **High Throughput and Low Latency**
   - Optimized for high-throughput workloads (millions of messages/second)
   - Batching and compression (lz4) reduces network overhead
   - Asynchronous publishing with callback-based acknowledgment
   - Meets p95 <50ms latency requirement for event publishing

6. **Native Java/Spring Boot Integration**
   - Spring Kafka library provides first-class integration
   - Spring Boot auto-configuration simplifies setup
   - Reactive Kafka support for non-blocking event processing
   - Extensive community support and documentation

7. **Multi-Consumer Pattern Support**
   - Consumer groups enable independent consumption by multiple services
   - Each consumer group maintains its own offset (audit, notification, analytics can consume independently)
   - Fan-out pattern without source system knowing about consumers (loose coupling)

8. **Operational Maturity**
   - Confluent Cloud offers fully managed Kafka with 99.99% SLA
   - Comprehensive monitoring via Confluent Control Center and Prometheus
   - Automatic scaling, patching, and upgrades (managed service)
   - Professional support with enterprise SLAs

**Comparison Summary**:

| Criteria | Confluent Kafka | Azure Event Hubs | Azure Service Bus | RabbitMQ | Amazon Kinesis |
|----------|-----------------|------------------|-------------------|----------|----------------|
| **Event Ordering** | ✅ 100% in-order per partition [1] | ✅ Per-partition ordering [2] | ⚠️ FIFO queues only [3] | ⚠️ Queue-based (no guarantee) | ✅ Per-shard ordering [4] |
| **Replay Capability** | ✅ Full replay (offset rewind to any point) [1] | ✅ Consumer group offsets [2] | ❌ No replay (deleted after consumption) [3] | ❌ No replay | ⚠️ 24hr-365 day retention [4] |
| **Throughput** | ✅ 1M+ msgs/sec (per broker) [5] | ✅ 1M+ msgs/sec [2] | ⚠️ 2K msgs/sec (standard tier) [3] | ⚠️ 20K msgs/sec [6] | ✅ 1M+ records/sec [4] |
| **Latency** | ✅ <10ms p95 [5] | ✅ <10ms [2] | ⚠️ 50-100ms [3] | ⚠️ 20-50ms [6] | ⚠️ 200ms+ (batch-oriented) [4] |
| **Schema Management** | ✅ Confluent Schema Registry (built-in) [7] | ⚠️ External solution needed | ⚠️ External solution needed | ❌ No built-in support | ❌ No built-in support |
| **Multi-Consumer** | ✅ Native consumer groups (unlimited) [1] | ✅ Native consumer groups (20 max) [2] | ⚠️ Topic subscriptions (2K max) [3] | ✅ Exchanges/queues | ⚠️ Multiple applications |
| **Ecosystem** | ✅ 100+ Kafka Connect connectors, KSQL [8] | ⚠️ Azure-specific integrations | ⚠️ Azure-specific integrations | ✅ 50+ plugins [6] | ⚠️ AWS-specific integrations |
| **Managed Service** | ✅ Confluent Cloud (multi-cloud) [9] | ✅ Azure-native (PaaS) | ✅ Azure-native (PaaS) | ❌ Self-hosted only | ✅ AWS-native (PaaS) |
| **Operational Complexity** | ⚠️ Medium (self-hosted) / ✅ Low (Confluent Cloud) | ✅ Low (fully managed) | ✅ Low (fully managed) | ❌ High (self-hosted) | ✅ Low (fully managed) |
| **Cost** | ⚠️ $700+/month (Standard cluster) [9] | ✅ $0.028/million events [2] | ✅ $0.05/million ops [3] | ✅ Free (OSS) + $300 infra | ⚠️ $500+/month [4] |
| **Vendor Lock-in** | ✅ Open-source Apache Kafka core [1] | ❌ Azure-specific APIs | ❌ Azure-specific APIs | ✅ Open-source AMQP | ❌ AWS-specific APIs |
| **Financial Services Adoption** | ✅ 80%+ of Fortune 500 banks [10] | ✅ High (Azure customers) | ✅ High (Azure customers) | ⚠️ Medium | ✅ High (AWS customers) |

**Decision Factors**:
- **Confluent Kafka** selected for superior event replay, schema management, and multi-cloud portability
- **Azure Event Hubs** eliminated due to Azure-specific lock-in and lack of Confluent Schema Registry integration
- **Azure Service Bus** eliminated due to limited throughput and no replay capability (queue-based model)
- **RabbitMQ** eliminated due to high operational overhead (self-hosted) and lack of event replay
- **Amazon Kinesis** eliminated due to AWS-specific lock-in and batch-oriented latency

---

## Consequences

**Positive**:
1. ✅ **Event Replay**: Full audit trail with ability to reprocess historical events for compliance
2. ✅ **Loose Coupling**: Multiple consumers can independently process events without coordinating with publisher
3. ✅ **Schema Evolution**: Confluent Schema Registry prevents breaking changes to event consumers
4. ✅ **High Throughput**: Handles 10,000+ events/second with room for 10x growth
5. ✅ **Operational Simplicity**: Confluent Cloud managed service reduces operational burden
6. ✅ **Multi-Cloud Portability**: Can migrate between Azure, AWS, GCP without vendor lock-in
7. ✅ **Ecosystem**: Rich ecosystem of Kafka Connect connectors, KSQL for stream processing

**Negative**:
1. ❌ **Cost**: Confluent Cloud starts at $700/month (higher than Azure Event Hubs pay-per-use)
   - **Mitigation**: Evaluate cost vs. value of enterprise features; consider self-hosted Confluent Platform if cost prohibitive
2. ❌ **Operational Complexity (Self-Hosted)**: If self-hosting on AKS, requires managing Kafka brokers, ZooKeeper, Schema Registry
   - **Mitigation**: Use Confluent Cloud managed service to eliminate operational overhead
3. ❌ **Learning Curve**: Kafka has steeper learning curve than simple message queues
   - **Mitigation**: Invest in team training, leverage Spring Kafka abstractions
4. ❌ **Message Size Limits**: Default 1MB max message size (configurable but not recommended for large payloads)
   - **Mitigation**: Keep event payloads small (<100KB), use reference data pattern for large objects

**Trade-offs**:
- **What we gained**: Event replay, schema management, multi-consumer pattern, multi-cloud portability
- **What we sacrificed**: Lower cost of Azure-native solutions (Event Hubs, Service Bus), operational simplicity of fully managed PaaS

---

## Alternatives Considered

**Alternative 1: Azure Event Hubs**
- **Why Considered**:
  - Fully managed Azure-native service with Kafka-compatible API
  - Pay-per-use pricing model (potentially lower cost for low volume)
  - Native integration with Azure ecosystem (Monitor, Log Analytics, Event Grid)
  - 99.99% SLA with automatic scaling
- **Why Rejected**:
  - **Azure Lock-in**: Tightly coupled to Azure; difficult to migrate to other clouds
  - **Limited Schema Registry Support**: No native Confluent Schema Registry integration
  - **Kafka API Compatibility Gaps**: Not 100% compatible with Apache Kafka protocol (some Kafka features unsupported)
  - **Vendor-Specific Tooling**: Requires learning Azure-specific tools instead of standard Kafka ecosystem
  - **Long-Term Strategic Risk**: Event streaming is critical infrastructure; prefer open-source standard over proprietary

**Alternative 2: Azure Service Bus**
- **Why Considered**:
  - Fully managed Azure service with strong enterprise features
  - Built-in dead-letter queue, scheduled delivery, duplicate detection
  - Excellent integration with Azure ecosystem
  - Lower operational complexity than Kafka
- **Why Rejected**:
  - **No Event Replay**: Queue-based model deletes messages after consumption (not suitable for audit/replay)
  - **Lower Throughput**: Designed for 1000s msgs/sec, not millions (may not scale with growth)
  - **Limited Multi-Consumer**: Topic subscriptions less flexible than Kafka consumer groups
  - **Higher Latency**: 50-100ms typical latency vs. <10ms for Kafka
  - **Architectural Mismatch**: Queue-based model doesn't fit event streaming use case

**Alternative 3: RabbitMQ**
- **Why Considered**:
  - Open-source, mature message broker (10+ years production use)
  - Flexible routing with exchanges, bindings, and queues
  - Strong Spring AMQP integration
  - Lower infrastructure cost (self-hosted on AKS)
- **Why Rejected**:
  - **No Event Replay**: Messages deleted after acknowledgment (no audit trail)
  - **High Operational Overhead**: Requires managing RabbitMQ cluster on AKS (clustering, persistence, monitoring)
  - **Lower Throughput**: Optimized for 10,000s msgs/sec, not millions
  - **No Schema Management**: No built-in schema registry (would need external solution)
  - **Not Purpose-Built for Event Streaming**: Designed for traditional messaging, not event logs

**Alternative 4: Amazon Kinesis**
- **Why Considered**:
  - Fully managed AWS service for real-time event streaming
  - Proven at massive scale (used by Amazon internally)
  - Automatic scaling with pay-per-use pricing
  - Good replay capability (24hr-365 day retention)
- **Why Rejected**:
  - **AWS Lock-in**: AWS-specific service; cannot run on Azure or other clouds
  - **Batch-Oriented Latency**: 200ms+ typical latency due to batching (p95 <50ms requirement not met)
  - **Sharding Complexity**: Manual shard management more complex than Kafka partition management
  - **Limited Ecosystem**: Smaller ecosystem compared to Kafka (fewer connectors, tools)
  - **Not Azure-Native**: Current architecture on Azure; cross-cloud complexity not justified

**Alternative 5: Self-Hosted Apache Kafka (Open Source)**
- **Why Considered**:
  - Zero licensing cost (open-source)
  - Full control over configuration and infrastructure
  - Same Kafka functionality as Confluent
- **Why Rejected**:
  - **High Operational Overhead**: Managing Kafka cluster, ZooKeeper, upgrades, patching, monitoring on AKS
  - **No Schema Registry**: Would need to self-host Confluent Schema Registry or implement alternative
  - **No Enterprise Support**: Rely on community support instead of vendor SLA
  - **Maintenance Burden**: Team must handle security patches, version upgrades, capacity planning
  - **TCO Consideration**: Engineering time cost for operations may exceed Confluent Cloud licensing

---

## Implementation Plan

**Phase 1: Confluent Cloud Setup** (Week 1)
1. Create Confluent Cloud account and cluster (Standard tier, multi-zone)
2. Configure topic `job-lifecycle-events` (12 partitions, replication factor 3, 14-day retention)
3. Set up Confluent Schema Registry and register initial event schemas
4. Configure API keys for producer and consumer authentication
5. Integrate Confluent Cloud with Azure Key Vault for secret management

**Phase 2: Producer Integration** (Week 2)
1. Add Spring Kafka dependency to Task Scheduler Service
2. Implement event producer with Confluent Schema Registry integration
3. Define event schemas (Avro) for job lifecycle events
4. Configure idempotent producer with acks=all for reliability
5. Implement error handling and retry logic for failed publishes

**Phase 3: Consumer Implementation** (Week 3-4)
1. Implement audit service consumer (consume all events, store in audit database)
2. Implement notification service consumer (consume completed/failed events, send notifications)
3. Implement analytics service consumer (batch processing for reporting)
4. Configure consumer groups with appropriate offset commit strategies
5. Implement dead-letter queue for failed message processing

**Phase 4: Monitoring and Observability** (Week 5)
1. Configure Prometheus JMX Exporter for Kafka client metrics
2. Create Grafana dashboards for producer/consumer monitoring
3. Set up alerts for consumer lag, producer errors, under-replicated partitions
4. Integrate with Confluent Control Center for operational monitoring
5. Document runbooks for common Kafka operational tasks

**Phase 5: Production Rollout** (Week 6)
1. Deploy to staging environment and conduct load testing
2. Validate event ordering, replay, and schema evolution
3. Perform failover testing (broker failure, network partition)
4. Gradual rollout to production (10%, 50%, 100% traffic)
5. Monitor for 1 week before declaring production-ready

---

## Monitoring and Success Metrics

**Technical Metrics**:
- Producer send rate: Target 10,000 msgs/sec, measure actual throughput
- Producer latency: p95 <50ms, p99 <100ms
- Consumer lag: <10,000 messages per consumer group
- Event processing success rate: >99.9%
- Schema evolution events: Track backward/forward compatibility validations

**Business Metrics**:
- Audit coverage: 100% of job executions recorded in audit log
- Notification delivery time: <5 minutes from job completion to customer notification
- Compliance reporting: Ability to generate audit reports for any time period

**Operational Metrics**:
- Confluent Cloud uptime: 99.99% SLA compliance
- Incident response time: <15 minutes for critical Kafka issues
- Schema Registry availability: 99.95%

---

## References

### Comparison Data Sources
- [1] Apache Kafka documentation - Guarantees: https://kafka.apache.org/documentation/#semantics
- [2] Azure Event Hubs documentation: https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas
- [3] Azure Service Bus pricing and quotas: https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quotas
- [4] Amazon Kinesis Data Streams documentation: https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html
- [5] Confluent Kafka performance benchmarks: https://www.confluent.io/blog/kafka-fastest-messaging-system/
- [6] RabbitMQ performance specifications: https://www.rabbitmq.com/blog/2012/04/25/rabbitmq-performance-measurements-part-2
- [7] Confluent Schema Registry documentation: https://docs.confluent.io/platform/current/schema-registry/index.html
- [8] Kafka Connect ecosystem: https://www.confluent.io/hub/
- [9] Confluent Cloud pricing: https://www.confluent.io/confluent-cloud/pricing/
- [10] Financial services adoption: Confluent customer case studies (https://www.confluent.io/customers/) - 80%+ of Fortune 500 banks use Kafka

### Additional Resources
- Confluent Kafka Documentation: https://docs.confluent.io/
- Apache Kafka Documentation: https://kafka.apache.org/documentation/
- Spring Kafka Integration: https://docs.spring.io/spring-kafka/reference/html/
- Kafka Best Practices for Financial Services: https://www.confluent.io/resources/kafka-financial-services/
- Event-Driven Architecture Patterns: https://martinfowler.com/articles/201701-event-driven.html

---

**Last Updated**: 2024-11-25
**Status**: ✅ Accepted