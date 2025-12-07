# ADR-003: Azure SQL Database as Quartz Job Store

**Status**: ✅ Accepted
**Date**: 2024-11-22
**Authors**: Architecture Team
**Related**: ADR-001 (Quartz Scheduler), ADR-002 (AKS Deployment)

---

## Context

Quartz Scheduler requires a persistent job store for clustering and high availability. The job store stores job definitions, triggers, schedules, and execution state. Key requirements:

**Functional Requirements**:
- ACID transactions for job state consistency
- Pessimistic locking for distributed job execution
- Support for Quartz JDBC job store schema
- High write throughput (300 TPS peak job creation, frequent state updates)

**Non-Functional Requirements**:
- High availability (99.99% uptime)
- Low latency (<10ms for lock acquisition)
- Geo-replication for disaster recovery
- Automated backups with point-in-time restore

**Constraints**:
- Must use Azure-managed database (no self-hosted)
- Team has strong SQL expertise (prefer relational over NoSQL)
- Budget: ~$1,500/month for database

---

## Decision

**We will use Azure SQL Database (General Purpose tier, 8 vCores) as the Quartz job store with the following configuration:**

- **Tier**: General Purpose (vCore-based)
- **Compute**: 8 vCores (baseline), auto-scale to 16 vCores during peak
- **Storage**: 500GB with auto-growth
- **High Availability**: Zone-redundant deployment + geo-replication to paired region
- **Backup**: Automated daily backups, 35-day retention, point-in-time restore

---

## Rationale

**Primary Drivers**:

1. **Native Quartz Support**
   - Quartz JDBC JobStore has first-class support for SQL Server
   - Mature, battle-tested schema (QRTZ_* tables)
   - Pessimistic locking (SELECT FOR UPDATE) supported natively

2. **High Availability and Disaster Recovery**
   - Zone-redundant deployment provides 99.99% uptime SLA
   - Geo-replication to paired region (RPO <5 minutes, RTO <30 seconds)
   - Automated backups with 35-day retention and point-in-time restore

3. **Performance and Scalability**
   - General Purpose tier provides sufficient IOPS (5000+ IOPS for 8 vCores)
   - vCore-based scaling (scale up/down based on load)
   - Read replicas available for offloading read-heavy queries (job history)

4. **Azure Ecosystem Integration**
   - Managed Identity for authentication (no passwords)
   - Private endpoint for network isolation (no public internet access)
   - Azure Monitor integration for metrics and alerting
   - Transparent Data Encryption (TDE) for encryption at rest

5. **Operational Simplicity**
   - Fully managed (no database administration required)
   - Automated patching and updates
   - Built-in monitoring and query performance insights
   - Automated threat detection and vulnerability assessment

**Comparison Summary**:

| Criteria | Azure SQL | Azure Cosmos DB (SQL API) | Azure Database for PostgreSQL | Azure Table Storage |
|----------|-----------|--------------------------|-------------------------------|---------------------|
| **Quartz Support** | ✅ Native (JDBC JobStore) [1] | ❌ Requires custom adapter | ✅ Supported (PostgreSQL driver) [1] | ❌ No official support |
| **ACID Transactions** | ✅ Full ACID [2] | ⚠️ Eventual consistency [3] | ✅ Full ACID [4] | ❌ None |
| **Locking** | ✅ Pessimistic locks (SELECT FOR UPDATE) [2] | ❌ Optimistic concurrency [3] | ✅ Pessimistic locks [4] | ❌ Optimistic concurrency |
| **Latency** | ✅ <5ms same-region [5] | ✅ <10ms [3] | ✅ <5ms [4] | ✅ <10ms [6] |
| **Scalability** | ✅ Vertical (4-128 vCores) [7] | ✅ Horizontal (partitioning, unlimited) [3] | ✅ Vertical (2-64 vCores) [4] | ✅ Horizontal (unlimited) [6] |
| **HA/DR** | ✅ 99.99% SLA + geo-replication [8] | ✅ 99.99% SLA + multi-region writes [3] | ⚠️ 99.95% SLA + read replicas [4] | ⚠️ 99.9% SLA [6] |
| **Cost** | ⚠️ $1,400/month (8 vCores General Purpose) [7] | ❌ $2,000+/month (10K RU/s provisioned) [3] | ✅ $800/month (4 vCores General Purpose) [4] | ✅ $100/month (1TB storage) [6] |
| **Team Expertise** | ✅ Strong SQL Server skills (5 engineers, avg 3 years) [9] | ❌ Limited NoSQL experience [9] | ⚠️ Some PostgreSQL (2 engineers, 1 year) [9] | ⚠️ Limited [9] |
| **Backup/Restore** | ✅ Automated PITR (7-35 days retention) [2] | ✅ Automated PITR (continuous) [3] | ✅ Automated PITR (7-35 days) [4] | ⚠️ Manual (soft delete 90 days) [6] |

**Decision Factors**:
- **Azure SQL** scored highest for Quartz compatibility, ACID transactions, and team expertise
- **Cosmos DB** eliminated due to eventual consistency (incompatible with Quartz locking requirements)
- **PostgreSQL** viable alternative but team has stronger SQL Server expertise
- **Table Storage** eliminated due to lack of ACID transactions and pessimistic locking

---

## Consequences

**Positive**:
1. ✅ **Reliability**: 99.99% SLA with zone-redundancy and geo-replication
2. ✅ **Performance**: <5ms query latency for job state updates
3. ✅ **Operational Simplicity**: Fully managed, zero database administration
4. ✅ **Team Familiarity**: Leverages existing SQL Server expertise
5. ✅ **Security**: TDE encryption, Managed Identity authentication, private endpoint

**Negative**:
1. ❌ **Cost**: $1,400/month (vs $800 for PostgreSQL, $100 for Table Storage)
   - **Mitigation**: Use reserved capacity for 30% discount (~$1,000/month)
2. ❌ **Vertical Scaling Only**: Cannot scale horizontally (sharding complex for Quartz)
   - **Mitigation**: 8-16 vCores sufficient for peak load (300 TPS create, 500 TPS execute); current 8 vCores may be downsizable to 6 vCores with load monitoring
3. ❌ **Potential Bottleneck**: Database can become bottleneck under extreme load
   - **Mitigation**: Redis caching for read-heavy queries, read replicas for job history

**Trade-offs**:
- **What we gained**: Native Quartz support, ACID guarantees, high availability, team expertise
- **What we sacrificed**: Cost optimization (vs cheaper alternatives), horizontal scalability

---

## Alternatives Considered

**Alternative 1: Azure Cosmos DB (SQL API)**
- **Why Considered**: Globally distributed, elastic scalability, multi-region writes
- **Why Rejected**:
  - Eventual consistency incompatible with Quartz pessimistic locking
  - Requires custom Quartz JobStore implementation (significant dev effort)
  - Higher cost (~$2,000/month for required RU/s)

**Alternative 2: Azure Database for PostgreSQL**
- **Why Considered**: Lower cost ($800/month), Quartz supports PostgreSQL
- **Why Rejected**:
  - Team has limited PostgreSQL expertise (SQL Server preferred)
  - Slightly lower SLA (99.95% vs 99.99% for Azure SQL)
  - Decision was close; PostgreSQL remains viable fallback option

**Alternative 3: Azure Table Storage**
- **Why Considered**: Very low cost ($100/month), high throughput
- **Why Rejected**:
  - No ACID transactions (incompatible with Quartz requirements)
  - No pessimistic locking (optimistic concurrency only)
  - Requires custom Quartz JobStore implementation
  - High development risk and effort

---

## References

### Comparison Data Sources
- [1] Quartz JDBC JobStore documentation: https://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigJobStoreTX.html
- [2] Azure SQL Database documentation - Transactions and locking: https://learn.microsoft.com/en-us/azure/azure-sql/database/
- [3] Azure Cosmos DB consistency levels: https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels
- [4] Azure Database for PostgreSQL documentation: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/
- [5] Azure SQL Database performance: Internal testing (same-region latency <5ms for SELECT queries)
- [6] Azure Table Storage documentation: https://learn.microsoft.com/en-us/azure/storage/tables/
- [7] Azure SQL Database pricing: https://azure.microsoft.com/en-us/pricing/details/azure-sql-database/single/
- [8] Azure SQL Database SLA: https://azure.microsoft.com/en-us/support/legal/sla/azure-sql-database/
- [9] Team expertise assessment: Survey of 5 engineers (SQL Server: avg 3 years experience, PostgreSQL: 2 engineers with 1 year)

### Additional Resources
- Azure SQL Documentation: https://docs.microsoft.com/en-us/azure/azure-sql/
- Azure SQL Best Practices: https://docs.microsoft.com/en-us/azure/azure-sql/database/performance-guidance
- Quartz Clustering: https://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigJDBCJobStoreClustering.html

---

**Last Updated**: 2024-11-22
**Status**: ✅ Accepted