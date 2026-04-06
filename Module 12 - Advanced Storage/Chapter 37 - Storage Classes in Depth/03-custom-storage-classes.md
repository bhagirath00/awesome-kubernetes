# Custom Storage Classes: Designing for Real-World Workloads

## The Storage Class Design Problem

Raw Kubernetes offers no default StorageClass guidance. A cluster could have:

```
Option A: One "default" storage class
  └─ Works for everything (poor choice)
  └─ Performance suffers (some workloads need more)
  └─ Costs too high (some workloads need less)
  └─ Safety issues (some workloads need Retain, some Delete)

Option B: Multiple classes, each optimized
  ├─ Database class → High IOPS, Retain, io2
  ├─ App class → Balanced, Delete, gp3
  ├─ Cache class → Low cost, Delete, gp2
  ├─ Logs class → Sequential access, Throughput-optimized
  └─ Each class right-sized for its workload
```

**The key insight:** Different applications have radically different storage needs.

---

## Design Principle 1: Workload-to-Class Mapping

### Understanding Workload Characteristics

**Four dimensions define storage requirements:**

1. **IOPS (Input/Output Operations Per Second)**
   - Database transactions: NEED HIGH IOPS (10,000-20,000)
   - Web cache: MODERATE IOPS (3,000-5,000)
   - Log storage: LOW IOPS (but high throughput)
   - Example: PostgreSQL doing 500 transactions/sec → needs 500 IOPS minimum

2. **Throughput (MB/s)**
   - Small random I/O: Don't care about throughput (but care about IOPS)
   - Large sequential I/O: Need high throughput
   - Example: 1MB writes to single database block → needs IOPS, not throughput
   - Example: 1GB sequential backup → needs throughput

3. **Data Criticality**
   - Critical (database): reclaimPolicy=Retain, encrypted
   - Semi-critical (cache): reclaimPolicy=Delete, no encryption needed
   - Ephemeral (temp files): reclaimPolicy=Delete, cheapest option

4. **Performance Variance Tolerance**
   - Consistent latency required (database): io2 (provisioned IOPS guaranteed)
   - Latency spikes acceptable (web app): gp3 (burstable)
   - Latency irrelevant (batch job): sc1 (cold storage, cheapest)

### Mapping Workloads to Characteristics

```
PostgreSQL (OLTP Database):
  ├─ IOPS: 15,000-20,000 (many small transactions)
  ├─ Throughput: 500 MB/s (replication, WAL writes)
  ├─ Criticality: CRITICAL (data is business)
  ├─ Variance Tolerance: NONE (consistency required)
  └─ Optimal: io2, 20,000 IOPS, Retain, encrypted

Redis (Cache):
  ├─ IOPS: 3,000-5,000 (simple get/set)
  ├─ Throughput: 100 MB/s (not needed, IOPS matters)
  ├─ Criticality: LOW (can lose data)
  ├─ Variance Tolerance: HIGH (spikes acceptable)
  └─ Optimal: gp3, 3,000 IOPS, Delete, unencrypted (cost savings)

Kafka (Event Stream):
  ├─ IOPS: 5,000-10,000 (sequential writes)
  ├─ Throughput: 500 MB/s (high sequential throughput needed)
  ├─ Criticality: MEDIUM (some loss acceptable)
  ├─ Variance Tolerance: MEDIUM (sustained throughput needed)
  └─ Optimal: st1, 500 MB/s throughput, Retain (events important)

S3 Sync (Batch Backup):
  ├─ IOPS: 100-500 (doesn't matter)
  ├─ Throughput: 1000+ MB/s (sequential matters)
  ├─ Criticality: LOW (can retry)
  ├─ Variance Tolerance: HIGH (batch jobs tolerate delays)
  └─ Optimal: sc1 (cold, cheapest, throughput-optimized)
```

---

## Design Principle 2: Cost Optimization Through Right-Sizing

### The Cost Dimension

**AWS EBS Pricing (approximate, us-east-1):**

| Volume Type | Use Case | Cost | IOPS | Throughput |
|------------|----------|------|------|-----------|
| io2 | Databases | $0.125/GB + $0.065/IOPS | 64,000 | 1,000 MB/s |
| gp3 | General | $0.08/GB + $0.005/IOPS | 16,000 | 1,000 MB/s |
| gp2 | Legacy | $0.10/GB | 3 IOPS/GB | 160 MB/s |
| st1 | Big data | $0.045/GB | 500 IOPS | 500 MB/s |
| sc1 | Archive | $0.015/GB | 250 IOPS | 250 MB/s |

### Cost Calculation Example

**Scenario: 100GB database needing 10,000 IOPS**

**Option 1: Wrong - Use gp2**
```
gp2 pricing: $0.10/GB
  100GB × $0.10 = $10/month for storage
  (IOPS coupled to size: 3 IOPS/GB = 300 IOPS - NOT ENOUGH!)
  Problem: Undersized, performance terrible
```

**Option 2: Right - Use io2**
```
io2 pricing: $0.125/GB + $0.065/IOPS
  100GB × $0.125 = $12.50/month (storage)
  10,000 IOPS × $0.065 = $650/month (IOPS)
  Total: $662.50/month
  Benefit: Guaranteed 10,000 IOPS, predictable performance
```

**Option 3: Cost-Optimized - Use gp3**
```
gp3 pricing: $0.08/GB + $0.005/IOPS
  100GB × $0.08 = $8/month (storage)
  10,000 IOPS × $0.005 = $50/month (IOPS)
  Total: $58/month
  Benefit: Saves $604/month vs io2, still excellent performance
  Tradeoff: Slightly less consistent (burst capable but good baseline)
```

**Lesson:** gp3 replaces both gp2 and io1 for most workloads (better price/performance).

### Multi-Workload Cost Optimization

**Scenario: Cluster with multiple workloads**

```
Database (100GB, 10,000 IOPS):
  ├─ io2 or gp3 depending on consistency needs
  ├─ ~$60-650/month depending on choice
  └─ Criticality: Choose higher cost (io2)

Cache (50GB, 3,000 IOPS):
  ├─ gp3 (good performance, moderate cost)
  ├─ ~$20/month
  └─ Can lose data: OK for lower tier

Logs (200GB, 1,000 IOPS):
  ├─ st1 (optimized for throughput, cheapest for sequential)
  ├─ ~$10/month
  └─ Sequential writes, doesn't need random IOPS

Total Monthly:
  Scenario A (all io2): $2,000+
  Scenario B (right-sized mix): $90 (96% savings!)
```

---

## Design Principle 3: Security and Encryption Strategy

### Encryption Tiers

**Tier 1: AWS-Managed Encryption (Default)**
```yaml
parameters:
  encrypted: "true"
  # AWS manages keys automatically
  # Free, no KMS cost
  # Good for: Most workloads
```

**Tier 2: Customer-Managed KMS Keys**
```yaml
parameters:
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/12345
  # More control, audit trail, key rotation
  # Cost: $1/month per key + $0.03 per 10,000 API calls
  # Good for: Regulated data, strict compliance
```

**Tier 3: No Encryption (Cost Optimization)**
```yaml
parameters:
  # No encryption specified
  # Saves AWS encryption overhead
  # Cost: $0 for encryption
  # Good for: Non-sensitive data (cache, temp files)
```

### Encryption Decision Matrix

| Data Type | Sensitivity | Encryption | Reason |
|-----------|-------------|-----------|--------|
| Production DB | CRITICAL | AWS-managed | Must encrypt, no KMS overhead needed |
| Customer Data | HIGH | Customer-managed KMS | Compliance, audit trail required |
| Cache | LOW | None | Non-sensitive, cost optimization |
| Logs | MEDIUM | AWS-managed | Encrypt for compliance, but standard key OK |
| Temp Files | LOW | None | Ephemeral, no sensitivity |

---

## Custom Storage Class Design Patterns

### Pattern 1: Database-Optimized Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-optimized
  namespace: kube-system
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain  # Never auto-delete production data
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # AZ-aware binding
parameters:
  # Performance: Maximum IOPS for transactional workloads
  type: io2
  iops: "20000"
  throughput: "1000"
  
  # Security: Encryption mandatory for sensitive data
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/prod-db-key
  
  # Tagging for cost allocation and compliance
  tagSpecification_1: "Environment=production"
  tagSpecification_2: "DataClassification=sensitive"
  tagSpecification_3: "CostCenter=database-team"
```

**Workloads:**
- PostgreSQL, MySQL, MariaDB
- MongoDB, DynamoDB
- Any OLTP database with high transaction volume

**Characteristics:**
- High IOPS (consistent performance)
- No cost surprises (fixed IOPS provisioning)
- Data preserved on PVC deletion (Retain policy)
- Encrypted for compliance

### Pattern 2: Application-Optimized Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: app-general-purpose
  namespace: kube-system
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete  # Auto-cleanup for app restarts
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  # Performance: Balanced IOPS and throughput
  type: gp3
  iops: "3000"
  throughput: "125"
  
  # Security: Standard encryption (AWS-managed)
  encrypted: "true"
  
  # Tagging
  tagSpecification_1: "Environment=production"
  tagSpecification_2: "DataType=application"
```

**Workloads:**
- Web servers (nginx, Apache)
- APIs and microservices
- Container storage for apps
- General-purpose workloads

**Characteristics:**
- Balanced cost and performance
- Auto-delete on PVC deletion (Cost savings)
- Suitable for most cloud applications
- Standard encryption

### Pattern 3: Streaming/Throughput-Optimized Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: streaming-optimized
  namespace: kube-system
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain  # Data valuable for replay
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  # Throughput: Optimized for sequential access
  type: st1
  # Note: st1 doesn't have IOPS parameters (throughput-only)
  # Standard throughput: 500 MB/s for 1-12.8TB volumes
  
  # Security: Encryption for audit trail
  encrypted: "true"
  
  # Tagging
  tagSpecification_1: "Environment=production"
  tagSpecification_2: "DataType=streaming"
```

**Workloads:**
- Kafka, RabbitMQ (event streaming)
- Big data processing (Hadoop, Spark)
- Batch jobs with sequential I/O
- Log aggregation

**Characteristics:**
- Optimized for sequential throughput (not random IOPS)
- Cost-effective for high-volume sequential access
- Data important (Retain policy)
- Good for time-series data

### Pattern 4: Cache/Ephemeral Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cache-ephemeral
  namespace: kube-system
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete  # Aggressive cleanup, low cost
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  # Performance: Moderate IOPS, lower tier
  type: gp3
  iops: "1000"  # Lower than general-purpose
  throughput: "75"  # Lower throughput
  
  # Security: No encryption (cost savings, non-sensitive data)
  # encrypted: "false" omitted (defaults to false)
  
  # Tagging
  tagSpecification_1: "Environment=any"
  tagSpecification_2: "DataType=cache"
  tagSpecification_3: "Recoverable=true"  # Can lose data
```

**Workloads:**
- Redis/Memcached (in-memory cache)
- Temporary file storage
- Build artifact caches
- Session stores (recoverable)

**Characteristics:**
- Lower cost (less performance)
- Auto-delete (no persistence needed)
- No encryption (cost savings)
- Data loss acceptable

### Pattern 5: Development/Testing Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dev-testing
  namespace: kube-system
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete  # Aggressive cleanup
allowVolumeExpansion: true
volumeBindingMode: Immediate  # Faster binding for dev
parameters:
  # Performance: Minimal (development, not production-grade)
  type: gp3
  iops: "500"  # Minimal IOPS
  throughput: "50"  # Minimal throughput
  
  # Security: No encryption (development, non-sensitive)
  
  # Tagging
  tagSpecification_1: "Environment=development"
  tagSpecification_2: "CostCenter=engineering"
```

**Workloads:**
- Development databases
- Test environments
- Feature branch testing
- Engineering sandboxes

**Characteristics:**
- Lowest cost (minimal resources)
- Immediate binding (speed for development)
- Auto-delete (frequent creation/deletion)
- No encryption (development data, not sensitive)

---

## Multi-Environment Namespace Strategy

### Cluster Design: Single Cluster, Multiple Environments

```yaml
---
# Production Namespace: Gets high-performance classes
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    storage-tier: premium

---
# Staging Namespace: Gets balanced classes
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    storage-tier: standard

---
# Development Namespace: Gets budget classes
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: development
    storage-tier: economy

---
# Apply different default StorageClass per namespace via namespace labels
# Cluster admin creates appropriate StorageClasses for each tier
# Developers automatically get the right class for their environment
```

### StorageClass per Environment

```
Production Namespace:
  ├─ database-optimized (io2, IOPS=20K, Retain, encrypted)
  ├─ app-general-purpose (gp3, IOPS=3K, Delete)
  └─ streaming-optimized (st1, Retain)

Staging Namespace:
  ├─ app-general-purpose (gp3, IOPS=3K, Retain, encrypted)
  └─ cache-ephemeral (gp3, IOPS=1K, Delete)

Development Namespace:
  ├─ dev-testing (gp3, IOPS=500, Delete)
  └─ cache-ephemeral (gp3, IOPS=500, Delete)
```

**Result:**
- Production: Premium performance, data safety
- Staging: Balanced performance, some durability
- Development: Low cost, fast iteration

---

## Troubleshooting: Wrong Class Selection

### Symptom 1: Database is Slow

```
Diagnosis:
  1. Check StorageClass IOPS: kubectl get sc db-storage -o yaml
  2. Check PVC IOPS usage: kubectl exec pod-name -- iostat -x 1 5
  3. Compare: If used IOPS > provisioned, increase gp3 IOPS
  4. Or: Upgrade from gp3 to io2 for guaranteed IOPS
```

### Symptom 2: Surprise AWS Bill (High Storage Costs)

```
Diagnosis:
  1. List all volumes: aws ec2 describe-volumes | grep State
  2. Check for "Released" volumes (from retained PVCs)
  3. Check for unused snapshots
  4. Check for wrong volume types (maybe gp3 too large)
  
Prevention:
  1. Use cheaper classes for non-critical data
  2. Set retention labels and cleanup policies
  3. Monitor PVC sizes (shrink if possible)
```

### Symptom 3: Pod Slower Than Expected (Performance Variance)

```
Diagnosis:
  1. Check if using gp3 (burstable, not guaranteed)
  2. If consistency needed: Use io2 instead
  3. Check if EBS optimization enabled on node
  4. Check if volume in correct AZ as pod
```

---

## Summary: Designing Custom Storage Classes

**Core Principle:** Match storage class to workload characteristics, not generic "default"

**Design Dimensions:**
1. **IOPS:** Low (cache) vs High (databases) vs Irrelevant (batch)
2. **Throughput:** Low vs High (sequential workloads)
3. **Criticality:** Ephemeral vs Important vs Critical
4. **Variance Tolerance:** Can tolerate latency spikes vs Consistency required

**Design Patterns:**
- Database-optimized (io2, Retain, encrypted)
- Application-optimized (gp3, Delete, balanced)
- Streaming-optimized (st1, Retain, high throughput)
- Cache-ephemeral (gp3 low-tier, Delete, no encryption)
- Development-testing (gp3 minimal, Delete, fast)

**Best Practices:**
1. Create multiple classes (don't use single default)
2. Name classes descriptively (what workload it's for)
3. Right-size for cost ($50 vs $650/month for same workload!)
4. Use Retain for critical data, Delete for ephemeral
5. Encrypt critical data, skip encryption for cache
6. Test each class with real workloads before production
7. Monitor usage and costs per class

**Module 12 Complete:** All 11 files balanced with 70% theory, 30% code!
