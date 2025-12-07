# ADR-001: Selection of Quartz Scheduler

**Status**: ✅ Accepted
**Date**: 2024-11-15
**Authors**: Architecture Team
**Related**: N/A

---

## Context

The Task Scheduling System requires a robust, enterprise-grade job scheduling framework to manage scheduled transfers, payment reminders, and recurring payments. Key requirements:

**Functional Requirements**:
- Cron-based and interval-based scheduling
- Job persistence across service restarts
- Distributed execution across multiple instances (clustering)
- Job retry and error handling
- Support for 10,000+ concurrent scheduled jobs

**Non-Functional Requirements**:
- High availability (99.99% uptime)
- Horizontal scalability (support for 300 TPS peak job creation, 500 TPS peak execution)
- Low latency (p99 <200ms for job execution)
- Mature ecosystem with strong community support

**Constraints**:
- Must integrate with existing Java/Spring Boot microservices
- Must use Azure-managed data stores (no self-hosted infrastructure)
- Limited budget for licensing (prefer open-source)

---

## Decision

**We will use Quartz Scheduler (version 2.3+) as the job scheduling engine for the Task Scheduling System.**

**Deployment Mode**: Clustered mode with Azure SQL Database as the job store

**Integration**: Embedded within Spring Boot application using Spring Boot Quartz Starter

---

## Rationale

**Primary Drivers**:

1. **Enterprise Maturity and Adoption**
   - Quartz is a battle-tested, 20+ year old framework used by thousands of enterprises
   - Proven track record in financial services (banking, insurance, fintech)
   - Active community and extensive documentation

2. **Native Java/Spring Boot Integration**
   - First-class Spring Boot support via `spring-boot-starter-quartz`
   - Seamless dependency injection of Spring beans into job classes
   - Auto-configuration reduces boilerplate code

3. **Clustering and High Availability**
   - Built-in clustering support with database-backed job store (JDBC)
   - Automatic failover and load balancing across instances
   - No single point of failure (stateless scheduler instances)

4. **Feature Completeness**
   - Flexible scheduling: Cron, simple, calendar-based triggers
   - Job chaining and dependencies
   - Listener interfaces for job lifecycle events
   - Misfire handling for missed executions

5. **Azure SQL Compatibility**
   - Quartz JDBC job store works seamlessly with Azure SQL Database
   - Standard SQL dialect, no vendor lock-in
   - Leverages Azure SQL's high availability and geo-replication

**Comparison Summary**:

| Criteria | Quartz Scheduler | Spring Scheduler | Azure Functions (Timer Trigger) | Apache Airflow | BMC Control-M |
|----------|-----------------|------------------|--------------------------------|---------------|---------------|
| **Clustering** | ✅ JDBC-based (50+ nodes tested) [1] | ❌ None (single instance) [2] | ⚠️ Durable functions (limited HA) [3] | ✅ Celery/Redis backend [4] | ✅ Enterprise HA (multi-datacenter) [5] |
| **Persistence** | ✅ JDBC JobStore (SQL Server, PostgreSQL) [1] | ❌ In-memory only [2] | ✅ Azure Storage (queues/tables) [3] | ✅ PostgreSQL/MySQL [4] | ✅ Database-backed (Oracle, SQL Server) [5] |
| **Spring Integration** | ✅ spring-boot-starter-quartz (native) [6] | ✅ @Scheduled annotation (built-in) [2] | ⚠️ Requires Azure Functions runtime [3] | ❌ Separate Python/web system | ❌ REST API integration only [5] |
| **Scalability** | ✅ Horizontal (3-50 pods, load-balanced) [1] | ❌ Vertical scaling only [2] | ✅ Auto-scale (consumption plan) [3] | ✅ Horizontal (workers scale independently) [4] | ✅ 200K+ jobs/hour (enterprise) [5] |
| **Latency** | ✅ <10ms trigger latency (p95) [7] | ✅ <5ms (in-memory) [2] | ⚠️ 1-3s cold start latency [3] | ❌ 5-10s task overhead [4] | ⚠️ 100-500ms API overhead [5] |
| **Complexity** | ⚠️ Medium (clustering config, JDBC setup) [8] | ✅ Low (single annotation) [2] | ⚠️ Medium (Azure-specific deployment) [3] | ❌ High (DAG authoring, ops overhead) [4] | ❌ Very high (separate infra, agent install) [5] |
| **Cost** | ✅ Free (Apache 2.0 license) [1] | ✅ Free (included in Spring) [2] | ⚠️ $0.20/million executions + compute [3] | ✅ Free (Apache 2.0) + $400 infra [4] | ❌ $2,400-$28,800/year (SaaS) [5] |
| **Maturity** | ✅ 20+ years (since 2001) [1] | ✅ 15+ years (Spring 3.0+) [2] | ⚠️ 7 years (GA 2018) [3] | ⚠️ 9 years (Airbnb 2014, ASF 2016) [4] | ✅ 30+ years (since 1992) [5] |

**Decision Factors**:
- **Quartz** scored highest for clustering, persistence, and Spring integration with zero licensing cost
- **Spring Scheduler** eliminated due to lack of clustering (single point of failure)
- **Azure Functions** eliminated due to cold start latency and Azure-specific lock-in
- **Airflow** eliminated as overkill (designed for data pipelines, not general task scheduling)
- **Control-M** eliminated due to prohibitive cost and complexity (requires separate infrastructure)

---

## Consequences

**Positive**:
1. ✅ **High Availability**: Clustered Quartz with Azure SQL eliminates single point of failure
2. ✅ **Scalability**: Horizontal scaling by adding more scheduler instances (50+ pods)
3. ✅ **Familiarity**: Development team has strong Java/Spring Boot expertise
4. ✅ **Vendor Independence**: Open-source, no licensing costs, can migrate off Azure if needed
5. ✅ **Rich Ecosystem**: Extensive community plugins and integrations

**Negative**:
1. ❌ **Database Dependency**: Quartz relies heavily on Azure SQL for job store (potential bottleneck)
   - **Mitigation**: Use Azure SQL geo-replication for HA, optimize with read replicas
2. ❌ **Operational Complexity**: Requires managing Quartz clustering, misfires, lock contention
   - **Mitigation**: Comprehensive monitoring, alerting, and runbooks for common issues
3. ❌ **Limited Cloud-Native Features**: No built-in integration with Azure ecosystem (unlike Azure Functions)
   - **Mitigation**: Custom integration code for Confluent Kafka, Key Vault

**Trade-offs**:
- **What we gained**: Enterprise-grade reliability, horizontal scalability, mature ecosystem
- **What we sacrificed**: Simplicity of managed solutions (Azure Functions), cloud-native integrations

---

## Alternatives Considered

**Alternative 1: Spring @Scheduled Annotations**
- **Why Considered**: Simplest solution, zero external dependencies
- **Why Rejected**:
  - No persistence (jobs lost on pod restart)
  - No clustering (cannot run multiple instances safely)
  - Not suitable for production financial services workloads

**Alternative 2: Azure Functions with Timer Triggers**
- **Why Considered**: Fully managed, serverless, auto-scaling
- **Why Rejected**:
  - Cold start latency (~1 second) unacceptable for p99 <200ms requirement
  - Azure-specific lock-in (difficult to migrate to other clouds)
  - Consumption-based pricing unpredictable for high-volume workloads

**Alternative 3: Apache Airflow**
- **Why Considered**: Powerful workflow orchestration, widely adopted for data pipelines
- **Why Rejected**:
  - Overkill for simple scheduled jobs (designed for complex DAGs)
  - High operational overhead (requires separate Airflow cluster)
  - 5-10 second task overhead unacceptable for low-latency requirements

**Alternative 4: BMC Control-M**
- **Why Considered**:
  - Enterprise-grade workflow orchestration with 30+ years of proven reliability
  - Widely used in financial services for mission-critical batch processing
  - Handles massive scale (200,000+ jobs in production environments)
  - Comprehensive SLA management, monitoring, and predictive analytics
  - Hybrid cloud support (mainframe to cloud workflows)
- **Why Rejected**:
  - **Prohibitive Cost**: $2,400+/month minimum ($28,800+/year) vs free open-source Quartz
  - **Architectural Mismatch**: Control-M is an external system requiring separate infrastructure; Quartz embeds directly in Spring Boot application
  - **Overkill for Use Case**: Designed for complex cross-platform, cross-system workflows; our needs are simpler (embedded job scheduling within microservice)
  - **Operational Complexity**: Requires dedicated Control-M cluster, specialized expertise, additional infrastructure to maintain
  - **Integration Overhead**: Would require custom API integration code to connect Spring Boot apps to external Control-M system
  - **Latency**: 100-500ms overhead unacceptable for p99 <200ms requirement

---

## References

### Comparison Data Sources
- [1] Quartz Scheduler documentation and clustering guide: https://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigJDBCJobStoreClustering.html
- [2] Spring Framework @Scheduled documentation: https://docs.spring.io/spring-framework/reference/integration/scheduling.html
- [3] Azure Functions Timer trigger documentation: https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer
- [4] Apache Airflow documentation: https://airflow.apache.org/docs/
- [5] BMC Control-M product specifications and pricing: https://www.bmc.com/it-solutions/control-m.html
- [6] Spring Boot Quartz starter: https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.quartz
- [7] Quartz performance: Internal POC testing (p95 <10ms trigger latency with JDBC job store, 300 TPS peak job creation, 180 TPS sustained)
- [8] Team complexity assessment: Survey of 5 engineers (Quartz: 2-3 weeks ramp-up, Spring Scheduler: 1-2 days)

### Additional Resources
- Quartz Scheduler Documentation: https://www.quartz-scheduler.org/documentation/
- Spring Boot Quartz Integration: https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.quartz
- Quartz Clustering Guide: https://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigJobStoreTX.html
- Azure SQL Best Practices: https://docs.microsoft.com/en-us/azure/azure-sql/database/performance-guidance

---

**Last Updated**: 2025-01-16
**Status**: ✅ Accepted