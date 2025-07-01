# GC Parameter Tuning to Reduce Pauses

## Introduction

Garbage Collection pauses are one of the most critical performance bottlenecks in Java applications. Long GC pauses can lead to application timeouts, poor user experience, and SLA violations. This guide provides comprehensive strategies for tuning GC parameters to minimize pause times across different collectors.

## Understanding GC Pauses

### Types of GC Pauses

#### Stop-the-World (STW) Pauses
- **Definition**: All application threads are paused during collection
- **Impact**: Direct user-visible latency
- **Typical Duration**: 10ms to several seconds depending on heap size and collector

#### Concurrent Collection Overhead
- **Definition**: Background GC work that competes with application threads
- **Impact**: Reduced application throughput
- **Typical Impact**: 5-20% CPU overhead

### Key Metrics to Monitor

```bash
# Essential GC metrics
GC Pause Time (ms)          # Time application is frozen
GC Frequency (collections/sec) # How often GC occurs  
GC Throughput (%)           # % time spent in application vs GC
Allocation Rate (MB/s)      # Speed of object allocation
Promotion Rate (MB/s)       # Young to Old generation promotion
```

## Parallel GC Tuning

### Basic Configuration
```bash
# Enable Parallel GC (default in Java 8 and below)
-XX:+UseParallelGC

# Set number of GC threads (usually CPU cores)
-XX:ParallelGCThreads=8

# Target maximum pause time
-XX:MaxGCPauseMillis=200
```

### Advanced Tuning Parameters

#### Heap Size Optimization
```bash
# Initial and maximum heap size
-Xms4g -Xmx4g  # Equal values prevent heap expansion overhead

# Young generation sizing
-XX:NewRatio=2              # Old:Young = 2:1 (Young = 33% of heap)
-XX:NewSize=1g              # Initial young gen size
-XX:MaxNewSize=1g           # Maximum young gen size

# Survivor space sizing
-XX:SurvivorRatio=8         # Eden:Survivor = 8:1
-XX:TargetSurvivorRatio=90  # Target survivor occupancy
```

#### Pause Time Optimization
```bash
# Aggressive pause reduction
-XX:MaxGCPauseMillis=100    # Target 100ms max pauses
-XX:GCTimeRatio=19          # 5% max time in GC (1/(1+19))

# Young generation collection tuning
-XX:MaxTenuringThreshold=3  # Promote objects after 3 collections
-XX:InitialTenuringThreshold=1

# Old generation collection optimization
-XX:+UseParallelOldGC       # Parallel collection for old gen
```

#### Example Configuration for Low Latency
```bash
#!/bin/bash
# Parallel GC optimized for pause reduction
JAVA_OPTS="
-XX:+UseParallelGC
-XX:+UseParallelOldGC
-Xms8g -Xmx8g
-XX:NewRatio=1
-XX:SurvivorRatio=6
-XX:MaxGCPauseMillis=150
-XX:GCTimeRatio=19
-XX:MaxTenuringThreshold=2
-XX:+AlwaysPreTouch
-Xlog:gc*:gc-parallel.log:time
"
```

## G1GC Tuning

### Basic Configuration
```bash
# Enable G1GC (default since Java 9)
-XX:+UseG1GC

# Primary tuning parameter - target pause time
-XX:MaxGCPauseMillis=200
```

### Region and Generation Sizing
```bash
# Region size (power of 2, 1MB to 32MB)
-XX:G1HeapRegionSize=16m    # Optimal for most workloads

# Young generation sizing
-XX:G1NewSizePercent=20     # Min young gen = 20% of heap
-XX:G1MaxNewSizePercent=40  # Max young gen = 40% of heap

# Humongous objects threshold
# Objects > G1HeapRegionSize/2 are humongous
# Tune region size to avoid frequent humongous allocations
```

### Mixed Collection Tuning
```bash
# Mixed GC frequency and intensity
-XX:G1MixedGCCountTarget=8          # Spread mixed GC over 8 collections
-XX:G1MixedGCLiveThresholdPercent=85 # Only collect regions <85% live
-XX:G1HeapWastePercent=5            # Allow 5% heap waste
-XX:G1OldCSetRegionThreshold=10     # Max old regions per mixed GC
```

### Concurrent Marking Tuning
```bash
# Concurrent cycle triggers
-XX:G1ConcRefinementThreads=8       # Concurrent refinement threads
-XX:InitiatingHeapOccupancyPercent=45 # Start concurrent cycle at 45%

# SATB buffer tuning
-XX:G1ConcRSLogCacheSize=10         # Remember set log cache
```

#### Example High-Performance G1GC Configuration
```bash
#!/bin/bash
# G1GC optimized for low pause and high throughput
JAVA_OPTS="
-XX:+UseG1GC
-Xms16g -Xmx16g
-XX:MaxGCPauseMillis=100
-XX:G1HeapRegionSize=16m
-XX:G1NewSizePercent=30
-XX:G1MaxNewSizePercent=50
-XX:G1MixedGCCountTarget=12
-XX:G1MixedGCLiveThresholdPercent=90
-XX:InitiatingHeapOccupancyPercent=35
-XX:G1HeapWastePercent=3
-XX:+G1UseAdaptiveIHOP
-Xlog:gc*:gc-g1.log:time
"
```

## ZGC Tuning

### Basic Configuration
```bash
# Enable ZGC (production ready since Java 17)
-XX:+UseZGC

# Soft maximum heap size (recommended)
-XX:SoftMaxHeapSize=30g  # With -Xmx32g
```

### Advanced ZGC Parameters
```bash
# Heap management
-XX:SoftMaxHeapSize=28g     # Keep 4g headroom with 32g max heap
-XX:ZUncommitDelay=300      # Delay before uncommitting memory (seconds)

# Collection triggers
-XX:ZCollectionInterval=5   # Force collection every 5 seconds (if needed)

# Generational ZGC (Java 21+, experimental)
-XX:+UnlockExperimentalVMOptions
-XX:+UseZGenerationalGC
```

#### Example ZGC Configuration for Ultra-Low Latency
```bash
#!/bin/bash
# ZGC optimized for sub-10ms pauses
JAVA_OPTS="
-XX:+UseZGC
-Xms64g -Xmx64g
-XX:SoftMaxHeapSize=60g
-XX:+UnlockExperimentalVMOptions
-XX:+UseTransparentHugePages
-Xlog:gc*:gc-zgc.log:time
"
```

## Workload-Specific Tuning

### High Allocation Rate Applications
```bash
# Increase young generation size
-XX:NewRatio=1              # For Parallel GC
-XX:G1NewSizePercent=40     # For G1GC

# Increase parallel GC threads
-XX:ParallelGCThreads=16    # More threads for faster collection

# Tune survivor spaces
-XX:SurvivorRatio=4         # Larger survivor spaces
-XX:MaxTenuringThreshold=15 # Allow more aging in young gen
```

### Large Heap Applications (>32GB)
```bash
# Use ZGC for very large heaps
-XX:+UseZGC
-XX:SoftMaxHeapSize=120g

# Or use G1GC with larger regions
-XX:+UseG1GC
-XX:G1HeapRegionSize=32m
-XX:MaxGCPauseMillis=50
```

### Low Latency Trading Systems
```bash
# ZGC with minimal pause configuration
-XX:+UseZGC
-XX:SoftMaxHeapSize=28g
-XX:+AlwaysPreTouch         # Pre-touch memory pages
-XX:+UseLargePages          # Use large memory pages
-XX:LargePageSizeInBytes=2m

# JIT compiler optimizations
-XX:+UseG1GC                # Alternative if ZGC not available
-XX:MaxGCPauseMillis=10
-XX:+AggressiveOpts
```

## Memory Layout Optimization

### Large Pages Configuration
```bash
# Enable large pages (requires OS configuration)
-XX:+UseLargePages
-XX:LargePageSizeInBytes=2m

# Pre-touch memory at startup
-XX:+AlwaysPreTouch

# OS configuration (Linux)
# echo 1024 > /proc/sys/vm/nr_hugepages
# echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### NUMA Awareness
```bash
# Enable NUMA awareness
-XX:+UseNUMA

# Parallel GC NUMA optimization
-XX:+UseParallelGC
-XX:+UseNUMA

# Check NUMA topology
# numactl --hardware
```

## Advanced Tuning Techniques

### String Deduplication (G1GC only)
```bash
# Enable string deduplication to reduce memory pressure
-XX:+UseG1GC
-XX:+UseStringDeduplication
-XX:StringDeduplicationAgeThreshold=3
```

### Compressed OOPs Optimization
```bash
# Optimize object pointers (automatic under 32GB heap)
-XX:+UseCompressedOops
-XX:+UseCompressedClassPointers

# Monitor compressed OOP usage
-XX:+PrintCompressedOopsMode
```

### GC Ergonomics Tuning
```bash
# Disable adaptive sizing for predictable behavior
-XX:-UseAdaptiveSizePolicy   # Parallel GC
-XX:-G1UseAdaptiveIHOP      # G1GC

# Or tune adaptive behavior
-XX:AdaptiveSizePolicyWeight=10
-XX:GCTimeRatio=99          # Allow only 1% time in GC
```

## Monitoring and Analysis

### Essential GC Logging
```bash
# Java 9+ unified logging
-Xlog:gc*:gc.log:time,level,tags

# Additional useful logs
-Xlog:gc+heap=debug:gc-heap.log:time
-Xlog:safepoint:gc-safepoint.log:time

# Java 8 logging
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+PrintGCApplicationStoppedTime
-Xloggc:gc.log
```

### GC Analysis Tools

#### GCViewer Analysis
```bash
# Download and analyze GC logs
wget https://github.com/chewiebug/GCViewer/releases/download/1.36/gcviewer-1.36.jar
java -jar gcviewer-1.36.jar gc.log

# Key metrics to examine:
# - Max pause time
# - Average pause time  
# - Pause time distribution
# - GC frequency
# - Memory usage patterns
```

#### JVM Flags for Analysis
```bash
# Application pause time analysis
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintSafepointStatistics

# Memory allocation tracking
-XX:+PrintTenuringDistribution
-XX:+PrintGCApplicationConcurrentTime
```

### Performance Regression Detection
```java
// JMX monitoring example
public class GCMonitor {
    
    public void monitorGCMetrics() {
        List<GarbageCollectorMXBean> gcBeans = 
            ManagementFactory.getGarbageCollectorMXBeans();
            
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            long collections = gcBean.getCollectionCount();
            long time = gcBean.getCollectionTime();
            
            if (collections > 0) {
                double avgTime = (double) time / collections;
                System.out.printf("%s: %d collections, avg=%.2fms%n", 
                    gcBean.getName(), collections, avgTime);
                    
                // Alert if average pause > threshold
                if (avgTime > 100) {
                    alertHighGCPause(gcBean.getName(), avgTime);
                }
            }
        }
    }
    
    private void alertHighGCPause(String collectorName, double avgTime) {
        // Send alert to monitoring system
        logger.warn("High GC pause detected: {} avg={}ms", 
                   collectorName, avgTime);
    }
}
```

## Troubleshooting Common Issues

### Long Young Generation Pauses
```bash
# Problem: Young GC taking too long
# Analysis
-Xlog:gc+heap=debug:gc-heap.log:time

# Solutions
-XX:NewRatio=3              # Reduce young gen size
-XX:MaxTenuringThreshold=2  # Promote objects faster  
-XX:ParallelGCThreads=16    # More GC threads
```

### Frequent Full GC Events
```bash
# Problem: Old generation filling up too quickly
# Analysis
-Xlog:gc*:gc.log:time
-XX:+PrintTenuringDistribution

# Solutions
-Xmx16g                     # Increase heap size
-XX:NewRatio=1              # Larger young generation
-XX:MaxTenuringThreshold=15 # Keep objects in young gen longer
```

### G1GC Mixed Collection Issues
```bash
# Problem: Long mixed GC pauses
# Analysis
-Xlog:gc+regions=debug:gc-regions.log:time

# Solutions
-XX:G1MixedGCCountTarget=16    # More mixed collections
-XX:G1OldCSetRegionThreshold=5 # Fewer regions per collection
-XX:G1MixedGCLiveThresholdPercent=95 # Only very empty regions
```

### ZGC Allocation Spikes
```bash
# Problem: Allocation rate exceeding collection rate
# Analysis
-Xlog:gc*:gc.log:time
-Xlog:gc+heap=info:gc-heap.log:time

# Solutions
-XX:SoftMaxHeapSize=56g     # More headroom (with 64g max)
-XX:ZCollectionInterval=1   # More frequent collections
```

## Production Deployment Strategy

### Gradual Rollout Process
1. **Baseline Measurement**: Collect current GC metrics
2. **Staging Testing**: Test new parameters under production load
3. **Canary Deployment**: Apply to small percentage of instances
4. **Monitor and Validate**: Confirm improvement in metrics
5. **Full Rollout**: Apply to all instances

### Rollback Plan
```bash
#!/bin/bash
# Keep original configuration for quick rollback
ORIGINAL_GC_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=200"
NEW_GC_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:G1NewSizePercent=30"

# Use feature flags or environment variables for easy switching
GC_OPTS=${USE_NEW_GC_CONFIG:-$ORIGINAL_GC_OPTS}
```

### Monitoring Checklist
- [ ] Average pause time decreased
- [ ] 99.9th percentile pause time within SLA
- [ ] No increase in Full GC frequency
- [ ] Application throughput maintained
- [ ] Memory usage patterns stable
- [ ] No allocation rate spikes

## Conclusion

Effective GC tuning requires understanding your application's allocation patterns, latency requirements, and throughput needs. Start with conservative changes, measure the impact, and iterate gradually. Remember that the best GC configuration is highly dependent on your specific workload and hardware environment.

Key principles for successful GC tuning:
- Measure before and after changes
- Change one parameter at a time
- Test under realistic load
- Monitor long-term trends, not just short-term improvements
- Have a rollback plan ready
