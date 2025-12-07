# ADR-002: Azure Kubernetes Service (AKS) as Deployment Platform

**Status**: ✅ Accepted
**Date**: 2024-11-20
**Authors**: Architecture Team
**Related**: ADR-001 (Quartz Scheduler selection)

---

## Context

The Task Scheduling System requires a container orchestration platform for deploying and managing the Quartz-based microservices. Key requirements:

**Functional Requirements**:
- Container orchestration with auto-scaling
- Service discovery and load balancing
- Rolling deployments with zero downtime
- Health checks and automatic recovery

**Non-Functional Requirements**:
- High availability (99.99% uptime SLA)
- Horizontal scalability (3-50 pod replicas)
- Multi-zone deployment for fault tolerance
- Integration with Azure ecosystem (SQL, Redis, Confluent Kafka)

**Constraints**:
- Must use Azure cloud (corporate mandate)
- Team has limited Kubernetes expertise (need managed solution)
- Budget: ~$2,000/month for compute infrastructure

---

## Decision

**We will deploy the Task Scheduling System on Azure Kubernetes Service (AKS) with the following configuration:**

- **Cluster**: Multi-zone AKS cluster (3 availability zones)
- **Node Pools**:
  - System node pool: 3x Standard_D4s_v3 (4 vCPU, 16GB RAM)
  - Application node pool: 6x Standard_D8s_v3 (8 vCPU, 32GB RAM) with auto-scaling
- **Networking**: Azure CNI with network policies
- **Ingress**: Azure Application Gateway Ingress Controller

---

## Rationale

**Primary Drivers**:

1. **Managed Kubernetes Service**
   - AKS manages Kubernetes control plane (free, no control plane costs)
   - Automated upgrades and patches for Kubernetes version
   - Reduces operational burden on team with limited Kubernetes expertise

2. **Native Azure Integration**
   - Seamless integration with Azure SQL Database (private endpoint)
   - Azure AD for authentication and RBAC
   - Managed Identity for accessing Azure services without credentials
   - Azure Monitor for centralized logging and metrics

3. **High Availability and Scalability**
   - Multi-zone deployment provides 99.99% SLA
   - Horizontal Pod Autoscaler (HPA) for dynamic scaling (3-50 replicas)
   - Cluster Autoscaler for automatic node scaling
   - Pod anti-affinity ensures pods distributed across zones

4. **Enterprise-Grade Features**
   - Network policies for micro-segmentation
   - Azure Container Registry (ACR) integration for private images
   - Azure Application Gateway for ingress with WAF capabilities
   - Azure Key Vault integration via CSI driver for secrets

5. **Cost Effectiveness**
   - Pay only for worker nodes (control plane free)
   - Reserved instances available for 30% discount
   - Spot instances option for dev/staging environments

**Comparison Summary**:

| Criteria | AKS | Azure Container Instances (ACI) | Azure App Service (Containers) | Self-Managed Kubernetes on VMs |
|----------|-----|--------------------------------|-------------------------------|-------------------------------|
| **Scalability** | ✅ HPA (3-50 pods) + CA (3-20 nodes) [1] | ⚠️ Manual scaling only (no autoscale) [2] | ✅ Auto-scale (1-30 instances) [3] | ✅ Manual HPA + CA configuration [4] |
| **High Availability** | ✅ 3 availability zones [1] | ❌ Single instance (no multi-zone) [2] | ✅ Multi-instance (zone redundant) [3] | ⚠️ Manual setup required [4] |
| **Azure Integration** | ✅ Managed Identity, ACR, Key Vault, Monitor [1] | ✅ Managed Identity, ACR integration [2] | ✅ Managed Identity, ACR, App Insights [3] | ⚠️ Manual integration (self-managed) |
| **Operational Overhead** | ⚠️ 4-8 hours/week (cluster mgmt) [7] | ✅ <2 hours/week (minimal ops) [7] | ✅ <2 hours/week (PaaS) [7] | ❌ 20+ hours/week (full K8s ops) [7] |
| **Flexibility** | ✅ Full Kubernetes API (1.27+) [1] | ❌ Container-only (no orchestration) | ⚠️ Limited (no custom ingress/CRDs) [3] | ✅ Full control (any K8s version) |
| **Cost** | ⚠️ $2,000/month (9 nodes: 3×D4s_v3 + 6×D8s_v3) [5] | ✅ $500/month (10 vCPU, 20GB RAM continuous) [2] | ⚠️ $1,500/month (Premium P2v3, 3 instances) [3] | ❌ $2,500 infra + $500 ops time [6] |
| **SLA** | ✅ 99.99% (multi-zone, uptime SLA) [8] | ⚠️ 99.9% (472 min/year downtime) [2] | ✅ 99.95% (262 min/year downtime) [3] | ⚠️ Self-managed (depends on setup) |
| **Team Expertise** | ⚠️ 2-4 weeks ramp-up (team survey, n=5) [7] | ✅ 1-3 days (simple container model) [7] | ✅ 3-5 days (familiar PaaS) [7] | ❌ Requires K8s experts (6+ months exp) [7] |

**Decision Factors**:
- **AKS** provides best balance of flexibility, HA, and Azure integration
- **ACI** eliminated due to lack of auto-scaling and multi-zone deployment
- **App Service** eliminated due to PaaS constraints (cannot run Quartz clustering effectively)
- **Self-Managed K8s** eliminated due to high operational overhead and team expertise gap

---

## Consequences

**Positive**:
1. ✅ **High Availability**: Multi-zone deployment provides 99.99% uptime SLA
2. ✅ **Scalability**: Auto-scaling handles variable load (250-500 TPS) efficiently
3. ✅ **Azure Ecosystem**: Seamless integration with Azure SQL, Redis, Confluent Kafka, Key Vault
4. ✅ **Operational Simplicity**: Managed control plane reduces operational burden
5. ✅ **Industry Standard**: Kubernetes skills transferable, avoids vendor lock-in

**Negative**:
1. ❌ **Learning Curve**: Team needs Kubernetes training (estimated 4-week ramp-up)
   - **Mitigation**: Invest in team training, hire Kubernetes-experienced engineer
2. ❌ **Complexity**: Kubernetes introduces complexity (YAML manifests, networking, storage)
   - **Mitigation**: Use Helm charts for standardization, comprehensive documentation
3. ❌ **Cost**: $2,000/month for compute (vs $500/month for ACI)
   - **Mitigation**: Use reserved instances for 30% discount, spot instances for non-prod

**Trade-offs**:
- **What we gained**: High availability, horizontal scalability, flexibility, Azure integration
- **What we sacrificed**: Simplicity (vs ACI/App Service), higher cost, steeper learning curve

---

## Alternatives Considered

**Alternative 1: Azure Container Instances (ACI)**
- **Why Considered**: Simplest container deployment, pay-per-second billing
- **Why Rejected**:
  - No built-in auto-scaling (requires manual intervention)
  - No multi-zone deployment (single point of failure)
  - Limited networking options (no network policies)
  - Not suitable for high-availability production workloads

**Alternative 2: Azure App Service (Containers)**
- **Why Considered**: Managed PaaS, easy deployment, built-in auto-scaling
- **Why Rejected**:
  - PaaS constraints prevent Quartz clustering (shared job store locking issues)
  - Limited control over container orchestration
  - Difficult to implement custom health checks and pod anti-affinity

**Alternative 3: Self-Managed Kubernetes on Azure VMs**
- **Why Considered**: Full control, no AKS limitations
- **Why Rejected**:
  - High operational overhead (control plane management, upgrades, security patches)
  - Team lacks Kubernetes operations expertise
  - Higher total cost of ownership (ops time + infrastructure)

---

## References

### Comparison Data Sources
- [1] AKS documentation - Availability zones and autoscaling: https://learn.microsoft.com/en-us/azure/aks/availability-zones
- [2] Azure Container Instances documentation: https://learn.microsoft.com/en-us/azure/container-instances/
- [3] Azure App Service documentation: https://learn.microsoft.com/en-us/azure/app-service/
- [4] Kubernetes documentation: https://kubernetes.io/docs/
- [5] Azure pricing calculator (AKS compute): https://azure.microsoft.com/en-us/pricing/calculator/ (9 nodes: 3×Standard_D4s_v3 @ $140/mo + 6×Standard_D8s_v3 @ $280/mo = $2,100/month)
- [6] Self-managed K8s cost estimate: Infrastructure ($2,500) + operational time (20 hrs/week × $50/hr avg = $4,000/mo, amortized to $500/mo overhead)
- [7] Team expertise and operational overhead: Internal survey of 5 engineers + operational time tracking
- [8] AKS SLA documentation: https://azure.microsoft.com/en-us/support/legal/sla/kubernetes-service/

### Additional Resources
- AKS Documentation: https://docs.microsoft.com/en-us/azure/aks/
- AKS Best Practices: https://docs.microsoft.com/en-us/azure/aks/best-practices
- AKS Multi-Zone Deployment: https://docs.microsoft.com/en-us/azure/aks/availability-zones
- AKS Pricing: https://azure.microsoft.com/en-us/pricing/details/kubernetes-service/

---

**Last Updated**: 2024-11-20
**Status**: ✅ Accepted