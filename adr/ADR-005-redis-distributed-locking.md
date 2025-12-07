# ADR-005: Azure Managed Redis as Distributed Locking Mechanism

**Status**: ✅ Accepted
**Date**: 2024-11-26
**Authors**: Architecture Team
**Related**: ADR-001 (Quartz Scheduler), ADR-002 (AKS Deployment), ADR-003 (Azure SQL Job Store)

---

## Context

The Task Scheduling System uses Quartz Scheduler in clustered mode across multiple AKS pods. To prevent duplicate job execution, we need a distributed locking mechanism. Additionally, we require a high-performance distributed cache for job metadata, authentication tokens, and rate limiting.

**Functional Requirements**:
- Distributed locking to ensure only one pod executes each job
- Lock acquisition latency <5ms (p95) to meet job execution SLA
- Support for 10,000+ lock operations per second
- L2 distributed cache for job metadata (reduce database queries)
- Session storage for API authentication tokens
- Rate limiting counters (100 requests/minute per API key)

**Non-Functional Requirements**:
- High availability (99.95%+ uptime SLA)
- Low latency (<2ms for lock acquisition at p95)
- Data persistence to survive pod restarts
- Horizontal scalability to support growth
- Spring Boot ecosystem integration

**Strategic Requirements**:
- Future-proof solution (avoid migration work in 1-2 years)
- Aligned with Microsoft Azure's strategic direction
- Minimal operational overhead (prefer managed service)

**Constraints**:
- Must use Azure-managed service
- Budget: $300-500/month for caching infrastructure
- Team has Spring Boot expertise but limited Redis operations experience
- Must support private endpoints (no public internet access)

---

## Decision

**We will use Azure Managed Redis - Memory Optimized M10 (10GB) as the distributed locking and caching platform.**

**Configuration**:
- **Service**: Azure Managed Redis (Microsoft's current Redis offering)
- **Tier**: Memory Optimized M10
- **Memory**: 10GB
- **Throughput**: 25,000 operations/second
- **Persistence**: RDB snapshots every 15 minutes to managed disks
- **High Availability**: Zone-redundant deployment (99.99% SLA)
- **Authentication**: Microsoft Entra ID (service principal) + access keys
- **Network**: Private endpoint within VNet
- **Redis Version**: 7.4 (latest)
- **TLS Port**: 10000
- **Eviction Policy**: allkeys-lru (evict least recently used when memory full)
- **Total Cost**: ~$500/month

---

## Rationale

**Primary Drivers**:

1. **Future-Proof Architecture**
   - Azure Cache for Redis (Basic/Standard/Premium/Enterprise) is legacy service
   - Microsoft is migrating to Azure Managed Redis as strategic platform
   - Starting with current service avoids migration work in 1-2 years
   - Estimated migration cost avoided: $10,000-$20,000 (engineering + testing + risk)

2. **Superior Availability**
   - 99.99% SLA (52 minutes downtime/year) vs 99.95% legacy (4.4 hours/year)
   - Zone-redundant deployment with automatic failover
   - Half the expected downtime compared to legacy service

3. **Modern Authentication and Security**
   - Microsoft Entra ID support (service principals, managed identity)
   - Keyless authentication (credentials rotation automated)
   - TLS-only connections (port 10000, no non-TLS option)
   - Integration with Azure RBAC for fine-grained access control

4. **Latest Redis Features**
   - Redis 7.4 (latest version with newest features, security patches)
   - Legacy service stuck on Redis 6.x (missing 18+ months of improvements)
   - Access to Redis 7.x features: client-side caching, ACL improvements, performance enhancements

5. **Simplified Persistence Management**
   - RDB and AOF persistence to managed disks (included in pricing)
   - No separate Azure Storage account to manage
   - Avoids "soft delete" storage cost issues documented in legacy Premium tier
   - Active geo-replication compatible with persistence (not possible with legacy Premium)

6. **Performance and Capacity**
   - 25,000 ops/sec (5x current load of 5K ops/sec)
   - 10GB memory (current usage ~3GB, provides headroom for 2-year growth)
   - Sub-2ms lock acquisition latency (p95)
   - Cannot scale down (forces conservative sizing, preventing under-provisioning)

**Comparison Summary**:

| Criteria | Azure Managed Redis M10 (Chosen) | Azure Cache Premium P1 (Legacy) | Azure Managed Redis B1 | Redis on AKS | Azure SQL Locks |
|----------|----------------------------------|--------------------------------|------------------------|--------------|-----------------|
| **Future-Proof** | ✅ Microsoft's strategic direction | ❌ Legacy (phasing out) | ✅ Current service | N/A | N/A |
| **Latency** | ✅ <2ms p95 [3] | ✅ <2ms [4] | ✅ <3ms [3] | ⚠️ 5-10ms [5] | ❌ 20-50ms [5] |
| **Throughput** | ✅ 25K ops/sec [3] | ✅ 20K ops/sec [4] | ⚠️ 10K ops/sec [3] | ✅ 50K+ [5] | ❌ 1K ops/sec [5] |
| **SLA** | ✅ 99.99% [2] | ⚠️ 99.95% [2] | ⚠️ 99.9% [2] | ❌ Self-managed | ✅ 99.99% [2] |
| **Persistence** | ✅ RDB/AOF (managed disks) [3] | ⚠️ RDB (storage account) [4] | ✅ RDB/AOF (managed) [3] | ✅ Self-managed | ✅ ACID |
| **Geo-Replication** | ✅ Active (with persistence) [3] | ❌ Not with persistence [4] | ✅ Active [3] | ❌ Complex setup | ⚠️ Passive only |
| **Redis Version** | ✅ 7.4 (latest) [3] | ⚠️ 6.x (outdated) [4] | ✅ 7.4 [3] | ✅ 7.x [5] | N/A |
| **Auth** | ✅ Entra ID + keys [3] | ⚠️ Keys only [4] | ✅ Entra ID + keys [3] | ⚠️ Manual setup | ✅ Entra ID |
| **Storage Management** | ✅ Managed disks (included) [3] | ❌ Separate storage account [4] | ✅ Managed (included) [3] | ❌ Self-managed | Included |
| **Memory** | ✅ 10GB [3] | 6GB [4] | 1GB [3] | Configurable | N/A |
| **Cost** | ⚠️ $500/month [1] | ✅ $270/month [4] | ✅ $100/month [1] | ⚠️ $400/month [6] | Included in SQL |
| **Migration Risk** | ✅ None (current service) | ❌ Will require migration [6] | ✅ None | N/A | N/A |
| **Ops Overhead** | ✅ Fully managed | ✅ Fully managed | ✅ Fully managed | ❌ High | ✅ Managed |

**Decision Factors**:
- **Azure Managed Redis M10** selected for future-proof architecture, superior SLA, modern features
- **Azure Cache Premium P1** eliminated as legacy service being phased out (will require migration)
- **Azure Managed Redis B1** eliminated due to insufficient capacity (1GB too small for growth)
- **Redis on AKS** eliminated due to high operational overhead and complexity
- **Azure SQL Locks** eliminated due to 10x higher latency and 25x lower throughput

---

## Consequences

**Positive**:
1. ✅ **Future-Proof Platform**: Built on Microsoft's strategic Redis direction, avoids migration in 1-2 years
2. ✅ **Higher Availability**: 99.99% SLA (52 min/year downtime) vs 99.95% legacy (4.4 hours/year)
3. ✅ **Modern Authentication**: Entra ID integration enables keyless auth, service principals, managed identity
4. ✅ **Latest Redis Features**: Version 7.4 with newest performance improvements and security patches
5. ✅ **Simplified Operations**: Managed disks for persistence (no storage account to manage)
6. ✅ **Active Geo-Replication**: Works with persistence enabled (not possible with legacy Premium tier)
7. ✅ **Ultra-Low Latency**: <2ms lock acquisition (p95) meets job execution SLA requirements
8. ✅ **High Capacity**: 10GB provides 3x headroom for 2-year growth projection
9. ✅ **Multi-Purpose Platform**: Single service for locks, cache, rate limiting, sessions
10. ✅ **Spring Integration**: Native Spring Data Redis support with Lettuce client

**Negative**:
1. ❌ **Higher Cost**: $500/month vs $270 (legacy Premium P1) or $100 (Balanced B1)
   - **Mitigation**: $230/month delta ($2,760/year) is less than migration cost ($10K-20K avoided)
   - **Justification**: ROI achieved in 1-2 years by avoiding future migration
2. ❌ **No Scale Down**: Cannot reduce tier if over-provisioned (only scale up)
   - **Mitigation**: Sized conservatively for 2-year growth (10GB accommodates 5GB projected usage)
3. ❌ **Breaking Changes from Legacy**: Different DNS suffix, port 10000 vs 6380
   - **Mitigation**: Not applicable (new deployment, no migration from legacy service)
4. ❌ **TLS-Only Connections**: No non-TLS option
   - **Mitigation**: Best practice anyway (TLS required for security compliance)

**Trade-offs**:
- **What we gained**: Future-proof platform, better SLA, modern auth, Redis 7.4, simplified operations, active geo-replication
- **What we sacrificed**: $230/month higher cost, no scale-down flexibility

**Cost-Benefit Analysis**:
- **Annual Cost**: $500/month × 12 = $6,000/year
- **Avoided Migration Cost**: $10,000-$20,000 (engineering effort to migrate from legacy in 1-2 years)
- **ROI Period**: 6-12 months
- **Additional Benefits**: Higher SLA reduces incident response costs, Entra ID reduces security risks

---

## Alternatives Considered

**Alternative 1: Azure Cache for Redis Premium P1 (Legacy Service)**
- **Why Considered**:
  - Lower cost ($250 P1 + $20 storage = $270/month vs $500/month)
  - Proven track record (mature service with years of production use)
  - Good performance (20K ops/sec, <2ms latency)
  - Team familiarity (well-documented, extensive community knowledge)
- **Why Rejected**:
  - **Legacy Service**: Microsoft is phasing out in favor of Azure Managed Redis
  - **Inevitable Migration**: Will require migration to Azure Managed Redis in 1-2 years anyway
  - **Migration Overhead**: $10K-20K engineering cost + testing + deployment risk
  - **Lower SLA**: 99.95% vs 99.99% (twice the annual downtime: 4.4 hours vs 52 minutes)
  - **No Entra ID**: Access keys only (less secure, manual rotation required)
  - **Outdated Redis**: Version 6.x (18+ months behind latest features and security patches)
  - **Storage Complexity**: Separate Azure Storage account required for persistence
  - **Soft Delete Issue**: Microsoft docs warn "soft delete causes high storage costs" on persistence storage
  - **Geo-Replication Limitation**: Cannot use geo-replication with persistence enabled
  - **Technical Debt**: Known migration creates backlog item, distracts from feature development

**Alternative 2: Azure Managed Redis B1 Balanced (1GB)**
- **Why Considered**:
  - Lower cost ($100/month, within original $250-300 budget)
  - Future-proof (current Azure Managed Redis service)
  - Adequate throughput (10K ops/sec sufficient for current 5K load)
  - Same modern features (Redis 7.4, Entra ID, active geo-replication)
- **Why Rejected**:
  - **Insufficient Capacity**: 1GB memory too small for current needs (~3GB in use)
  - **No Growth Headroom**: Projected 5GB+ usage by Q4 2025 (20% quarterly growth)
  - **Eviction Risk**: LRU eviction under memory pressure would impact cache hit rate
  - **Cannot Scale Down**: Azure Managed Redis only scales up (would need to over-provision to M10 anyway)
  - **Cost Trap**: Starting small then scaling to M10 = $100/month wasted + migration effort
  - **Conservative Sizing**: Better to size for 2-year projection upfront (no scale-down option)

**Alternative 3: Self-Hosted Redis on AKS (Redis Cluster)**
- **Why Considered**:
  - Full control over configuration and Redis version
  - No per-operation pricing (compute cost only)
  - Potential cost savings (3 nodes × $133/month = $400/month)
  - Learning opportunity for team (Redis operations expertise)
- **Why Rejected**:
  - **High Operational Overhead**: Managing Redis Sentinel/Cluster, monitoring, patching, upgrades
  - **Complexity**: Configuring clustering, persistence, backups, disaster recovery
  - **Team Expertise Gap**: Limited Redis operations knowledge (Spring Boot focus)
  - **No Vendor Support**: Microsoft won't troubleshoot production issues
  - **Hidden Costs**: Engineering time for ops runbooks, incident response, maintenance
  - **Resource Cost**: 3+ pods required for HA ($400/month infrastructure) + engineering overhead
  - **Availability Risk**: Self-managed failover less reliable than Azure Managed Redis auto-failover
  - **Not Core Competency**: Team should focus on application features, not Redis operations

**Alternative 4: Azure SQL Database Locking (SELECT FOR UPDATE)**
- **Why Considered**:
  - Already using Azure SQL for Quartz job store (no additional service)
  - ACID guarantees (strongest consistency model)
  - Team has deep SQL expertise
  - Included in existing Azure SQL cost (no incremental spend)
- **Why Rejected**:
  - **10x Higher Latency**: 20-50ms (p95) vs <2ms Redis (unacceptable for job execution SLA)
  - **25x Lower Throughput**: 1K locks/sec vs 25K Redis (cannot handle peak load)
  - **Database Contention**: Lock operations compete with Quartz job store queries
  - **No Multi-Purpose**: Only supports locking (no caching, rate limiting, sessions)
  - **Scalability Limit**: Database vertical scaling expensive ($1,400/month for 16 vCores)
  - **Single Function**: Would still need separate solution for cache, rate limits, sessions

**Alternative 5: Delay Decision (Start with Azure Cache Premium, Migrate Later)**
- **Why Considered**:
  - Immediate cost savings ($270 vs $500 for initial months)
  - Well-understood legacy service (lower initial risk)
  - Defer migration complexity until forced by Microsoft
- **Why Rejected**:
  - **Inevitable Migration**: Microsoft's direction clear (Azure Managed Redis is future)
  - **Migration Cost**: $10K-20K engineering effort + testing + deployment risk
  - **Opportunity Cost**: Engineering time better spent on product features
  - **Technical Debt**: Known migration work creates backlog burden
  - **Distraction**: Future migration will disrupt roadmap, require regression testing
  - **ROI Math**: $230/month × 12-24 months = $2,760-$5,520 (less than migration cost)
  - **Risk**: Migration timelines uncertain (Microsoft could force migration sooner)

**Alternative 6: Redlock Algorithm (Distributed Lock Across 5 Redis Instances)**
- **Why Considered**:
  - Strongest lock safety guarantees (quorum-based consensus)
  - Tolerates up to 2 Redis instance failures
  - Research-backed algorithm (Martin Kleppmann analysis)
- **Why Rejected**:
  - **Extreme Operational Complexity**: Manage 5 separate Redis instances
  - **Very High Cost**: 5 × $500/month = $2,500/month (5x single instance)
  - **Higher Latency**: 10-20ms due to quorum consensus (p95)
  - **Overkill**: Job execution idempotency mitigates duplicate execution risk
  - **Maintenance Burden**: 5x monitoring, patching, incident response
  - **Network Complexity**: Ensure pods can reach all 5 instances with low latency
  - **Not Worth Trade-off**: Slightly better lock safety not worth 5x cost + 10x complexity

---

## Implementation Plan

**Phase 1: Infrastructure Provisioning** (Week 1)
1. Provision Azure Managed Redis M10 (Memory Optimized) via Terraform
2. Configure zone-redundant deployment across availability zones
3. Enable RDB persistence (15-minute snapshot interval to managed disks)
4. Create private endpoint in production VNet (subnet: `redis-subnet`)
5. Configure firewall rules (deny all public access, allow only private endpoint)
6. Set up Microsoft Entra ID authentication (service principal for Task Scheduler Service)
7. Configure eviction policy: `allkeys-lru`
8. Enable diagnostic logging to Azure Log Analytics
9. Create monitoring alerts (memory >80%, ops/sec >20K, latency >5ms)
10. Document connection details in Azure Key Vault

**Phase 2: Spring Boot Integration** (Week 2)
1. Add Spring Data Redis dependency (`spring-boot-starter-data-redis`)
2. Configure Lettuce connection factory:
   - Host: `<name>.<region>.redis.azure.net`
   - Port: 10000 (TLS)
   - SSL: Enabled (required for Azure Managed Redis)
   - Auth: Entra ID service principal via Azure Identity SDK
3. Implement `RedisLockService` using SET NX EX pattern:
   ```java
   public boolean tryLock(String jobId, int ttlSeconds) {
       String lockKey = "job:lock:" + jobId;
       return redisTemplate.opsForValue()
           .setIfAbsent(lockKey, "locked", ttlSeconds, TimeUnit.SECONDS);
   }
   ```
4. Add circuit breaker (Resilience4j) for Redis failures:
   - After 5 consecutive failures, open circuit (fallback to Azure SQL locks)
   - Half-open state after 60 seconds (test if Redis recovered)
5. Implement graceful degradation (SQL locks as fallback)
6. Unit tests for lock acquisition, release, timeout, circuit breaker

**Phase 3: Distributed Locking** (Week 3)
1. Integrate `RedisLockService` with Quartz `JobExecutionContext`
2. Acquire lock before job execution, release after completion
3. Implement lock TTL monitoring (alert if locks held >30 seconds = potential deadlock)
4. Add lock expiry safety (job timeout = lock TTL to prevent orphaned locks)
5. Load test lock contention scenarios:
   - 1,000 concurrent jobs competing for locks
   - Simulate Redis failures (validate circuit breaker fallback to SQL)
   - Measure lock acquisition latency under load (target p95 <5ms, p99 <10ms)
6. Document lock acquisition patterns in runbooks

**Phase 4: Two-Level Caching** (Week 4)
1. Implement L1 cache (Caffeine in-memory, 5-minute TTL):
   - Job metadata cache (reduce database queries)
   - Hot data cached locally in each pod
2. Implement L2 cache (Redis distributed, 5-minute TTL):
   - Shared cache across all pods
   - Fallback if L1 cache miss
3. Cache warming on pod startup:
   - Pre-load top 1,000 frequently accessed jobs
   - Reduces database load during pod restarts
4. Implement cache invalidation:
   - Explicit eviction on job updates (REST API PUT/DELETE)
   - Async refresh on job completion (publish event, update cache)
5. Monitor cache hit rate:
   - Target: >90% hit rate for job metadata
   - Alert if hit rate <85% (indicates cache warming issue or eviction pressure)

**Phase 5: Rate Limiting and Sessions** (Week 5)
1. Implement API rate limiting (Redis counters):
   - Key: `ratelimit:{apiKey}:{minute}`
   - INCR on each request, TTL = 60 seconds
   - Reject if count > 100 (enforce 100 req/min per API key)
2. Implement session storage (Redis strings):
   - Key: `session:{sessionId}`
   - Value: Encrypted session data (user ID, auth token, expiry)
   - TTL: 1 hour (session timeout)
3. Test rate limiting under load (simulate 200 req/min bursts)
4. Validate session recovery after pod restart (sessions persisted in Redis)

**Phase 6: Production Rollout** (Week 6)
1. Deploy to staging environment with synthetic load testing
2. Run soak test (24 hours with production-like traffic)
3. Validate metrics:
   - Lock acquisition latency <2ms (p95)
   - Cache hit rate >90%
   - Memory utilization <60% (4GB headroom)
   - No circuit breaker activations (Redis stable)
4. Gradual production rollout:
   - 10% traffic for 1 hour (monitor for issues)
   - 50% traffic for 4 hours
   - 100% traffic after successful soak
5. Monitor for 1 week before declaring production-ready
6. Document runbooks for Redis operations (failover, cache invalidation, lock debugging)

---

## Use Cases

**Use Case 1: Distributed Job Execution Locking**
- **Pattern**: `SET job:lock:{jobId} "locked" NX EX 30`
- **Purpose**: Ensure only one pod executes each scheduled job
- **TTL**: 30 seconds (job execution timeout)
- **Failure Mode**: Lock auto-expires if pod crashes, allows retry by another pod
- **Key Operations**:
  1. Before job execution: Attempt lock acquisition (SET NX)
  2. If acquired: Execute job
  3. If not acquired: Another pod already executing, skip
  4. After execution: Release lock (DEL) or let TTL expire

**Use Case 2: L2 Distributed Cache (Job Metadata)**
- **Pattern**: `GET/SET job:metadata:{jobId}` with 5-minute TTL
- **Purpose**: Reduce Azure SQL database queries for frequently accessed jobs
- **Eviction**: LRU when memory full (10GB capacity)
- **Hit Rate Target**: >90%
- **Key Operations**:
  1. Check L1 cache (Caffeine in-memory)
  2. If L1 miss: Check L2 cache (Redis)
  3. If L2 miss: Query Azure SQL, populate L1+L2 caches
  4. On job update: Invalidate L1+L2 caches explicitly

**Use Case 3: API Rate Limiting**
- **Pattern**: `INCR ratelimit:{apiKey}:{minute}` with 60-second TTL
- **Purpose**: Enforce 100 requests/minute per API key
- **Implementation**: Sliding window counter
- **Key Operations**:
  1. On API request: INCR counter
  2. If INCR result > 100: Reject with HTTP 429 (Too Many Requests)
  3. If INCR result ≤ 100: Allow request
  4. Counter auto-expires after 60 seconds

**Use Case 4: Session Storage**
- **Pattern**: `GET/SET session:{sessionId}` with 1-hour TTL
- **Purpose**: Store authentication tokens and user session data
- **Security**: Session data encrypted before storing in Redis
- **Key Operations**:
  1. On login: Generate session ID, store encrypted session data (SET with 1h TTL)
  2. On API request: Retrieve session (GET), validate auth token
  3. On logout: Delete session (DEL)
  4. Auto-expire: Sessions older than 1 hour deleted by Redis TTL

---

## Monitoring and Success Metrics

**Technical Metrics**:
- **Lock Acquisition Latency**: p50 <1ms, p95 <2ms, p99 <5ms (target: <10ms)
- **Cache Hit Rate**: >90% for job metadata (target: >85% minimum)
- **Redis Operations/Sec**: <20,000 average (headroom for 25K capacity)
- **Memory Utilization**: <70% (7GB of 10GB used, 3GB headroom)
- **Connection Count**: <500 concurrent connections (monitor for leaks)
- **Eviction Rate**: <100 evictions/minute (indicates memory pressure if higher)

**Business Metrics**:
- **Zero Duplicate Job Executions**: Distributed locking prevents duplicates
- **Job Execution Latency Improvement**: 20% reduction (due to caching reducing DB queries)
- **Database Query Reduction**: 30% fewer queries (cache offloading)
- **API Rate Limit Enforcement**: 100% compliance (all rate limit violations blocked)

**Operational Metrics**:
- **Redis Uptime**: 99.99% SLA compliance (52 min/year downtime budget)
- **Failover Recovery Time**: <30 seconds (zone-redundant automatic failover)
- **Cache Warming Time**: <2 minutes on pod restart (top 1,000 jobs pre-loaded)
- **Circuit Breaker Activations**: 0 (Redis stable, no fallback to SQL locks needed)

**Alerts** (Prometheus + Azure Monitor):
- **High Memory**: Memory >80% for >5 minutes (consider scale up to M20)
- **High Latency**: p95 latency >5ms for >5 minutes (investigate slow operations)
- **High Ops/Sec**: Operations >20K/sec for >10 minutes (approaching capacity limit)
- **Cache Hit Rate Drop**: Hit rate <85% for >10 minutes (cache warming issue or eviction pressure)
- **Connection Errors**: >10 connection failures in 5 minutes (network or auth issue)
- **Circuit Breaker Open**: Redis circuit breaker opened (fallback to SQL locks, high severity alert)

---

## Fallback Strategy

**If Azure Managed Redis becomes unavailable:**

**Detection** (Circuit Breaker Pattern):
1. Monitor Redis connection failures
2. After 5 consecutive failures within 30 seconds, open circuit breaker
3. Alert on-call engineer (PagerDuty high-severity)

**Fallback** (Graceful Degradation to Azure SQL Locks):
1. `DistributedLockService` switches to `SqlLockService` implementation
2. Use Azure SQL `SELECT FOR UPDATE` for pessimistic locking
3. Expected degradation:
   - Latency increases from 2ms to 20-50ms (10-25x slower)
   - Throughput decreases from 25K to 1K ops/sec (25x lower)
   - Job execution latency increases by 20-50ms per job

**Recovery** (Circuit Breaker Auto-Close):
1. Circuit breaker enters half-open state after 60 seconds
2. Test Redis connection with single request
3. If successful: Close circuit, resume Redis operations
4. If failed: Re-open circuit, remain in SQL fallback mode

**Cache Degradation**:
1. L1 cache (Caffeine) remains operational (pod-local)
2. L2 cache (Redis) unavailable during outage
3. Cache miss rate increases (all L2 misses fall through to Azure SQL)
4. Database query load increases by ~30% (cache offloading lost)

**Implementation** (Spring Boot):
```java
@Service
public class DistributedLockService {
    private final RedisLockService redisLockService;
    private final SqlLockService sqlLockService;
    private final CircuitBreaker circuitBreaker;

    public boolean acquireLock(String jobId, int ttlSeconds) {
        return circuitBreaker.executeSupplier(() -> {
            return redisLockService.tryLock(jobId, ttlSeconds);
        }, throwable -> {
            log.warn("Redis unavailable, falling back to SQL locks", throwable);
            metricsService.incrementCounter("lock.fallback.sql");
            return sqlLockService.tryLock(jobId, ttlSeconds);
        });
    }
}
```

---

## Migration Notes

**For New Deployments (This Architecture)**:
- ✅ Start directly with Azure Managed Redis (no migration needed)
- ✅ Configure from day 1 with correct port (10000), DNS suffix, Entra ID auth
- ✅ No breaking changes or migration work required

**If Migrating from Azure Cache for Redis (Legacy Service)**:

**Breaking Changes**:
1. **DNS Suffix**: `*.redis.cache.windows.net` → `*.<region>.redis.azure.net`
2. **TLS Port**: 6380 → 10000
3. **Non-TLS**: Port 6379 no longer supported (TLS-only)
4. **Connection Strings**: Must update all application configurations

**Migration Strategies**:
1. **RDB Export/Import** (5-15 min downtime):
   - Export RDB snapshot from legacy Redis
   - Import into Azure Managed Redis
   - Update connection strings, deploy application
   - Risk: Data written between export and cutover is lost

2. **Dual-Write Strategy** (zero downtime, 2x cost during migration):
   - Deploy application writing to both old and new Redis
   - Monitor for consistency
   - Cutover reads to new Redis
   - Remove old Redis writes
   - Cost: 2x Redis instances during migration (1-2 weeks)

3. **RIOT-X Tool** (programmatic migration):
   - Real-time replication from legacy to Azure Managed Redis
   - Lower risk of data loss
   - Complexity: Requires RIOT-X setup and monitoring

**Testing Requirements**:
- Full regression testing (connection string changes)
- Load testing (validate port 10000 TLS performance)
- Failover testing (zone-redundant behavior)

**Estimated Migration Effort** (if applicable):
- Engineering: 40-80 hours ($10,000-$20,000 at $250/hour)
- Testing: 20-40 hours
- Deployment risk: Medium (connection string changes impact all pods)
- Downtime: 5-15 minutes (RDB strategy) or 0 (dual-write strategy)

---

## References

### Comparison Data Sources
- [1] Azure Managed Redis pricing: https://azure.microsoft.com/en-us/pricing/details/cache/
- [2] Azure Managed Redis SLA: https://azure.microsoft.com/en-us/support/legal/sla/redis-cache/
- [3] Azure Managed Redis performance tiers: https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/managed-redis/managed-redis-overview#service-tiers
- [4] Azure Cache for Redis (Legacy) pricing: https://azure.microsoft.com/en-us/pricing/calculator/ (archived pricing for Premium P1 tier)
- [5] Redis OSS performance benchmarks: https://redis.io/docs/management/optimization/benchmarks/
- [6] Migration cost estimate: Based on 2-week engineering effort (2 senior engineers × 40 hours × $150/hour average loaded cost)

### Additional Resources
- Azure Managed Redis Overview: https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/managed-redis/managed-redis-overview
- Azure Managed Redis Migration Guide: https://learn.microsoft.com/en-us/azure/redis/migrate/migrate-overview
- Azure Cache for Redis Persistence (Legacy): https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-how-to-premium-persistence
- Redis Distributed Locks: https://redis.io/docs/manual/patterns/distributed-locks/
- Spring Data Redis: https://docs.spring.io/spring-data/redis/reference/
- Lettuce (Redis Client): https://lettuce.io/core/release/reference/

---

**Last Updated**: 2024-11-26
**Status**: ✅ Accepted