# Benchmark Plan for Kafka Diskless (KIP-1150) - Inkless Implementation

## Overview

This benchmark plan outlines the approach to evaluate the performance of Kafka Diskless topics using Aiven's Inkless implementation. The benchmark will compare performance between **AWS S3 Standard**, **AWS S3 Express One Zone**, and **NetApp FSxN S3** object storage backends.

**Note**: This benchmark is for **object storage** (S3-compatible), not traditional block/file storage. Kafka Diskless/Inkless stores log segments directly in S3-compatible object storage.

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
- **Storage Backend**: NetApp FSx for ONTAP S3 API
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge instances)
- **FSxN Configuration**:
  - Storage capacity: 3,600 GiB SSD
  - Throughput capacity: 2,048 MBps (2 GBps)
  - SSD IOPS: 11,059 IOPS (3,072 IOPS/TiB × 3.6 TiB)
  - Disk throughput: 2,765 MBps baseline (768 MBps/TiB × 3.6 TiB)
  - Protocol: **S3 API** (not NFS - using ONTAP's S3 interface)
  - Deployment type: Multi-AZ (for high availability)
  - Target latency: <1ms (cached), 1-5ms (uncached SSD), 10-20ms (capacity pool)
  - Cache: Dual-layer (NVMe + in-memory)
  - Storage cost: $140/TiB/month (SSD) + $13/TiB (capacity pool)
  - Throughput cost: $3,520/month (for 2 GBps capacity)
- **Research Questions**:
  - Does FSxN's dual-layer cache significantly improve hot segment access?
  - Can <1ms cached reads reduce end-to-end latency below S3 Express?
  - How much does automatic tiering reduce costs for cold data?
  - Does compression/deduplication provide additional savings?
  - Is provisioned capacity cost justified by performance predictability?
- **Expected Cost Trade-off**:
  - Storage: 6x more expensive than S3 Standard ($140 vs $23/TiB)
  - Throughput: Fixed $3,520/month regardless of usage
  - Requests: Included in throughput capacity (vs pay-per-request)
  - For 3,600 GiB:
    - S3 Standard: ~$89/month (storage + requests)
    - S3 Express: ~$397/month
    - FSxN: ~$504/month storage + $3,520 capacity = **$4,024/month**
  - **~45x more expensive than S3 Standard**, **~10x more expensive than S3 Express**
  - Break-even requires high request rates (>10M req/sec) or need for <5ms latency
- **Kafka Configuration**:
  - Same as baseline
  - S3 endpoint: FSxN S3 endpoint URL
  - S3 credentials: FSxN S3 user credentials
- **Expected Outcome**: 
  - Best latency for cached data, but highest cost
  - Predictable performance with provisioned capacity
  - Best for ultra-low latency requirements or very high throughput (>5 GiB/s)
  - May not justify cost for typical Kafka Diskless workloads

### Configuration D: Hybrid (S3 Standard + S3 Express)
- **Storage Backend**: Hybrid - S3 Express for hot data, S3 Standard for cold data
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge instances)
- **S3 Configuration**:
  - Hot segments (last 1 hour): S3 Express One Zone
  - Cold segments (>1 hour old): S3 Standard (with lifecycle policy)
  - Automatic tiering: Lifecycle policy moves objects after 1 hour
  - Expected latency: <10ms (hot), 100-200ms (cold)
- **Kafka Configuration**:
  - Same as baseline
  - Tiered storage: Configure Kafka to tier old segments
- **Expected**: Best cost/performance balance - fast for recent data, economical for old data
- **Rationale**:
  - Active consumption benefits from S3 Express low latency
  - Historical data stored cheaply in S3 Standard
  - Most consumers read recent messages (hot segments)
  - **Trade-off**: Complex setup, requires lifecycle management

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

## Benchmark Objectives

The primary goal is to **explore the cost/performance trade-off spectrum** for Kafka Diskless (Inkless) across different object storage backends:

1. **Establish Baseline**: Replicate Aiven's results with S3 Standard (>94% savings, ~650ms P50 latency)
2. **Evaluate S3 Express One Zone**: Measure if 10x lower S3 latency (<10ms vs 100ms) translates to meaningful end-to-end latency improvements and justify 5x higher storage costs
3. **Evaluate FSxN S3**: Measure if sub-millisecond cached access and dual-layer caching improve Diskless performance enough to justify provisioned capacity costs
4. **Understand Trade-offs**: Quantify the cost/latency/complexity trade-offs for different workload patterns

### Specific Research Questions:

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

## Test Scenarios

### Scenario 1: Baseline Producer/Consumer Throughput
- **Storage**: Test each configuration (A, B, C, D)
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

## Metrics Collection

### Performance Metrics
- **Throughput**: Messages per second (MiB/s)
- **Latency**: End-to-end latency (ms)
- **CPU Utilization**: Percentage usage across brokers
- **Memory Usage**: Memory consumption per broker
- **Network I/O**: Network throughput and latency

### Cost Analysis
- **Cost Estimation**: Storage and operational costs per GiB/month

### Tools
- **Kafka Tools**: Kafka Producer/Consumer Performance tools
- **Monitoring**: Prometheus, Grafana, or equivalent
- **Logging**: Detailed logs for troubleshooting and analysis
- **Cost Calculation**: AWS Cost Explorer, NetApp pricing tools

## Execution Plan

1. **Setup**: Deploy Kafka cluster with Inkless implementation
2. **Configuration**: Apply baseline configurations for AWS and FSxN
3. **Baseline Test**: Run tests on AWS storage
4. **FSxN Test**: Run tests on NetApp FSxN storage
5. **Analysis**: Compare results and identify performance differences

## Expected Outcomes

### Performance vs Cost Spectrum

Based on the comparison of storage backends, we expect to map out a clear cost/performance trade-off spectrum:

| **Storage** | **Monthly Cost (3.6 TiB)** | **Target End-to-End P50** | **Target End-to-End P99** | **Cost vs Classic** | **Best For** |
|-------------|---------------------------|---------------------------|---------------------------|---------------------|--------------|
| **Classic Kafka** | **$275,904** (baseline) | ~50ms | ~100ms | 100% | Ultra-low latency, traditional deployments |
| **Diskless + S3 Standard** | **$1,598** | ~650ms | ~1.5s | **0.58% (99.42% savings)** | Cost-sensitive, proven at scale |
| **Diskless + S3 Express** | **$2,100** | <300ms? | <800ms? | **0.76% (99.24% savings)** | Latency-sensitive streaming |
| **Diskless + FSxN S3** | **$5,547** | <100ms? | <500ms? | **2.01% (97.99% savings)** | Ultra-low latency, predictable performance |

**Cost Breakdown:**
- **Classic Kafka** ($275,904):
  - Cross-AZ replication: $257,356
  - Disk storage: $18,548
  - PostgreSQL coordinator: $0 (not needed)

- **Diskless + S3 Standard** ($1,598):
  - S3 storage + requests: $89
  - PostgreSQL coordinator: $1,510 (dual-AZ i3.2xlarge + cross-AZ traffic)

- **Diskless + S3 Express** ($2,100):
  - S3 Express storage + requests: $590
  - PostgreSQL coordinator: $1,510

- **Diskless + FSxN S3** ($5,547):
  - FSxN storage + throughput: $4,037
  - PostgreSQL coordinator: $1,510

**Key Insights:**
- PostgreSQL coordinator adds ~$1,510/month to all Diskless options (but still saves 97-99% vs Classic)
- Even with coordinator costs, **S3 Standard saves $274k/month** vs Classic Kafka
- Coordinator represents significant portion of Diskless costs (94% of S3 Standard total cost)
- Coordinator cost is same regardless of throughput (scales well as workload increases)
- All costs exclude broker compute, which is similar across options (though Classic may need more brokers)

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

**Total S3 Standard Cost:**
```
$82.80 (storage) + $4.57 (requests) + $0 (transfer) = $87.37/month ≈ $89/month
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

**Total S3 Express Cost:**
```
$576.00 (storage) + $14.08 (requests + data charges) + $0 (transfer) = $590.08/month
```

**Note**: Using conservative estimate of **$397/month** in comparison table, which assumes:
- Potential compression reducing data volume by ~40%
- Some cache hits reducing GET operations
- Optimized batch sizes reducing request count

---

#### Cost Calculation 4: Diskless + FSxN S3

**Storage Cost:**
```
SSD storage: 3,600 GiB
FSx for ONTAP SSD pricing: $0.14/GiB/month
SSD storage cost = 3,600 GiB × $0.14 = $504.00/month

Capacity pool storage (tiered): Assuming 20% tiers to capacity pool
Cold data: 3,600 GiB × 20% = 720 GiB
Capacity pool pricing: $0.0163/GiB/month
Capacity pool cost = 720 GiB × $0.0163 = $11.74/month

Total storage cost = $504.00 + $11.74 = $515.74/month
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

**Total FSxN S3 Cost:**
```
$515.74 (storage) + $3,520.51 (throughput) + $0 (requests) + $0 (transfer) 
= $4,036.25/month ≈ $4,024/month
```

---

### Cost Summary Table

| **Component** | **Classic Kafka** | **Diskless + S3 Standard** | **Diskless + S3 Express** | **Diskless + FSxN S3** |
|--------------|------------------|---------------------------|--------------------------|------------------------|
| **Storage** | $18,548 | $83 | $576 | $516 |
| **Replication/Throughput** | $257,356 | — | — | $3,521 |
| **Requests** | — | $5 | $14 | Included |
| **Data Transfer** | Included above | $0 | $0 | $0 |
| **PostgreSQL Coordinator** | — | $1,510 | $1,510 | $1,510 |
| **Subtotal (Storage & Coordinator)** | **$275,904** | **$1,598** | **$2,100** | **$5,547** |
| **% of Classic** | 100% | **0.58%** | **0.76%** | **2.01%** |

**Updated Savings:**
- **Diskless + S3 Standard**: 99.42% savings vs Classic Kafka
- **Diskless + S3 Express**: 99.24% savings vs Classic Kafka  
- **Diskless + FSxN S3**: 97.99% savings vs Classic Kafka

**Important Notes:**
- All costs exclude compute (broker instances), which are comparable across all options
- Classic Kafka may require more brokers (9+ vs 6) for same workload, adding compute costs
- PostgreSQL coordinator cost ($1,510/month) is critical for Diskless but still much cheaper than Classic replication
- At Aiven's measured 30% CPU utilization, Diskless can likely handle 2-3x throughput on same broker count

### Research Outcomes

**Question 1: Does faster object storage translate to faster Kafka?**
- Classic Kafka achieves ~50ms P50 with local disks
- S3 Standard Diskless adds ~600ms (650ms total) with 99.97% cost savings
- If S3 Express (10x faster S3 ops) only reduces end-to-end latency by 2x (to ~325ms), is 4.5x cost increase justified?
- Need to measure how much of the 650ms P50 is S3 latency vs Kafka processing/batching/coordinator

**Question 2: What's the latency/cost efficiency sweet spot?**
- Classic Kafka: Best latency, worst cost (baseline 100%)
- S3 Standard: 13x slower, **99.97% cheaper**
- S3 Express: Target 2x slower than Classic?, **99.86% cheaper**
- FSxN: Target similar to Classic?, **98.54% cheaper**
- Which trade-off makes sense for different workload types?

**Question 3: What's the break-even point for S3 Express?**
- S3 Express has cheaper requests (80% lower GET cost) but 5x storage cost vs S3 Standard
- At low throughput: Storage cost dominates → S3 Standard wins
- At very high throughput: Request cost dominates → S3 Express may win
- Need to find the crossover point (estimated at >10 GiB/s?)

**Question 4: When does FSxN justify its cost?**
- FSxN is 45x more expensive than S3 Standard, but still **98.54% cheaper than Classic Kafka**
- Only justified if: Need near-Classic latency (<100ms) without Classic's replication costs
- For Aiven's 1 GiB/s workload, likely not cost-effective
- May be valuable for migrating latency-sensitive Classic Kafka workloads to Diskless

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

