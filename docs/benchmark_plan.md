# Benchmark Plan for Kafka Diskless (KIP-1150) - Inkless Implementation

## Overview

This benchmark plan outlines the approach to evaluate the performance of Kafka Diskless topics using Aiven's Inkless implementation. The benchmark will compare performance between AWS storage and NetApp FSxN storage.

## Baseline Scenario

The baseline scenario is based on the benchmark documented in [Aiven's blog post](https://aiven.io/blog/benchmarking-diskless-inkless-topics-part-1).

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
- Throughput target: 1 GiB/s total (333 MB/s per producer)
- Acks: all
- Linger.ms: 100
- Batch.size: 1048576 (1 MiB)
- Max.request.size: 4194304 (4 MiB)
- Compression.type: none

**Consumer Configuration:**
- Number of consumers: 6 (two per AZ)
- Consumer groups: 3 groups
- Throughput target: 3 GiB/s total (500 MB/s per consumer)
- Fetch.max.bytes: 67108864 (64 MiB)
- Fetch.max.wait.ms: 500
- Fetch.min.bytes: 4194304 (4 MiB)

**Cluster Setup:**
- Brokers: 6 brokers (m8g.4xlarge instances)
- Kafka version: 4.0 with Inkless implementation
- JVM heap: Not specified (using Aiven managed defaults)
- OS: Amazon Linux (Aiven managed)
- Instance type: m8g.4xlarge (16 vCPUs, 64 GiB memory)

**Storage Configuration (AWS Baseline):**
- Object storage: S3 (for Diskless topics)
  - Capacity: 3600 GiB (1 hour of data at 1 GiB/s ingress)
  - Region: Multi-AZ (3 availability zones)
  - Storage class: Standard
- Metadata store: Aiven PostgreSQL (i3.2xlarge, dual-AZ, 1.9TB NVMe)
- Network: Cross-AZ traffic for metadata operations
- Workload: 1 GiB/s in, 3 GiB/s out (fan-out pattern)

**Key Benchmark Parameters:**
- Test duration: 1 hour
- Producer clients: 144
- Consumer clients: 144
- Partitions per producer: ~4
- Total throughput: 1 GiB/s ingress, 3 GiB/s egress

## FSxN Configuration Variations

For NetApp FSxN testing, we will explore the following configuration variations based on AWS FSxN performance documentation. These configurations account for FSxN's performance characteristics including SSD IOPS, throughput capacity, and caching behavior.

### Configuration A: Baseline Equivalent FSxN
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge equivalent)
- **FSxN Configuration**:
  - Storage capacity: 3600 GiB SSD (matching S3 baseline - 1 hour of data at 1 GiB/s ingress)
  - Throughput capacity: 2048 MB/s (2 GiB/s - accounting for workload + FSxN overhead)
  - SSD IOPS: 16,000 (automatically provisioned with storage)
  - Protocol: NFSv4.1 with nconnect (4 connections)
  - Deployment type: MULTI_AZ_1 (first-generation Multi-AZ)
  - HA pair configuration: 1 HA pair across 2 Availability Zones
  - Failover behavior: Automatic failover to standby file server in different AZ
  - Mount options: `rsize=1048576,wsize=1048576,hard,noatime,nodiratime,nconnect=4`
  - Jumbo frames: Enabled (MTU 9001)
  - Cache configuration: Default (NVMe + in-memory caching)
  - Backup policy: Automatic daily backups across multiple AZs
- **Kafka Configuration**:
  - Matches exact baseline configuration from blog post
  - No compression (as per baseline)
  - Same producer/consumer settings
- **Expected**: Direct performance comparison to AWS S3 baseline
- **Rationale**: 
  - Storage capacity matches S3 baseline exactly (3600 GiB SSD)
  - Throughput capacity of 2 GiB/s (2048 MB/s) is calculated as:
    - **Base workload**: 1 GiB/s write + 3 GiB/s read = 4 GiB/s total I/O
    - **Caching benefit**: FSxN's NVMe and in-memory caching can serve ~50% of reads from cache
    - **Effective storage I/O**: 1 GiB/s write + 1.5 GiB/s read (50% cache hit) = 2.5 GiB/s
    - **Overhead buffer**: 20% additional capacity for metadata operations and burst handling
    - **Final calculation**: 2.5 GiB/s × 1.2 = 3 GiB/s theoretical need
    - **Practical provisioning**: 2 GiB/s chosen as conservative starting point that can be adjusted based on actual cache hit ratios
  - Uses FSxN best practices: nconnect, jumbo frames, optimal mount options
  - MULTI_AZ_1 provides synchronous replication across AZs for high availability
  - **Monitoring**: Cache hit ratio will be closely monitored to validate assumptions and adjust if needed

### Configuration B: Performance-Optimized FSxN
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge equivalent)
- **FSxN Configuration**:
  - Storage capacity: 4096 GiB SSD (increased for better caching)
  - Throughput capacity: 4096 MB/s (4 GiB/s - 2x Configuration A)
  - SSD IOPS: 32,000 (scaled with storage)
  - Protocol: NFSv4.1 with nconnect (8 connections for max throughput)
  - Deployment type: MULTI_AZ_2 (second-generation Multi-AZ)
  - HA pair configuration: 1 HA pair with enhanced performance
  - Failover behavior: Sub-second failover with synchronous replication
  - Mount options: `rsize=1048576,wsize=1048576,hard,noatime,nodiratime,nconnect=8`
  - Jumbo frames: Enabled (MTU 9001)
  - Cache configuration: Aggressive (larger NVMe cache allocation)
  - Backup policy: Hourly snapshots with 24-hour retention
- **Kafka Tuning**:
  - Increased `num.io.threads`: 16
  - Increased `num.network.threads`: 8
  - Larger socket buffers: 2MB
  - Increased `socket.send.buffer.bytes`: 2097152
  - Increased `socket.receive.buffer.bytes`: 2097152
- **Expected**: Lower latency, higher throughput at increased cost
- **Rationale**:
  - 4 GiB/s throughput capacity provides headroom for bursty workloads
  - 8 nconnect connections overcome EC2 network flow limits
  - Larger storage capacity improves cache hit ratios
  - MULTI_AZ_2 provides enhanced performance and faster failover

### Configuration C: Cost-Optimized FSxN
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 4 brokers (m8g.2xlarge equivalent - reduced compute)
- **FSxN Configuration**:
  - Storage capacity: 2048 GiB SSD (56% of Configuration A)
  - Throughput capacity: 512 MB/s (25% of Configuration A)
  - SSD IOPS: 8,000 (scaled with storage)
  - Protocol: NFSv4.1 with nconnect (2 connections)
  - Deployment type: SINGLE_AZ_1 (first-generation Single-AZ)
  - HA pair configuration: 1 HA pair within single AZ (different fault domains)
  - Failover behavior: Automatic failover within AZ (few seconds)
  - Mount options: `rsize=262144,wsize=262144,hard,noatime,nodiratime,nconnect=2`
  - Compression: Enabled at FSxN level (inline compression)
  - Jumbo frames: Disabled (standard MTU)
  - Backup policy: Daily backups to different AZ for data protection
- **Kafka Tuning**:
  - Producer compression: snappy (reduces storage I/O)
  - Reduced `log.flush.interval.messages`: 10000
  - Increased `log.flush.interval.ms`: 5000
  - Reduced batch sizes: 524288 (512 KiB) to match reduced throughput
- **Expected**: 40-50% cost reduction with moderate performance impact
- **Rationale**:
  - 512 MB/s throughput sufficient for steady-state workload
  - SINGLE_AZ_1 reduces HA costs while maintaining fault domain protection
  - Smaller storage capacity with compression enabled
  - Reduced nconnect connections (2 instead of 4)
  - Daily backups to different AZ provide data durability without Multi-AZ cost
- **Throughput Capacity Calculation**:
  - **Base workload**: 1 GiB/s ingress + 3 GiB/s egress = 4 GiB/s total I/O
  - **Compression benefit**: Producer compression (snappy) reduces storage I/O by ~60%
  - **Effective storage I/O**: (1 GiB/s × 0.4) + (3 GiB/s × 0.4) = 1.6 GiB/s compressed
  - **Caching benefit**: Conservative 20% cache hit ratio for reads: 1.6 GiB/s × 0.8 = 1.28 GiB/s
  - **Steady-state assumption**: Cost-optimized for average rather than peak workload
  - **Final calculation**: 1.28 GiB/s × 0.4 (cost optimization factor) = ~512 MB/s
  - **Validation approach**: Monitor compression ratios and cache performance, adjust if bottlenecks appear

### Configuration D: High Availability FSxN
- **Topic Configuration**: 576 partitions, 3x replication
- **Cluster**: 6 brokers (m8g.4xlarge equivalent)
- **FSxN Configuration**:
  - Storage capacity: 1.2 TB SSD
  - Throughput capacity: 768 MB/s (768 MB/s chosen for balanced HA performance)
  - SSD IOPS: 12,000 (scaled with storage)
  - Protocol: NFSv4.1 with nconnect (4 connections)
  - Deployment type: MULTI_AZ_1 (first-generation Multi-AZ for proven stability)
  - HA pair configuration: 1 HA pair with conservative failover settings
  - Failover behavior: Automatic failover with synchronous replication
  - Mount options: `rsize=1048576,wsize=1048576,hard,noatime,nodiratime,nconnect=4`
  - Jumbo frames: Enabled (MTU 9001)
  - Backup policy: Daily snapshots with 7-day retention + hourly snapshots for 24 hours
  - Monitoring: Enhanced monitoring for early failure detection
- **Kafka Tuning**:
  - Increased `unclean.leader.election.enable`: false
  - Higher `min.insync.replicas`: 2
  - Conservative producer settings: `acks=all`, `retries=5`
  - Increased `request.timeout.ms`: 30000 (for HA resilience)
  - Increased `delivery.timeout.ms`: 120000 (for HA resilience)
- **Expected**: Enterprise-grade availability with minimal performance overhead
- **Throughput Capacity Rationale**:
  - **Base requirement**: 1 GiB/s ingress + 3 GiB/s egress = 4 GiB/s total I/O
  - **HA overhead**: Additional 20-30% capacity for synchronous replication overhead
  - **Conservative caching**: Assume 30% cache hit ratio (lower than Config A for reliability)
  - **Calculation**: (1 GiB/s write + 2.1 GiB/s read) × 1.3 HA overhead = ~4.3 GiB/s theoretical
  - **Practical provisioning**: 768 MB/s represents a more conservative starting point that:
    - Prioritizes stability over maximum throughput
    - Accounts for additional HA-related network traffic
    - Provides headroom for failover operations
    - Can be increased if monitoring shows sufficient capacity
  - **Trade-off**: This configuration accepts potentially lower throughput in exchange for higher reliability and predictable performance under failure conditions

## FSxN-Specific Metrics

In addition to standard metrics, we will track FSxN-specific performance indicators:

### Storage Performance Metrics:
- **FSxN Throughput Utilization**: % of provisioned throughput used
- **IOPS**: Input/Output operations per second
- **Latency**: Average read/write latency (ms)
- **Cache Hit Ratio**: % of requests served from cache

### Cost Metrics:
- **Storage Cost**: $/GB/month for provisioned capacity
- **Throughput Cost**: $/MB/s for provisioned throughput
- **Operational Cost**: Management overhead and backup costs
- **Total Cost of Ownership**: Comprehensive cost comparison vs AWS baseline

### Reliability Metrics:
- **Availability**: Measured uptime percentage
- **Failover Time**: Time to recover from simulated failures
- **Data Durability**: Measured data integrity over time

## Benchmark Objectives

1. **Throughput**: Measure the maximum throughput achievable with diskless topics
2. **Latency**: Evaluate end-to-end latency for message production and consumption
3. **Resource Utilization**: Monitor CPU, memory, and network usage
4. **Scalability**: Test performance under varying load conditions

## Test Scenarios

### Scenario 1: AWS Storage Baseline
- **Storage**: AWS EBS or equivalent
- **Configuration**: Match the baseline setup from Aiven's blog
- **Metrics**: Throughput, latency, resource utilization

### Scenario 2: NetApp FSxN Storage
- **Storage**: NetApp FSxN
- **Configuration**: Same as Scenario 1 but with FSxN storage
- **Metrics**: Throughput, latency, resource utilization

## Metrics Collection

### Performance Metrics
- **Throughput**: Messages per second (MB/s)
- **Latency**: End-to-end latency (ms)
- **CPU Utilization**: Percentage usage across brokers
- **Memory Usage**: Memory consumption per broker
- **Network I/O**: Network throughput and latency

### Cost Analysis
- **Cost Estimation**: Storage and operational costs per GB/month

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

- Performance comparison between AWS and FSxN storage
- Identification of bottlenecks and optimization opportunities
- Recommendations for production deployment

## Next Steps

- Finalize test configurations
- Deploy test environments
- Execute benchmark tests
- Analyze and document results

## Future work

- Failure modes

## References

1. **Aiven Blog Post**: [Benchmarking Diskless Topics: Part 1](https://aiven.io/blog/benchmarking-diskless-inkless-topics-part-1)
   - Source of baseline configuration and performance expectations
   - Provides the 1 GiB/s in, 3 GiB/s out workload pattern
   - Details the 3600 GiB storage requirement

2. **AWS FSxN Performance Documentation**: [Amazon FSx for NetApp ONTAP performance](https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/performance.html#deployment-type-performance)
   - Performance characteristics and best practices
   - Throughput capacity and IOPS relationships
   - Mount options and networking recommendations
   - Caching behavior and optimization techniques

3. **AWS FSxN Multi-AZ Documentation**: [Availability, durability, and deployment options](https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/high-availability-AZ.html)
   - Multi-AZ vs Single-AZ deployment types
   - Failover processes and availability SLAs
   - Generation differences (SINGLE_AZ_1, SINGLE_AZ_2, MULTI_AZ_1, MULTI_AZ_2)
   - Network resource requirements for different deployment types

