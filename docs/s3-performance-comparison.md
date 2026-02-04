# Performance Comparison: NetApp FSxN S3 vs AWS S3

## Overview

This document compares the performance characteristics of NetApp FSxN S3 (FSx for ONTAP S3) and AWS S3 for benchmarking Kafka Diskless (Aiven's Inkless implementation of KIP-1500) workloads.

## Architecture & Throughput Comparison

### Understanding FSxN vs S3 Throughput

The throughput numbers can be confusing. Here's what they actually mean:

| **System** | **Per Node/Instance** | **Max Aggregate** | **Architecture** |
|------------|----------------------|-------------------|------------------|
| **FSxN (1 HA pair)** | ~3 GBps per node | **6 GBps** | 2 nodes (active + standby), single namespace |
| **FSxN (12 HA pairs)** | ~3 GBps per node | **72 GBps** | 24 nodes total, single namespace |
| **S3 (1 EC2 client)** | **12.5 GBps** (100 Gbps) | 12.5 GBps | 1 instance accessing S3 |
| **S3 (multiple clients)** | 12.5 GBps each | **Unlimited** | N instances, must partition data across prefixes |

### Key Architectural Differences

**FSxN Approach:**
- 1 HA Pair = **2 nodes** (1 active, 1 standby for failover)
- Gen2 Single-AZ supports **1-12 HA pairs** (2-24 nodes)
- All nodes serve a **single unified namespace**
- All Kafka brokers access the **same filesystem**
- Aggregate: 12 pairs × 6 GBps = **72 GBps to same data**

**S3 Approach:**
- Each EC2 instance/client can achieve **12.5 GBps** (100 Gbps) network throughput
- To exceed 12.5 GBps, deploy **multiple EC2 instances**
- Must **partition data** across S3 prefixes for best performance
- Each client typically accesses **different data subsets**
- Aggregate: **Virtually unlimited** with proper partitioning

**For Kafka Diskless:**
- Multiple Kafka brokers will access the **same segments** (producer writes, follower reads, consumer reads)
- FSxN advantage: Single namespace with high aggregate throughput to **same data**
- S3 limitation: Each broker limited to ~12.5 GBps, but that's often sufficient
- **Typical Kafka cluster**: 1-10 GBps total is common, so S3's 12.5 GBps per client is usually plenty

## Performance Comparison Table

> **Note on FSxN "Throughput Capacity"**: FSx for ONTAP requires you to provision a "throughput capacity" when creating the file system (e.g., 128 MBps, 512 MBps, 2 GBps, etc.). This setting determines your file system's network bandwidth, cache size, and overall performance. Higher throughput capacity = better performance but higher cost.

| **Metric** | **NetApp FSxN S3 (FSx for ONTAP S3)** | **AWS S3 Standard** | **AWS S3 Express One Zone** |
|------------|--------------------------------------|---------------------|----------------------------|
| **Latency (First-Byte)** | **<1ms** (SSD tier, cached)<br>1-5ms (SSD tier, uncached)<br>10-20ms (capacity pool) | **100-200ms** (typical)<br>Can vary by region/distance | **<10ms** (single-digit millisecond)<br>**10x faster** than S3 Standard |
| **GET/HEAD Requests** | **50,000-200,000 req/sec**<br>(scales with provisioned capacity) | **5,500 req/sec** per prefix<br>(scales with multiple prefixes) | **Up to 2,000,000 req/sec**<br>per directory bucket |
| **PUT/POST/DELETE Requests** | **25,000-100,000 req/sec**<br>(scales with provisioned capacity) | **3,500 req/sec** per prefix<br>(scales with multiple prefixes) | **Up to 2,000,000 req/sec**<br>per directory bucket |
| **Max Throughput** | **Per File System:**<br>• 128-4,096 MBps (Gen1, 1 HA pair)<br>• 3-6 GBps per HA pair (Gen2)<br>• Up to 72 GBps (Gen2 with 12 HA pairs)<br><br>**Per HA Pair (2 nodes):**<br>• ~6 GBps max | **Up to 12.5 GBps** (100 Gbps) per EC2 instance<br>**Multiple TBps** aggregate | **Up to 12.5 GBps** (100 Gbps) per instance<br>Network-limited |
| **Baseline Network Throughput** | **312-10,000 MBps**<br>(varies by throughput capacity) | **Varies by EC2 instance:**<br>• m5.large: 0.75 GBps (Up to 10 Gbps)<br>• m5.2xlarge: 1.25 GBps (Up to 10 Gbps)<br>• m5.8xlarge: 1.25 GBps (10 Gbps)<br>• m5.16xlarge: 2.5 GBps (20 Gbps)<br>• m5n.24xlarge: 12.5 GBps (100 Gbps) | **Varies by EC2 instance:**<br>• m5.large: 0.75 GBps (Up to 10 Gbps)<br>• m5.2xlarge: 1.25 GBps (Up to 10 Gbps)<br>• m5.8xlarge: 1.25 GBps (10 Gbps)<br>• m5.16xlarge: 2.5 GBps (20 Gbps)<br>• m5n.24xlarge: 12.5 GBps (100 Gbps) |
| **Burst Network Throughput** | **625-20,000 MBps**<br>(2x baseline, varies by capacity) | N/A (auto-scales) | N/A (auto-scales) |
| **IOPS** | **3,072 IOPS/TiB baseline**<br>768 MBps/TiB disk throughput<br>**Up to millions** with provisioning | **No explicit limit**<br>Scales with parallelization | **Hundreds of thousands**<br>per directory bucket |
| **Storage Capacity** | **1 TiB - 1 PiB** per file system | **Unlimited** | **Unlimited** |
| **Cache Performance** | **Dual-layer:**<br>• In-memory cache<br>• NVMe cache (if enabled)<br>• Sub-ms for cached data | None (requires CloudFront/ElastiCache) | None (low latency native) |
| **Durability** | 99.999999999% (11 nines) | 99.999999999% (11 nines) | 99.999999999% (11 nines) |
| **Availability** | **99.99%** (Multi-AZ)<br>99.9% (Single-AZ) | **99.99%** (≥3 AZs) | **99.95%** (Single AZ) |
| **API Compatibility** | Full S3 API + extensions | Native S3 API | S3 API (directory buckets) |
| **Data Efficiency** | • Compression<br>• Deduplication<br>• Compaction<br>• Auto-tiering | None | None |
| **Protocol Support** | **Multi-protocol:**<br>S3, NFS, SMB, iSCSI, NVMe | S3 only | S3 only |
| **Consistency** | Strong consistency | Strong consistency | Strong consistency |

### FSx for ONTAP Throughput Capacity Explained

When you create an FSx for ONTAP file system, you must choose a **throughput capacity** level. This is different from S3 which auto-scales. Think of it like choosing an EC2 instance size.

#### Architecture Overview

**HA Pair Structure:**
- **1 HA Pair** = 2 nodes (1 active, 1 standby for failover)
- **Gen1 (older)**: 1 HA pair per file system, max 4 GBps
- **Gen2 Single-AZ**: 1-12 HA pairs per file system, max 6 GBps per pair
- **Gen2 Multi-AZ**: 1 HA pair per file system, max 6 GBps

**The 72 GBps number** = 12 HA pairs × 6 GBps per pair (only for Gen2 Single-AZ)

#### Throughput Capacity Tiers (Gen1, 1 HA pair)

| **Throughput Capacity** | **Network Baseline** | **Network Burst** | **Memory Cache** | **Est. IOPS** | **Monthly Cost** |
|------------------------|---------------------|-------------------|------------------|---------------|-----------------|
| 128 MBps | 312 MBps | 625 MBps | 8 GB | ~25k | ~$220 |
| 256 MBps | 625 MBps | 1,250 MBps | 16 GB | ~50k | ~$440 |
| 512 MBps | 1,250 MBps | 2,500 MBps | 32 GB | ~100k | ~$880 |
| 1,024 MBps (1 GBps) | 2,500 MBps | 5,000 MBps | 64 GB | ~150k | ~$1,760 |
| 2,048 MBps (2 GBps) | 5,000 MBps | 10,000 MBps | 128 GB | ~200k | ~$3,520 |
| 4,096 MBps (4 GBps) | 10,000 MBps | 20,000 MBps | 256 GB | ~300k | ~$7,040 |

**Gen2 Single-AZ**: You can add up to 12 HA pairs, each providing 3-6 GBps, for aggregate throughput up to 72 GBps (but costs scale linearly with pairs).

**Key Point**: Unlike S3 which scales automatically, FSxN requires upfront capacity planning. You pay for the throughput capacity regardless of usage.

### Cost Comparison (Estimated Monthly - us-east-1)

| **Component** | **FSxN S3** | **S3 Standard** | **S3 Express One Zone** |
|--------------|-------------|----------------|------------------------|
| **Storage** | $125-140/TiB (SSD)<br>$13/TiB (capacity pool) | $23/TiB | $110/TiB |
| **Throughput Capacity** | $220-3,520/month<br>(must provision, fixed cost) | $0 (auto-scales) | $0 (auto-scales) |
| **PUT Requests** | Included in capacity | $5 per million | $1.13 per million |
| **GET Requests** | Included in capacity | $0.40 per million | $0.03 per million |
| **Data Transfer Out** | $90/TiB (internet) | $90/TiB (internet) | $90/TiB (internet) |

## Key Differences for Kafka Diskless/Inkless

### Why FSxN Can Have Higher Aggregate Throughput

**FSxN's advantage isn't "per node" - it's architectural:**

**FSxN for High Throughput:**
```
File System with 12 HA Pairs (24 nodes total)
├── HA Pair 1 (2 nodes) → 6 GBps
├── HA Pair 2 (2 nodes) → 6 GBps
├── HA Pair 3 (2 nodes) → 6 GBps
├── ...
└── HA Pair 12 (2 nodes) → 6 GBps
────────────────────────────────
Total: 72 GBps to SINGLE NAMESPACE
(All Kafka brokers see same filesystem)
```

**S3 for High Throughput:**
```
Multiple EC2 Instances
├── EC2 Instance 1 → 12.5 GBps (100 Gbps) (to its prefixes)
├── EC2 Instance 2 → 12.5 GBps (100 Gbps) (to its prefixes)
├── EC2 Instance 3 → 12.5 GBps (100 Gbps) (to its prefixes)
└── ...
────────────────────────────────
Total: Unlimited aggregate
(Must partition data across prefixes)
```

**Critical Difference for Kafka:**
- **FSxN**: 10 Kafka brokers can collectively achieve 72 GBps reading/writing the **same segments**
- **S3**: Each Kafka broker gets up to 12.5 GBps, but must partition data to exceed 5,500 GET/sec per prefix
- **Reality**: Most Kafka clusters need 1-10 GBps total, making S3's single-client 12.5 GBps sufficient

### NetApp FSxN S3 Advantages

1. **Much lower latency** (<1ms cached, 1-5ms uncached vs 100-200ms) - **100-200x better than S3 Standard**
2. **Higher aggregate throughput to single namespace** (up to 72 GBps with 12 HA pairs, all brokers accessing same data)
3. **Dual-layer caching** (NVMe + memory) for hot data access
4. **Automatic data tiering** reduces storage costs
5. **Data efficiency** (deduplication, compression, compaction)
6. **Consistent, predictable performance** with provisioned resources
7. **Multi-protocol support** (can test with NFS/SMB for comparison)

### AWS S3 Standard Advantages

1. **Massively scalable** with no provisioning required
2. **Lower operational complexity** (fully serverless)
3. **Pay-per-use** model (no upfront capacity planning)
4. **Unlimited horizontal scaling** across prefixes
5. **Global availability** and cross-region replication
6. **Lower storage costs** ($23/TiB vs $125-140/TiB)
7. **Mature ecosystem** and tooling

### AWS S3 Express One Zone Advantages

1. **Low latency** (<10ms) - **10-20x better than S3 Standard**
2. **Very high request rates** (2M req/sec per bucket)
3. **Lower request costs** (80% cheaper than S3 Standard)
4. **Co-location with compute** in same AZ for best performance
5. **Simpler than FSxN** (no capacity provisioning)
6. **Still fully managed** (no infrastructure management)

## Performance Estimates for Kafka Workloads

### Typical Kafka Segment Access Pattern
- **Write**: Sequential, high-throughput (append-only)
- **Read**: Mixed sequential (follower replication) and random (consumer catch-up, time-based reads)
- **Metadata**: Frequent small reads/writes

### Expected Performance (Single Kafka Broker)

| **Workload** | **FSxN S3 (1 HA pair)** | **S3 Standard** | **S3 Express One Zone** |
|--------------|------------------------|----------------|------------------------|
| **Sequential Write** | 1-4 GBps | 1-1.25 GBps | 1-1.25 GBps |
| **Sequential Read** | 2-6 GBps | 1-1.25 GBps | 2-1.875 GBps |
| **Random Read (4KB)** | **5,000-20,000 IOPS**<br>Latency: 1-5ms | **100-500 IOPS**<br>Latency: 100-200ms | **10,000-50,000 IOPS**<br>Latency: 5-15ms |
| **Metadata Operations** | **Very fast** (<1ms) | Slow (100-200ms) | Fast (5-15ms) |
| **Producer Throughput** | **50-100 MB/s** per partition | **10-30 MB/s** per partition | **30-60 MB/s** per partition |
| **Consumer Lag Catch-up** | **Fast** (cached reads <1ms) | **Very slow** (random reads) | **Medium** (5-15ms) |

### Multi-Broker Scenarios

**FSxN (12 HA pairs):**
- 10 Kafka brokers × 6 GBps each = **60 GBps aggregate** (all accessing same segments)
- Good for: Very high throughput clusters (100k+ msgs/sec, large messages)
- Cost: **$140/TiB storage + ~$7,000-$15,000/month** in throughput capacity for 12 pairs

**S3 Standard:**
- 10 Kafka brokers × 1.25 GBps each = **12.5 GBps aggregate**
- Limited by: Request rate per prefix (5,500 GET/sec)
- Cost: **$23/TiB + request charges** (~$50-200/month for typical workload)

**S3 Express One Zone:**
- 10 Kafka brokers × 1.25 GBps each = **12.5 GBps aggregate**
- 2M req/sec per bucket (no prefix limits)
- Cost: **$110/TiB + request charges** (~$100-500/month for typical workload)

## Recommendation for Benchmarking

### Best Use Cases by Storage Type

**FSxN S3:**
- Workloads requiring **<5ms latency**
- **High random read performance** (consumer lag catch-up)
- Need **predictable performance** with SLAs
- Cost-effective at **sustained high throughput** (>1 GBps)
- Can leverage **deduplication/compression** (20-40% savings)

**S3 Standard:**
- **Cold/archive** Kafka tiers (rarely accessed)
- **Cost-sensitive** workloads with sequential access
- **Long-term retention** with infrequent reads
- **Simple operational model** preferred
- Expect **20-50% slower** producer/consumer throughput

**S3 Express One Zone:**
- **Best latency/cost balance** for Kafka
- **Active segments** with frequent access
- Need **high request rates** (>10k ops/sec)
- Co-locate with **brokers in same AZ**
- **80% cheaper** requests than S3 Standard

### Suggested Benchmark Configuration

1. **Test 1: Sequential Write Performance** (Producers)
   - Measure: MB/s per partition, P99 latency
   
2. **Test 2: Sequential Read Performance** (Follower replication)
   - Measure: MB/s, replication lag
   
3. **Test 3: Random Read Performance** (Consumer catch-up)
   - Measure: IOPS, P99 latency, time to catch up 1 hour lag
   
4. **Test 4: Mixed Workload** (Production-like)
   - 70% sequential write, 20% sequential read, 10% random read
   - Measure: Overall throughput, P95/P99 latency
   
5. **Test 5: Cost Analysis**
   - Calculate total monthly cost for 10TB, 50TB, 100TB scenarios
   - Include storage, throughput, requests, and data transfer

## References

- [AWS FSx for NetApp ONTAP Documentation](https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/what-is-fsx-ontap.html)
- [AWS FSx for ONTAP Performance](https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/performance.html)
- [AWS S3 Performance Optimization](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- [NetApp ONTAP S3 Supported Actions](https://docs.netapp.com/us-en/ontap/s3-config/ontap-s3-supported-actions-reference.html)
