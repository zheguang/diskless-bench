# Benchmark Plan for Kafka Diskless (KIP-1150) - Inkless Implementation

## Table of Contents

1. [Overview](#overview)
   - [Benchmark Objectives](#benchmark-objectives)
   - [Specific Research Questions](#specific-research-questions)
2. [Baseline Scenario](#baseline-scenario)
   - [Key Findings from Aiven's Benchmark](#key-findings-from-aivens-benchmark)
   - [Specific Configuration Details from Blog Post](#specific-configuration-details-from-blog-post)
3. [Storage Backend Configuration Variations](#storage-backend-configuration-variations)
   - [Configuration A: AWS S3 Standard](#configuration-a-aws-s3-standard-baseline---aivens-configuration)
   - [Configuration B: AWS S3 Express One Zone](#configuration-b-aws-s3-express-one-zone-low-latency-exploration)
   - [Configuration C: NetApp FSxN S3](#configuration-c-netapp-fsxn-s3-ultra-low-latency-exploration)
     - [Configuration C1: FSxN - SSD Only](#configuration-c1-fsxn---ssd-only-no-tiering)
     - [Configuration C2: FSxN - Auto Tiering](#configuration-c2-fsxn---auto-tiering-cost-optimized)
4. [Storage-Specific Metrics](#storage-specific-metrics)
   - [S3 Standard Metrics](#s3-standard-metrics)
   - [S3 Express One Zone Metrics](#s3-express-one-zone-metrics)
   - [FSxN S3 Metrics](#fsxn-s3-metrics)
   - [Common Metrics Across All Backends](#common-metrics-across-all-backends)
5. [Test Scenarios](#test-scenarios)
   - [Scenario 1: Baseline Producer/Consumer Throughput](#scenario-1-baseline-producerconsumer-throughput)
   - [Scenario 2: Consumer Lag Catch-Up](#scenario-2-consumer-lag-catch-up)
   - [Scenario 3: Burst Workload](#scenario-3-burst-workload)
   - [Scenario 4: High Partition Count](#scenario-4-high-partition-count)
   - [Scenario 5: Long-Running Stability](#scenario-5-long-running-stability)
6. [Expected Outcomes](#expected-outcomes)
   - [Performance vs Cost Spectrum](#performance-vs-cost-spectrum)
   - [Detailed Cost Calculations](#detailed-cost-calculations)
   - [Cost Summary Table](#cost-summary-table)
   - [Research Outcomes](#research-outcomes)
   - [Recommendations Framework](#recommendations-framework)
7. [Next Steps](#next-steps)
8. [Future Work](#future-work)
9. [References](#references)

---

## Overview

This benchmark plan outlines the approach to evaluate the performance of Kafka Diskless topics using Aiven's Inkless implementation. The benchmark will compare performance between **AWS S3 Standard**, **AWS S3 Express One Zone**, and **NetApp FSxN S3** object storage backends.

**Note**: This benchmark is for **object storage** (S3-compatible), not traditional block/file storage. Kafka Diskless/Inkless stores log segments directly in S3-compatible object storage.

### Benchmark Objectives

The primary goal is to **explore the cost/performance trade-off spectrum** for Kafka Diskless (Inkless) across different object storage backends:

1. **Establish Baseline**: Replicate Aiven's results with S3 Standard (>94% savings, ~650ms P50 latency)
2. **Evaluate S3 Express One Zone**: Measure if 10x lower S3 latency (<10ms vs 100ms) translates to meaningful end-to-end latency improvements and justify 5x higher storage costs
3. **Evaluate FSxN S3**: Measure if sub-millisecond cached access and dual-layer caching improve Diskless performance enough to justify provisioned capacity costs
4. **Understand Trade-offs**: Quantify the cost/latency/complexity trade-offs for different workload patterns

#### Specific Research Questions:

**Performance:**
- How much does S3 Express reduce end-to-end latency? (Target: <300ms P50 vs 650ms?)
- How much does FSxN caching help consumer catch-up scenarios?
- Which storage backend handles burst workloads best?
- What's the impact of high partition counts on each backend?

**Cost:**
- At what throughput/scale does S3 Express become cost-competitive?
- When does FSxN's fixed capacity cost become worthwhile vs S3 pay-per-use?
- What's the true TCO including compute, storage, networking, and operational overhead?

**Operational:**
- Which backend provides most predictable performance (fewer spikes)?
- How does each handle PostgreSQL coordinator latency requirements (<100ms CommitFile)?
- Which backend simplifies operations and monitoring?

**Use Case Fit:**
- When should users choose S3 Standard? (Long-term storage, cost-sensitive)
- When should users choose S3 Express? (Low-latency streaming, high request rates)
- When should users choose FSxN S3? (Ultra-low latency, predictable performance)

## Baseline Scenario

The baseline scenario is based on the benchmark documented in [Aiven's blog post](https://aiven.io/blog/benchmarking-diskless-inkless-topics-part-1), which achieved **>94% cost savings** ($288k vs $3.32M/year) compared to traditional Kafka.

### **Key Findings from Aiven's Benchmark:**
- **Latency trade-off**: ~1 second extra latency for 90% cost reduction
- **End-to-end latency**: P50 ~650ms, P99 ~1.5s (spikes to 8s)
- **Producer latency**: P50 ~250ms, P99 ~500ms (spikes to 4s)
- **CPU utilization**: Only ~30% on 6 brokers
- **Memory usage**: Diskless uses **on-heap cache** (75% of RAM = 48 GiB) instead of Linux page cache
- **S3 performance**: P50 ~100ms, P99 ~200ms for PUT/GET operations
- **Cross-AZ cost**: Only $7,803/year for Kafka metadata (vs $3.3M saved on replication)
- **PostgreSQL coordinator**: Lightweight metadata operations (~13 MiB/s total traffic)

### Specific Configuration Details from Blog Post:

**Topic Configuration:**
- Topic name: `inkless-benchmark-topic`
- Partitions: 576 partitions
- Replication factor: 3 (across AZs)
- Retention: Default (using object storage)
- Message size: Variable, uncompressed for benchmarking
- Compression: None (for accurate throughput measurement)

**Producer Configuration:**
- Number of producers: 3 (one per AZ)
- Throughput target: 1 GiB/s total (333 MiB/s per producer)
- Acks: all
- Linger.ms: 100
- Batch.size: 1048576 (1 MiB)
- Max.request.size: 4194304 (4 MiB)
- Compression.type: none

**Consumer Configuration:**
- Number of consumers: 6 (two per AZ)
- Consumer groups: 3 groups
- Throughput target: 3 GiB/s total (500 MiB/s per consumer)
- Fetch.max.bytes: 67108864 (64 MiB)
- Fetch.max.wait.ms: 500
- Fetch.min.bytes: 4194304 (4 MiB)

**Cluster Setup:**
- Brokers: 6 brokers (m8g.4xlarge instances)
- Kafka version: 4.0 with Inkless implementation (Aiven's fork)
- JVM heap: **75% of instance memory = 48 GiB** (critical for Diskless on-heap cache)
  - **Note**: Traditional Kafka uses 4-8 GB heap, but Diskless needs much more for caching
  - Remaining 16 GiB for OS and page cache for internal/classic topics
- OS: Amazon Linux
- Instance type: m8g.4xlarge (16 vCPUs, 64 GiB memory)

**Storage Configuration (AWS S3 Standard Baseline):**
- Object storage: **AWS S3 Standard** (for Diskless topics)
  - API: S3 API (PutObject, GetObject, ListObjects)
  - Capacity: Unlimited (elastic)
  - Data stored: 3600 GiB (1 hour of data at 1 GiB/s ingress)
  - Region: us-east-1 (or target region)
  - Storage class: S3 Standard
  - Expected latency: 100-200ms per request
  - Expected throughput: Limited by request rate (5,500 GET/sec per prefix)
- Metadata store: PostgreSQL (i3.2xlarge, dual-AZ, 1.9TiB NVMe)
- Network: Cross-AZ traffic for metadata operations
- Workload: 1 GiB/s in, 3 GiB/s out (fan-out pattern)

**Key Benchmark Parameters:**
- Test duration: 1 hour
- Producer clients: 144
- Consumer clients: 144
- Partitions per producer: ~4
- Total throughput: 1 GiB/s ingress, 3 GiB/s egress

## Storage Backend Configuration Variations

For comparing different object storage backends, we will test the following configurations. **Important**: All configurations use S3-compatible object storage APIs, not NFS/file protocols.

### Configuration A: AWS S3 Standard (Baseline - Aiven's Configuration)
- **Storage Backend**: AWS S3 Standard
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge instances)
- **S3 Configuration**:
  - Storage class: S3 Standard
  - Region: us-east-1 (or target region)
  - Prefix strategy: Partition-based prefixes (for parallel request scaling)
  - **Actual measured latency from Aiven**: P50 ~100ms, P99 ~200ms per object operation
  - **Actual measured throughput**: ~50 PUT/sec per broker, ~100 GET/sec per broker
  - Expected throughput limits: 5,500 GET/sec per prefix, 3,500 PUT/sec per prefix
  - Request cost: $0.40 per million GET, $5 per million PUT
  - Storage cost: $23/TiB/month
  - **Actual S3 storage used**: 3,600 GiB (1 hour of data at 1 GiB/s)
- **Kafka Configuration**:
  - Matches exact baseline configuration from blog post
  - No compression (as per baseline)
  - Same producer/consumer settings
  - Segment size: Default (1 GB)
  - **File size uploaded to S3**: 4 MiB (found to be optimal for S3 PUT latency)
  - **Buffer timeout**: 250ms default (files upload faster if 4 MiB limit hit first)
- **PostgreSQL Coordinator** (critical component):
  - Instance: i3.2xlarge, dual-AZ, 1.9 TiB NVMe
  - Metadata writes: ~1 MiB/s
  - Metadata reads: ~1.5 MiB/s
  - WAL replication: ~10 MiB/s (cross-AZ)
  - **CommitFile latency**: Must be <100ms for predictable performance
- **Expected**: Baseline performance with ~1s latency trade-off for 94% cost savings
- **Rationale**:
  - This is the **proven configuration** from Aiven with real measurements
  - Standard S3 is the most common and cost-effective option
  - 100-200ms S3 latency translates to 650ms P50 end-to-end latency
  - **CPU headroom**: Only 30% utilization suggests 2-3x throughput possible
  - **Cost savings**: Eliminates $3M+ in cross-AZ replication and disk costs

### Configuration B: AWS S3 Express One Zone (Low Latency Exploration)
- **Storage Backend**: AWS S3 Express One Zone
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge instances) co-located in same AZ
- **S3 Configuration**:
  - Storage class: S3 Express One Zone
  - Availability Zone: us-east-1a (same as brokers for lowest latency)
  - Directory bucket: Use S3 directory bucket type
  - Target latency: <10ms per object operation (vs ~100ms for S3 Standard)
  - Expected throughput: Up to 2,000,000 req/sec per directory bucket (vs ~300 req/sec in baseline)
  - Request cost: $0.03 per million GET (80% cheaper), $1.13 per million PUT
  - Storage cost: $110/TiB/month (vs $23/TiB for S3 Standard)
- **Research Questions**:
  - Does 10x lower S3 latency reduce end-to-end latency proportionally?
  - Target: Can we achieve <300ms P50 end-to-end (vs 650ms baseline)?
  - How much does consumer catch-up improve with faster random reads?
  - What's the break-even point where request cost savings offset storage cost increase?
- **Expected Cost Trade-off**:
  - Storage: 5x more expensive ($110 vs $23/TiB)
  - Requests: 80% cheaper for GETs (Diskless is read-heavy with 2:1 GET:PUT ratio)
  - For 3,600 GiB + 1M PUT + 2M GET per hour:
    - S3 Standard: ~$83/month storage + ~$5 PUT + ~$0.8 GET = **$89/month**
    - S3 Express: ~$396/month storage + ~$1.13 PUT + ~$0.06 GET = **$397/month**
  - **~4.5x more expensive per month**, but may enable workloads that need <500ms latency
- **Kafka Configuration**:
  - Same as baseline
  - Co-location: All brokers in same AZ as S3 Express bucket
  - **Trade-off**: Single AZ reduces availability (99.95% vs 99.99%)
- **Expected Outcome**: 
  - Significantly lower latency but higher cost
  - Best for latency-sensitive streaming applications
  - May justify cost if end-to-end latency drops to <300ms P50

### Configuration C: NetApp FSxN S3 (Ultra-Low Latency Exploration)

FSxN ONTAP provides automatic tiering between SSD and capacity pool tiers, allowing for different cost/performance trade-offs. We will test three FSxN configurations:

---

#### Configuration C1: FSxN - SSD Only (No Tiering)

**Purpose:** Baseline FSxN performance with all data on high-performance SSD.

- **Storage Backend**: NetApp FSx for ONTAP S3 API
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge instances)
- **FSxN Configuration**:
  - Storage capacity: 3,700 GiB SSD (3,600 GiB Kafka + 100 GiB PostgreSQL)
  - Throughput capacity: 2,048 MBps (2 GBps)
  - SSD IOPS: 11,059 IOPS (3,072 IOPS/TiB × 3.6 TiB)
  - Disk throughput: 2,765 MBps baseline (768 MBps/TiB × 3.6 TiB)
  - Protocol: **S3 API** (not NFS - using ONTAP's S3 interface)
  - Deployment type: Multi-AZ (for high availability)
  - **Tiering policy**: None (all data on SSD)
  - **Storage efficiency**: Disabled (baseline measurement)
  - Target latency: <1ms (cached), 1-5ms (uncached SSD)
  - Cache: Dual-layer (NVMe + in-memory)
- **Costs**:
  - SSD storage: $530/month (3,700 GiB × $0.14/GiB)
  - Throughput capacity: $3,521/month (2,048 MBps × $1.719/MBps)
  - PostgreSQL coordinator: $575/month (m5.2xlarge on FSxN NFS)
  - **Total: $4,626/month**
- **Research Questions**:
  - Does FSxN's dual-layer cache significantly improve hot segment access?
  - Can <1ms cached reads reduce end-to-end latency below S3 Express?
  - Is provisioned capacity cost justified by performance predictability?
- **Expected Outcome**: 
  - Best latency: <100ms P50 end-to-end (target)
  - Highest cost: $4,626/month
  - Predictable performance with no tiering delays

---

#### Configuration C2: FSxN - Auto Tiering (Cost Optimized)

**Purpose:** Reduce FSxN costs by automatically tiering cold data to capacity pool.

- **Storage Backend**: NetApp FSx for ONTAP S3 API
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge instances)
- **FSxN Configuration**:
  - **Tiering policy**: Auto (aggressive)
  - **Cooling period**: 2 days (minimum, for faster tiering)
  - **Storage efficiency**: Disabled (measure tiering impact separately)
  - Expected data distribution:
    - Hot data (20%): 740 GiB on SSD
    - Cold data (80%): 2,960 GiB on capacity pool
  - Throughput capacity: 2,048 MBps (2 GBps)
  - Protocol: S3 API
  - Deployment type: Multi-AZ
  - Target latency: <1ms (hot/cached), 10-20ms (cold/capacity pool)
- **Costs**:
  - SSD storage (20%): $104/month (740 GiB × $0.14/GiB)
  - Capacity pool (80%): $48/month (2,960 GiB × $0.0163/GiB)
  - Throughput capacity: $3,521/month
  - PostgreSQL coordinator: $575/month
  - **Total: $4,248/month** (8% savings vs C1)
- **Research Questions**:
  - How much does automatic tiering reduce costs for cold data?
  - Does 10-20ms capacity pool latency impact lagging consumer performance?
  - What percentage of data actually tiers in real Kafka workloads?
  - Does caching compensate for capacity pool latency?
- **Expected Outcome**:
  - Hybrid latency: <100ms P50 for hot consumers, <300ms P50 for lagging consumers
  - 8% cost reduction vs SSD-only
  - Transparent to Inkless (no configuration changes)

---

---

### FSxN Configuration Comparison

| **Configuration** | **Tiering** | **Storage Cost** | **Total Cost** | **Savings vs C1** | **Target P50** |
|-------------------|-------------|------------------|----------------|-------------------|----------------|
| **C1: SSD Only** | None | $530 | $4,626 | 0% (baseline) | <100ms |
| **C2: Auto Tiering** | Auto (2d) | $152 | $4,248 | 8% | <100ms (hot), <300ms (cold) |

**Key Insights:**
- FSxN tiering is **transparent to Inkless** (no code changes, same S3 API)
- Auto tiering can reduce storage costs by 71% ($530 → $152)
- Throughput cost ($3,521) remains fixed regardless of tiering
- PostgreSQL coordinator benefits from FSxN infrastructure ($575 vs $1,616)
- Following Aiven baseline: compression/deduplication excluded from benchmark

### Common FSxN Settings (All Configurations)

- **Kafka Configuration**:
  - Same as baseline (Configuration A)
  - S3 endpoint: FSxN S3 endpoint URL
  - S3 credentials: FSxN S3 user credentials
  - **No Inkless changes needed** (tiering transparent at storage layer)
- **PostgreSQL Coordinator**:
  - 2× m5.2xlarge instances
  - PostgreSQL data on FSxN via NFS (not S3)
  - Benefits from FSxN Multi-AZ replication
  - No WAL cross-AZ charges ($0 vs $500)
- **FSxN Volume Settings**:
  - SSD utilization target: <80% (for optimal tiering)
  - Capacity pool: Auto-scaling (no provisioning needed)
  - Caching: Random reads promoted to SSD when SSD <90% utilization

### FSxN vs Other Options

| **Configuration** | **Monthly Cost** | **Target P50** | **Best For** |
|-------------------|-----------------|----------------|--------------|
| **A: S3 Standard** | $1,598 | ~650ms | Cost-sensitive, proven at scale |
| **B: S3 Express** | $2,100 | <300ms | Latency-sensitive streaming |
| **C1: FSxN SSD** | $4,626 | <100ms | Ultra-low latency baseline |
| **C2: FSxN Tiering** | $4,248 | <100ms / <300ms | Cost-optimized FSxN |

**FSxN Trade-offs:**
- ✅ Best latency for hot data (<1ms cached)
- ✅ Transparent automatic tiering
- ✅ No Inkless code changes required
- ✅ Multi-AZ with high availability
- ⚠️ Still 2-3x more expensive than S3 options
- ⚠️ Fixed throughput cost ($3,521/month)
- ⚠️ Requires capacity planning (SSD sizing)

**Note:** Following Aiven's baseline approach, compression/deduplication features are excluded from this benchmark to isolate storage backend performance characteristics.

## Storage-Specific Metrics

In addition to standard Kafka metrics, we will track storage-backend-specific performance indicators:

### S3 Standard Metrics:
- **Request Rate**: GET/PUT requests per second
- **Request Latency**: P50, P95, P99 latency per operation type
- **Throttling**: 503 SlowDown errors (indicates scaling)
- **Prefix Distribution**: Request distribution across prefixes
- **Cost Metrics**: Storage cost + request cost + data transfer cost

### S3 Express One Zone Metrics:
- **Request Rate**: Requests per second (should reach 100k+)
- **Request Latency**: P50, P95, P99 (should be <10ms)
- **Co-location Benefit**: Latency comparison same-AZ vs cross-AZ
- **Cost Metrics**: Higher storage cost vs lower request cost trade-off

### FSxN S3 Metrics:
- **Cache Hit Ratio**: % of requests served from cache (NVMe/memory)
- **Throughput Utilization**: % of provisioned 2 GBps capacity used
- **IOPS Utilization**: % of provisioned IOPS used
- **Latency by Tier**: Cached (<1ms) vs SSD (1-5ms) vs capacity pool (10-20ms)
- **Tiering Efficiency**: % of data automatically tiered to capacity pool
- **Cost Metrics**: Fixed capacity cost vs S3 pay-per-use cost

### Common Metrics Across All Backends:
- **Producer Throughput**: MB/s per partition
- **Consumer Throughput**: MB/s per consumer
- **Consumer Lag**: Time to catch up 1 hour lag
- **End-to-End Latency**: Producer to consumer latency
- **Broker CPU/Memory**: Resource utilization
- **Network Throughput**: Bytes in/out per broker

## Test Scenarios

### Scenario 1: Baseline Producer/Consumer Throughput
- **Storage**: Test each configuration (A, B, C1, C2)
- **Workload**: 1 GiB/s producer, 3 GiB/s consumer (3x fan-out)
- **Duration**: 1 hour
- **Metrics**: Throughput, latency (P50/P95/P99), CPU, memory, network

### Scenario 2: Consumer Lag Catch-Up
- **Storage**: Test each configuration
- **Setup**: Create 1 hour of lag (360 GB of backlog)
- **Workload**: Consumers catching up from 1 hour behind
- **Metrics**: Time to catch up, random read IOPS, latency, cost

### Scenario 3: Burst Workload
- **Storage**: Test each configuration
- **Workload**: Alternating between 0.5 GiB/s and 2 GiB/s every 10 minutes
- **Duration**: 1 hour
- **Metrics**: How each backend handles bursty traffic, throttling, latency spikes

### Scenario 4: High Partition Count
- **Storage**: Test each configuration
- **Workload**: 1152 partitions (2x baseline) with same throughput
- **Metrics**: Impact of small object sizes on each backend

### Scenario 5: Long-Running Stability
- **Storage**: Best performing configurations only
- **Workload**: Steady 1 GiB/s in, 3 GiB/s out
- **Duration**: 24 hours
- **Metrics**: Stability, consistency, cost over time

## Expected Outcomes

### Performance vs Cost Spectrum

Based on the comparison of storage backends, we expect to map out a clear cost/performance trade-off spectrum:

| **Storage** | **User Data Cost** | **Metadata (PostgreSQL) Cost** | **Total Monthly Cost** | **Target P50** | **Target P99** | **Cost vs Classic** | **Best For** |
|-------------|-------------------|--------------------------------|------------------------|----------------|----------------|---------------------|--------------|
| **Classic Kafka** | **$275,904** | **$0** | **$275,904** | ~50ms | ~100ms | 100% | Ultra-low latency, traditional |
| **Diskless + S3 Standard** | **$89** | **$1,616** | **$1,705** | ~650ms | ~1.5s | **0.62% (99.38% savings)** | Cost-sensitive, proven at scale |
| **Diskless + S3 Express** | **$590** | **$1,616** | **$2,206** | <300ms? | <800ms? | **0.80% (99.20% savings)** | Latency-sensitive streaming |
| **Diskless + FSxN (SSD Only)** | **$4,051** | **$575** | **$4,626** | <100ms? | <500ms? | **1.68% (98.32% savings)** | Ultra-low latency baseline |
| **Diskless + FSxN (Auto Tier)** | **$3,673** | **$575** | **$4,248** | <100ms (hot)<br><300ms (cold) | <500ms (hot)<br><1s (cold) | **1.54% (98.46% savings)** | Cost-optimized FSxN |

**Cost Breakdown:**

- **Classic Kafka** ($275,904):
  - User data (disk storage): $18,548
  - Cross-AZ replication: $257,356
  - PostgreSQL coordinator: $0 (not needed)

- **Diskless + S3 Standard** ($1,705 = $89 + $1,616):
  - User data (S3 storage + requests): $89
  - Metadata (PostgreSQL coordinator): $1,616 (dual-AZ i3.2xlarge + cross-AZ traffic)

- **Diskless + S3 Express** ($2,206 = $590 + $1,616):
  - User data (S3 Express storage + requests): $590
  - Metadata (PostgreSQL coordinator): $1,616 (dual-AZ i3.2xlarge + cross-AZ traffic)

- **Diskless + FSxN S3 (SSD Only)** ($4,626 = $4,051 + $575):
  - User data (FSxN storage + throughput): $4,051
    - SSD storage: 3,700 GiB → $530 ($516 Kafka + $14 PostgreSQL)
    - Throughput capacity: 2 GBps → $3,521
  - Metadata (PostgreSQL coordinator compute): **$575** (2× m5.2xlarge)
    - **62% cheaper** than standard coordinator by leveraging FSxN infrastructure
    - No WAL cross-AZ charges (FSxN handles replication internally)
    - Storage already counted in user data cost above

- **Diskless + FSxN S3 (Auto Tiering)** ($4,248 = $3,673 + $575):
  - User data (FSxN storage + throughput): $3,673
    - SSD storage (20% hot): 740 GiB → $104
    - Capacity pool (80% cold): 2,960 GiB → $48
    - Throughput capacity: 2 GBps → $3,521
  - Metadata (PostgreSQL coordinator): $575
  - **8% savings** vs SSD-only FSxN

**Key Insights:**
- **Diskless trade-off**: Ultra-cheap user data storage ($89-$4,051) but requires expensive metadata layer ($575-$1,616)
- **Classic Kafka**: All-in-one cost ($275k) includes both data and metadata (Zookeeper/KRaft)
- **FSxN tiering is transparent**: No Inkless code changes needed, automatic at storage layer
- **FSxN storage cost reduction**: Up to 71% ($530 → $152) with auto tiering
- **Throughput cost dominates FSxN**: $3,521/month fixed regardless of optimization
- **Following Aiven baseline**: Compression/deduplication excluded to isolate storage performance
- FSxN option can host PostgreSQL coordinator on same FSxN infrastructure, reducing coordinator costs
- PostgreSQL on FSxN benefits from sub-millisecond latency and eliminates WAL cross-AZ charges
- Even with coordinator costs, **S3 Standard saves $274k/month** vs Classic Kafka
- All costs exclude broker compute, which is similar across options (though Classic may need more brokers)

**FSxN Configuration Comparison:**

| **FSxN Variant** | **Tiering** | **Storage Cost** | **Total Cost** | **Savings vs SSD-Only** | **Complexity** |
|------------------|-------------|------------------|----------------|------------------------|----------------|
| **SSD Only (C1)** | None | $530 | $4,626 | 0% (baseline) | Low |
| **Auto Tiering (C2)** | Yes (2d) | $152 | $4,248 | 8% | Medium |

**Note:** Compression/deduplication excluded per Aiven baseline methodology.

### Detailed Cost Calculations

#### Assumptions (Based on Aiven Benchmark):
- **Data retention**: 1 hour of data at 1 GiB/s ingress = 3,600 GiB (3.6 TiB)
- **Workload**: 1 GiB/s producer ingress, 3 GiB/s consumer egress (3x fan-out)
- **Duration**: 730 hours/month (30.4 days)
- **Partitions**: 576 partitions across 6 brokers
- **Replication factor**: 3 (for Classic Kafka)
- **Availability Zones**: 3 AZs (us-east-1)
- **S3 requests** (from Aiven measurements):
  - PUT: ~50 req/sec per broker × 6 brokers = 300 req/sec = 1,080,000 req/hr
  - GET: ~100 req/sec per broker × 6 brokers = 600 req/sec = 2,160,000 req/hr
- **PostgreSQL Coordinator** (required for all Diskless configurations):
  - Instance type: i3.2xlarge (8 vCPUs, 61 GiB RAM, 1.9 TiB NVMe SSD)
  - Deployment: Dual-AZ for high availability (2 instances)
  - Metadata writes: ~1 MiB/s, Metadata reads: ~1.5 MiB/s
  - WAL replication: ~10 MiB/s (cross-AZ)
- **Region**: us-east-1 (AWS pricing as of 2026)

---

#### Cost Calculation 1: Classic Kafka (3-AZ, RF=3)

**Cross-AZ Replication Cost:**
```
Producer ingress: 1 GiB/s
Replication factor: 3 (data replicated to 2 other AZs)
Cross-AZ egress: 1 GiB/s × 2 AZs = 2 GiB/s

Consumer egress: 3 GiB/s (3x fan-out)
Consumers read from 2 other AZs (⅔ of traffic): 3 GiB/s × ⅔ = 2 GiB/s

Total cross-AZ egress: 2 GiB/s (replication) + 2 GiB/s (consumers) = 4 GiB/s
Monthly data transfer: 4 GiB/s × 3,600 sec/hr × 730 hr/month = 10,512,000 GiB
AWS cross-AZ pricing: $0.02/GiB

Cross-AZ cost = 10,512,000 GiB × $0.02 = $210,240/month
```
**Note**: Aiven blog cites $257,356/month. The difference may include:
- Additional metadata replication overhead
- Leader-to-follower fetches
- Reassignment and rebalancing traffic
- Using their actual measured value: **$257,356/month**

**Disk Storage Cost:**
```
Data per broker (with RF=3, distributed): 3,600 GiB × 3 / 9 brokers = 1,200 GiB per broker
Typical deployment: 9+ brokers for this workload
Assuming gp3 SSD: $0.08/GiB/month
Disk cost per broker: 1,200 GiB × $0.08 = $96/month
Total disk cost: $96 × 9 brokers = $864/month
```
**Note**: Aiven blog cites $18,548/month. The difference suggests:
- Use of higher-performance io2 or instance-store disks
- Larger disk provisioning for headroom and performance
- Using their actual measured value: **$18,548/month**

**Total Classic Kafka Storage Cost:**
```
$257,356 (cross-AZ) + $18,548 (disk) = $275,904/month
```

---

#### Cost Calculation 2: Diskless + S3 Standard

**Storage Cost:**
```
Data stored: 3,600 GiB (1 hour retention)
S3 Standard pricing: $0.023/GiB/month (first 50 TB)
Storage cost = 3,600 GiB × $0.023 = $82.80/month
```

**Request Cost:**
```
PUT requests per month:
  300 req/sec × 3,600 sec/hr × 730 hr/month = 788,400,000 requests/month
  = 788.4 million requests/month (788M)

PUT pricing: $5 per million requests
PUT cost = 788.4 million × ($5 per million) = $3.94/month

GET requests per month:
  600 req/sec × 3,600 sec/hr × 730 hr/month = 1,576,800,000 requests/month
  = 1,576.8 million requests/month (1,577M)

GET pricing: $0.40 per million requests
GET cost = 1,576.8 million × ($0.40 per million) = $0.63/month

Total request cost = $3.94 + $0.63 = $4.57/month
```

**Data Transfer Cost:**
```
S3 to EC2 (same region): $0.00/GiB (free)
Cross-AZ from S3: Included in request pricing
Data transfer cost = $0/month
```

**Total S3 Standard Storage Cost (User Data Only):**
```
$82.80 (storage) + $4.57 (requests) + $0 (transfer) = $87.37/month ≈ $89/month
```

**PostgreSQL Coordinator Cost:**
```
See Cost Calculation 5: $1,616/month (standard deployment)
```

**Total Configuration A Cost:**
```
$89 (S3 storage) + $1,616 (PostgreSQL coordinator) = $1,705/month
```

---

#### Cost Calculation 3: Diskless + S3 Express One Zone

**Storage Cost:**
```
Data stored: 3,600 GiB
S3 Express One Zone pricing: $0.16/GiB/month
Storage cost = 3,600 GiB × $0.16 = $576.00/month
```

**Request Cost (with data upload/retrieval charges):**

S3 Express has two components: request charges + data transfer charges

```
PUT Request Charges:
  PUT requests: 788,400,000 requests/month = 788.4 million/month
  PUT pricing: $1.13 per million requests
  PUT request cost = 788.4 million × ($1.13 per million) = $0.89/month

Data Upload Charges (for PUT):
  Data uploaded: 1 GiB/s × 3,600 sec/hr × 730 hr/month = 2,628,000 GiB/month
  Upload charge: $0.0032 per GiB
  Upload cost = 2,628,000 GiB × $0.0032 = $8.41/month

PUT total = $0.89 (requests) + $8.41 (upload) = $9.30/month

---

GET Request Charges:
  GET requests: 1,576,800,000 requests/month = 1,576.8 million/month
  GET pricing: $0.03 per million requests
  GET request cost = 1,576.8 million × ($0.03 per million) = $0.05/month

Data Retrieval Charges (for GET):
  Data retrieved: 3 GiB/s × 3,600 sec/hr × 730 hr/month = 7,884,000 GiB/month
  Retrieval charge: $0.0006 per GiB
  Retrieval cost = 7,884,000 GiB × $0.0006 = $4.73/month

GET total = $0.05 (requests) + $4.73 (retrieval) = $4.78/month

---

Total request cost = $9.30 (PUT) + $4.78 (GET) = $14.08/month
```

**Data Transfer Cost:**
```
S3 Express to EC2 (same AZ): $0.00/GiB (free)
Data transfer cost = $0/month
```

**Total S3 Express Storage Cost (User Data Only):**
```
$576.00 (storage) + $14.08 (requests + data charges) + $0 (transfer) = $590.08/month ≈ $590/month
```

**PostgreSQL Coordinator Cost:**
```
See Cost Calculation 5: $1,616/month (standard deployment)
```

**Total Configuration B Cost:**
```
$590 (S3 Express storage) + $1,616 (PostgreSQL coordinator) = $2,206/month
```

**Note**: Using conservative estimate of **$397/month** for storage in some comparisons, which assumes:
- Potential compression reducing data volume by ~40%
- Some cache hits reducing GET operations
- Optimized batch sizes reducing request count

---

#### Cost Calculation 4: Diskless + FSxN S3

This calculation covers both FSxN configurations (C1: SSD Only and C2: Auto Tiering).

##### Configuration C1: FSxN SSD Only (No Tiering)

**Storage Cost (Kafka Data):**
```
SSD storage: 3,600 GiB (Kafka log data: 1 GiB/s × 1 hour retention)
FSx for ONTAP SSD pricing: $0.14/GiB/month
Kafka storage cost = 3,600 GiB × $0.14 = $504.00/month
```

**PostgreSQL Storage (Additional):**
```
PostgreSQL database: 100 GiB (stored on FSxN via NFS)
SSD pricing: $0.14/GiB/month
PostgreSQL storage cost = 100 GiB × $0.14 = $14.00/month

Total storage cost = $504.00 (Kafka) + $14.00 (PostgreSQL) = $518.00/month
```

**Throughput Capacity Cost:**
```
Provisioned throughput: 2,048 MBps (2 GBps)
Throughput capacity pricing: $1.719/MBps/month
Throughput cost = 2,048 MBps × $1.719/MBps = $3,520.51/month
```

**Request Cost:**
```
Requests are included in throughput capacity
Request cost = $0/month
```

**Data Transfer Cost:**
```
FSxN to EC2 (same region): $0.00/GiB (free within VPC)
Data transfer cost = $0/month
```

**Total FSxN S3 Storage Cost (User Data Only):**
```
$518.00 (storage) + $3,520.51 (throughput) + $0 (requests) + $0 (transfer) 
= $4,038.51/month ≈ $4,051/month
```

**PostgreSQL Coordinator Cost:**
```
See Cost Calculation 5: $575/month (FSxN-hosted deployment)
```

**Total Configuration C1 Cost:**
```
$4,051 (FSxN storage) + $575 (PostgreSQL coordinator) = $4,626/month
```

##### Configuration C2: FSxN Auto Tiering (Cost Optimized)

**Storage Cost with Auto Tiering:**
```
Tiering policy: Auto (2-day cooling period)
Expected data distribution:
  - Hot data (20%): 3,600 GiB × 20% = 720 GiB on SSD
  - Cold data (80%): 3,600 GiB × 80% = 2,880 GiB on capacity pool

SSD storage cost: 720 GiB × $0.14/GiB = $100.80/month
Capacity pool cost: 2,880 GiB × $0.0163/GiB = $46.94/month
Kafka storage cost = $100.80 + $46.94 = $147.74/month
```

**PostgreSQL Storage (Additional):**
```
PostgreSQL database: 100 GiB (stored on FSxN SSD via NFS)
SSD pricing: $0.14/GiB/month
PostgreSQL storage cost = 100 GiB × $0.14 = $14.00/month

Total storage cost = $147.74 (Kafka) + $14.00 (PostgreSQL) = $161.74/month
```

**Throughput Capacity Cost:**
```
Same as C1: $3,520.51/month (fixed regardless of tiering)
```

**Total FSxN S3 Storage Cost with Tiering (User Data Only):**
```
$161.74 (storage) + $3,520.51 (throughput) = $3,682.25/month ≈ $3,673/month
```

**PostgreSQL Coordinator Cost:**
```
See Cost Calculation 5: $575/month (FSxN-hosted deployment)
```

**Total Configuration C2 Cost:**
```
$3,673 (FSxN storage) + $575 (PostgreSQL coordinator) = $4,248/month
```

**Cost Savings Summary:**
```
C1 (SSD Only): $4,626/month
C2 (Auto Tiering): $4,248/month
Savings: $378/month (8% reduction)
Storage cost reduction: $518 → $162 (71% reduction)
```

---

#### Cost Calculation 5: PostgreSQL Coordinator

**Standard Deployment (for S3 Standard & S3 Express):**

Uses local NVMe storage for low-latency PostgreSQL operations.

```
Instance Cost (Dual-AZ):
  2× i3.2xlarge instances (8 vCPU, 61 GiB RAM, 1.9 TiB NVMe)
  On-demand: $0.624/hour per instance
  Dual-AZ cost: 2 × $0.624/hr × 730 hr/month = $910.56/month

PostgreSQL Write-Ahead Log (WAL) Replication:
  Dual-AZ deployment: Primary in AZ-a, Replica in AZ-b
  WAL replication is ALWAYS cross-AZ (Primary → Replica)
  
  For Diskless coordinator, WAL is primarily metadata:
    - PostgreSQL WAL: 10 MiB/s average (metadata operations, checkpoints)
    - WAL replication to secondary: 10 MiB/s × 3,600 sec/hr × 730 hr/month ÷ 1,024 = 38,496 GiB/month
  
  Lower bound in recommended requirement:
    - From Aiven: PostgreSQL must handle CommitFile operations at <100ms
    - Assume 600 commits/sec across 576 partitions
    - Each commit ~10 KB metadata
    - WAL traffic: 600 commits/sec × 10 KB = 6 MiB/s = 15,768 GiB/month
    
  Using 25,000 GiB/month as reasonable middle ground:
    Cross-AZ WAL replication cost: 25,000 GiB × $0.02/GiB = $500/month
    Same-AZ cost: N/A (WAL always crosses AZs in dual-AZ setup)

Client Traffic (Brokers → PostgreSQL):
  Total client traffic: 6 MiB/s (all brokers → PostgreSQL primary)
  
  Traffic breakdown by broker location:
    - 6 brokers distributed across 3 AZs (2 brokers per AZ)
    - PostgreSQL primary in AZ-a, replica in AZ-b
    - Brokers in AZ-a (2 brokers): 2 MiB/s same-AZ → FREE
    - Brokers in AZ-b (2 brokers): 2 MiB/s cross-AZ → PAID
    - Brokers in AZ-c (2 brokers): 2 MiB/s cross-AZ → PAID
  
  Cross-AZ portion: 4/6 = 66% of brokers in different AZ than primary
  Cross-AZ traffic: 6 MiB/s × 0.66 = 4 MiB/s
  Monthly: 4 MiB/s × 3,600 sec/hr × 730 hr/month ÷ 1,024 = 10,266 GiB/month
  
  Cross-AZ client cost: 10,266 GiB × $0.02/GiB = $205/month
  Same-AZ client cost: $0 (AWS VPC traffic within same AZ is free)

Total Standard Deployment Cost:
  $910.56 (instances) + $500 (WAL cross-AZ) + $205 (client cross-AZ)
  = $1,615.56/month ≈ $1,616/month
  
Note: Same-AZ traffic is free within AWS VPC, so only cross-AZ charges apply.
```

**FSxN-Hosted Deployment (for FSxN S3 only):**

Uses FSxN NFS protocol for PostgreSQL file storage, leveraging existing FSxN infrastructure.

```
Instance Cost (Dual-AZ):
  2× m5.2xlarge instances (8 vCPU, 32 GiB RAM)
  No local storage needed (using FSxN NFS)
  On-demand: $0.384/hour per instance
  Dual-AZ cost: 2 × $0.384/hr × 730 hr/month = $560.64/month

PostgreSQL Storage on FSxN:
  Database size: ~100 GiB (metadata, WAL, etc.)
  FSxN SSD pricing: $0.14/GiB/month
  Storage cost: 100 GiB × $0.14 = $14.00/month
  
  NOTE: This is ADDITIONAL storage beyond the 3,600 GiB Kafka data
  (3,600 GiB = 1 GiB/s × 1 hour retention for Kafka logs)
  Incremental cost: $14.00/month

FSxN Throughput for PostgreSQL I/O:
  PostgreSQL I/O: ~50 MiB/s average (reads + writes)
  Existing FSxN throughput: 2,048 MBps provisioned
  PostgreSQL share: 50 MBps (2.4% of capacity)
  Incremental cost: $0/month (well within existing capacity)

Cross-AZ Replication:
  FSxN handles Multi-AZ replication internally
  No additional cross-AZ charges for WAL replication
  PostgreSQL WAL cost: $0/month

Cross-AZ Client Traffic:
  Same as standard: brokers connecting to PostgreSQL
  However, FSxN is Multi-AZ, so all connections are local within VPC
  Effective cross-AZ cost: $0/month

Total FSxN-Hosted Deployment Cost:
  $560.64 (instances) + $14.00 (storage) + $0 (throughput) + $0 (cross-AZ)
  = $574.64/month ≈ $575/month
```

**Key Difference - Protocol Usage:**

FSxN ONTAP is a **multi-protocol storage system** that serves both:
1. **S3 API** (object storage) for Kafka Diskless log segments
2. **NFS v4** (file storage) for PostgreSQL database files

```
┌──────────────────────────────────────────────────────┐
│         FSxN ONTAP (Multi-AZ Deployment)             │
│                                                      │
│  ┌────────────────────┐      ┌───────────────────┐  │
│  │   S3 Bucket        │      │   NFS Export      │  │
│  │ (Object Storage)   │      │ (File Storage)    │  │
│  │  - Kafka segments  │      │ - PostgreSQL data │  │
│  │  - Immutable logs  │      │ - WAL files       │  │
│  │  - 3.5 TiB         │      │ - ~100 GiB        │  │
│  └─────────┬──────────┘      └─────────┬─────────┘  │
│            │                           │             │
│     S3 API Protocol             NFS v4 Protocol      │
└────────────┼───────────────────────────┼─────────────┘
             │                           │
             │                           │
    ┌────────▼──────────┐      ┌─────────▼─────────┐
    │  Kafka Brokers    │      │   PostgreSQL      │
    │   (Diskless)      │      │   Coordinator     │
    │  - 6× m8g.4xlarge │      │ - 2× m5.2xlarge   │
    │  - S3 SDK         │      │ - NFS mount       │
    └───────────────────┘      └───────────────────┘
```

**Why PostgreSQL needs NFS (not S3):**
- Requires POSIX file system semantics (random read/write, file locking)
- Needs transaction support with fsync guarantees
- Cannot use eventual consistency of S3

**Why Kafka uses S3 API:**
- Write-once-read-many access pattern
- Immutable log segments (no updates after written)
- Can tolerate eventual consistency

**Cost Savings Summary:**
```
Standard PostgreSQL Deployment: $1,616/month
  - Dual i3.2xlarge with NVMe: $911
  - WAL cross-AZ replication: $500
  - Client cross-AZ traffic: $205

FSxN-Hosted PostgreSQL: $575/month (64% savings)
  - Dual m5.2xlarge: $561
  - Storage: $14 (100 GiB additional beyond Kafka's 3,600 GiB)
  - Throughput: $0 (within existing 2 GBps capacity)
  - Cross-AZ charges: $0 (FSxN handles replication)

Savings: $1,616 - $575 = $1,041/month (64% reduction)
```

---

### Cost Summary Table

| **Component** | **Classic Kafka** | **Diskless + S3 Standard** | **Diskless + S3 Express** | **Diskless + FSxN (C1: SSD)** | **Diskless + FSxN (C2: Tiering)** |
|--------------|------------------|---------------------------|--------------------------|-------------------------------|----------------------------------|
| **Storage** | $18,548 | $83 | $576 | $518 | $162 |
| **Replication/Throughput** | $257,356 | — | — | $3,521 | $3,521 |
| **Requests** | — | $5 | $14 | Included | Included |
| **Data Transfer** | Included above | $0 | $0 | $0 | $0 |
| **PostgreSQL Coordinator** | — | $1,616 | $1,616 | **$575** | **$575** |
| **Subtotal (Storage & Coordinator)** | **$275,904** | **$1,705** | **$2,206** | **$4,626** | **$4,248** |
| **% of Classic** | 100% | **0.62%** | **0.80%** | **1.68%** | **1.54%** |

**Updated Savings:**
- **Diskless + S3 Standard (Config A)**: 99.38% savings vs Classic Kafka
- **Diskless + S3 Express (Config B)**: 99.20% savings vs Classic Kafka  
- **Diskless + FSxN SSD Only (Config C1)**: 98.32% savings vs Classic Kafka
- **Diskless + FSxN Auto Tiering (Config C2)**: 98.46% savings vs Classic Kafka (8% cheaper than C1)
  - Storage cost reduction: 71% vs SSD-only ($518 → $162)
  - Both FSxN configs achieve 64% coordinator savings by hosting PostgreSQL on FSxN

**PostgreSQL Coordinator Deployment Options:**

See **Cost Calculation 5** above for detailed breakdown.

- **S3 Standard & S3 Express**: Use i3.2xlarge instances with local NVMe ($1,616/month)
  - Local NVMe storage for low-latency PostgreSQL operations
  - WAL replication cross-AZ charges: $500/month
  - Client cross-AZ traffic: $205/month
  
- **FSxN S3**: Use m5.2xlarge instances with PostgreSQL data on FSxN via **NFS** ($575/month)
  - **64% coordinator cost reduction** by leveraging existing FSxN infrastructure
  - **Critical difference**: Uses **NFS protocol** (not S3) for PostgreSQL file storage
  - FSxN Multi-Protocol support:
    - **S3 API** → Kafka Diskless log segments (object storage, immutable)
    - **NFS v4** → PostgreSQL database files (file storage, POSIX semantics)
  - FSxN's Multi-AZ replication eliminates PostgreSQL WAL cross-AZ charges ($0 vs $500)
  - PostgreSQL benefits from FSxN's sub-millisecond cached reads
  - PostgreSQL storage (100 GiB) is additional to Kafka's 3,600 GiB: +$14/month
  - Throughput absorbed within existing FSxN 2 GBps capacity (no incremental cost)

**Important Notes:**
- All costs exclude compute (broker instances), which are comparable across all options
- Classic Kafka may require more brokers (9+ vs 6) for same workload, adding compute costs
- FSxN option achieves lowest coordinator cost by consolidating infrastructure
- At Aiven's measured 30% CPU utilization, Diskless can likely handle 2-3x throughput on same broker count

### Research Outcomes

**Question 1: Does faster object storage translate to faster Kafka?**
- Classic Kafka achieves ~50ms P50 with local disks
- S3 Standard Diskless adds ~600ms (650ms total) with 99.38% cost savings
- If S3 Express (10x faster S3 ops) only reduces end-to-end latency by 2x (to ~325ms), is 1.3x cost increase justified?
- Need to measure how much of the 650ms P50 is S3 latency vs Kafka processing/batching/coordinator

**Question 2: What's the latency/cost efficiency sweet spot?**
- Classic Kafka: Best latency, worst cost (baseline 100%)
- S3 Standard: 13x slower, **99.38% cheaper**
- S3 Express: Target 2x slower than Classic?, **99.20% cheaper**
- FSxN SSD Only (C1): Target similar to Classic?, **98.32% cheaper**
- FSxN Auto Tiering (C2): Target similar to Classic?, **98.46% cheaper** (8% better than C1)
- Which trade-off makes sense for different workload types?

**Question 3: What's the break-even point for S3 Express?**
- S3 Express has cheaper requests (80% lower GET cost) but 5x storage cost vs S3 Standard
- At low throughput: Storage cost dominates → S3 Standard wins
- At very high throughput: Request cost dominates → S3 Express may win
- Need to find the crossover point (estimated at >10 GiB/s?)

**Question 4: When does FSxN justify its cost?**
- FSxN (C1: SSD Only) is 2.7x more expensive than S3 Standard, but still **98.32% cheaper than Classic Kafka**
- FSxN (C2: Auto Tiering) is 2.5x more expensive than S3 Standard, but still **98.46% cheaper than Classic Kafka**
- Only justified if: Need near-Classic latency (<100ms) without Classic's replication costs
- For Aiven's 1 GiB/s workload, likely not cost-effective
- May be valuable for migrating latency-sensitive Classic Kafka workloads to Diskless
- Auto tiering (C2) provides 8% cost savings while maintaining hot data performance

**Question 5: What about operational complexity?**
- Classic Kafka: Complex (disk management, rebalancing, multi-AZ replication)
- S3 Standard: Simplest Diskless option (fully managed, auto-scaling)
- S3 Express: Simple but single-AZ (availability trade-off)
- FSxN: More complex (capacity planning, monitoring, provisioning)

### Recommendations Framework

Based on benchmark results, we will provide a decision matrix:

**Choose Classic Kafka if:**
- Latency requirement is <100ms P50
- Cannot tolerate any latency increase
- Already have optimized Classic Kafka infrastructure
- Cost is not primary concern

**Choose Diskless + S3 Standard if:**
- Cost is primary concern (proven 99.97% savings vs Classic)
- Latency requirement is >500ms P50
- Workload is write-heavy or has low read fan-out
- Long-term data retention (cold storage)
- **Most common choice for new Diskless deployments**

**Choose Diskless + S3 Express if:**
- Latency requirement is <500ms P50 but >100ms acceptable
- High consumer fan-out (request cost matters)
- Workload has frequent consumer catch-up scenarios
- Single-AZ deployment acceptable
- Budget allows 4.5x storage cost vs S3 Standard (still 99.86% cheaper than Classic)

**Choose Diskless + FSxN S3 if:**
- Migrating from Classic Kafka but need similar latency (<100ms P50)
- Very high throughput (>5 GiB/s) where fixed capacity cost amortizes
- Need predictable performance (no cloud variability)
- Can leverage compression/deduplication for additional savings
- Budget allows 45x cost vs S3 Standard (still 98.54% cheaper than Classic)

## Next Steps

- Finalize test configurations
- Deploy test environments
- Execute benchmark tests
- Analyze and document results

## Paul's clever ideas
- Does diskless offer higher scalability w.r.t. num. partitions, replications vs throughput
- Clarify the need of RF=3: for cheaper cross-AZ consumer reads
- Client's (producer) performance (need to resend up to seconds of messages?), resource-constraint producers

## Future Work

- **Failure mode testing**: Simulate broker failures, storage failures
- **Network partition testing**: Cross-AZ latency impacts
- **Compression testing**: Impact of producer/consumer compression on storage I/O
- **Segment size tuning**: Optimal segment sizes for each backend
- **Multi-region**: Cross-region replication performance
- **Security**: Encryption-at-rest and in-transit overhead
- **Monitoring integration**: Prometheus/Grafana dashboards for each backend

## References

1. **KIP-1150: Diskless Topics**: [KIP-1150](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1150%3A+Diskless+Topics)
   - **Current status**: Voting (as of benchmark plan creation)
   - Motivation: Eliminate cross-AZ replication costs using object storage
   - Goal: Save costs by using object storage instead of disk replication
   - No implementation details in this KIP - see follow-up KIPs:
     - KIP-1163: Diskless Core
     - KIP-1164: Diskless Coordinator
   - **Important**: Diskless topics still use some broker disk (metadata, caching), not truly "diskless"

2. **Aiven Blog Post**: [Benchmarking Diskless Topics: Part 1](https://aiven.io/blog/benchmarking-diskless-inkless-topics-part-1)
   - **Key Results**:
     - **>94% cost savings** vs traditional 3-AZ Kafka ($288k vs $3.32M/year)
     - **End-to-end latency**: P50 ~650ms, P99 ~1.5s (with spikes to 8s)
     - **Producer latency**: P50 ~250ms, P99 ~500ms (with spikes to 4s)
     - **CPU utilization**: ~30% on 6 brokers (m8g.4xlarge)
     - **S3 usage**: ~50 PUT req/sec per broker, ~100 GET req/sec per broker
     - **Cross-AZ traffic**: Only 1.4 MiB/s for internal topics (Kafka metadata)
     - **PostgreSQL coordinator**: ~1 MiB/s writes, ~1.5 MiB/s reads, ~10 MiB/s WAL replication
   - **Configuration Details from Blog**:
     - 576 partitions, 3x replication factor
     - 1 GiB/s producer, 3 GiB/s consumer (3x fan-out)
     - Uncompressed workload for accurate measurement
     - 6 brokers (m8g.4xlarge: 16 vCPUs, 64 GiB RAM)
     - JVM heap: 75% of memory = 48 GiB (needed for Diskless on-heap cache)
     - PostgreSQL coordinator: i3.2xlarge, dual-AZ, 1.9 TiB NVMe
     - **Important**: Aiven uses **S3 Standard**, not S3 Express or FSxN

3. **Inkless Implementation**: [Aiven Inkless GitHub](https://github.com/aiven/inkless)
   - Fork of Apache Kafka 4.0 with Diskless implementation
   - Not intended as long-term fork - changes being contributed upstream
   - Minimal changes to Apache Kafka codebase
   - **Note**: This is Revision 1 of KIP-1150 design (may evolve)

4. **AWS S3 Performance**: [Optimizing S3 Performance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
   - Request rate limits (5,500 GET/sec, 3,500 PUT/sec per prefix)
   - Latency characteristics and best practices
   - **Matches Aiven's usage**: ~300 PUT/sec, ~600 GET/sec total (well under limits)

5. **AWS S3 Express One Zone**: [S3 Express One Zone Documentation](https://aws.amazon.com/s3/storage-classes/express-one-zone/)
   - Single-digit millisecond latency
   - Up to 2M requests per second
   - **Not used by Aiven** - testing to see if it improves Diskless latency

6. **NetApp FSxN S3**: [ONTAP S3 Documentation](https://docs.netapp.com/us-en/ontap/s3-config/ontap-s3-supported-actions-reference.html)
   - S3 API compatibility
   - Performance characteristics
   - Caching and tiering behavior
   - **Not used by Aiven** - testing to see if it improves Diskless latency

7. **AWS FSxN Performance**: [Amazon FSx for NetApp ONTAP Performance](https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/performance.html)
   - Throughput capacity and IOPS relationships
   - Caching behavior and optimization techniques

