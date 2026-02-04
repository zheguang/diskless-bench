# Benchmark Plan for Kafka Diskless (KIP-1150) - Inkless Implementation

## Overview

This benchmark plan outlines the approach to evaluate the performance of Kafka Diskless topics using Aiven's Inkless implementation. The benchmark will compare performance between **AWS S3 Standard**, **AWS S3 Express One Zone**, and **NetApp FSxN S3** object storage backends.

**Key Correction**: This benchmark is for **object storage** (S3-compatible), not traditional block/file storage. Kafka Diskless/Inkless stores log segments directly in S3-compatible object storage.

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

Based on the comparison of the three storage backends, we expect to map out a clear cost/performance trade-off spectrum:

| **Storage** | **Monthly Cost (3.6 TiB)** | **Target End-to-End P50** | **Target End-to-End P99** | **Best For** |
|-------------|---------------------------|---------------------------|---------------------------|--------------|
| **S3 Standard** | **$89** (baseline) | ~650ms | ~1.5s | Cost-sensitive, long-term storage, proven at scale |
| **S3 Express** | **$397** (~4.5x) | <300ms? | <800ms? | Latency-sensitive streaming, high request rates |
| **FSxN S3** | **$4,024** (~45x) | <100ms? | <500ms? | Ultra-low latency, predictable performance, very high throughput |

### Research Outcomes

**Question 1: Does faster object storage translate to faster Kafka?**
- If S3 Express (10x faster S3 ops) only reduces end-to-end latency by 2x, it may not justify 4.5x cost
- Need to measure how much of the 650ms P50 is S3 latency vs Kafka processing/batching

**Question 2: What's the break-even point for S3 Express?**
- S3 Express has cheaper requests (80% lower GET cost) but 5x storage cost
- At low throughput: Storage cost dominates → S3 Standard wins
- At very high throughput: Request cost dominates → S3 Express may win
- Need to find the crossover point

**Question 3: When does FSxN justify its cost?**
- FSxN is 45x more expensive than S3 Standard
- Only justified if: Ultra-low latency requirement (<100ms) OR very high throughput (>5 GiB/s)
- For Aiven's 1 GiB/s workload, likely not cost-effective
- May be valuable for specific use cases (real-time analytics, fraud detection)

**Question 4: What about operational complexity?**
- S3 Standard: Simplest (fully managed, auto-scaling)
- S3 Express: Simple but single-AZ (availability trade-off)
- FSxN: Most complex (capacity planning, monitoring, provisioning)

### Recommendations Framework

Based on benchmark results, we will provide a decision matrix:

**Choose S3 Standard if:**
- Cost is primary concern (proven 94% savings)
- Latency requirement is >500ms P50
- Workload is write-heavy or has low read fan-out
- Long-term data retention (cold storage)

**Choose S3 Express if:**
- Latency requirement is <500ms P50
- High consumer fan-out (request cost matters)
- Workload has frequent consumer catch-up scenarios
- Single-AZ deployment acceptable

**Choose FSxN S3 if:**
- Latency requirement is <100ms P50
- Very high throughput (>5 GiB/s)
- Need predictable performance (no cloud variability)
- Budget allows for 45x cost premium
- Can leverage compression/deduplication for additional savings

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

