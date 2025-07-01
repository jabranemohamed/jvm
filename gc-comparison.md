# G1GC vs ZGC vs Parallel GC Comparison

## Introduction

Choosing the right Garbage Collector (GC) is crucial for Java application performance. Each collector has its own characteristics, advantages, and optimal use cases. This guide provides a detailed comparison of the three most commonly used collectors in production.

## Collectors Overview

### Parallel GC (ParallelGC)
- **Type**: Generational stop-the-world collector
- **Availability**: Since Java 1.4, default until Java 8
- **Philosophy**: Maximize throughput at the expense of latency

### G1GC (Garbage First)
- **Type**: Low-latency generational collector
- **Availability**: Since Java 7, default since Java 9
- **Philosophy**: Balance between throughput and latency with predictable pauses

### ZGC (Z Garbage Collector)
- **Type**: Concurrent ultra-low latency collector
- **Availability**: Since Java 11 (experimental), production ready since Java 17
- **Philosophy**: Ultra-low latency independent of heap size

## Detailed Comparison

### 1. Architecture and Operation

#### Parallel GC
```
┌─────────────────────────────────────┐
│           Heap Memory               │
├─────────────────┬───────────────────┤
│   Young Gen     │    Old Gen        │
├─────┬───────────┼───────────────────┤
│Eden │ Survivor  │   Tenured Space   │
└─────┴───────────┴───────────────────┘
```

**Characteristics:**
- Classic Young/Old Generation division
- Parallel collection using all available threads
- Complete stop-the-world during collection
- Mark-sweep-compact algorithm for Old Generation

#### G1GC
```
┌─────────────────────────────────────┐
│        Heap divided into regions    │
├───┬───┬───┬───┬───┬───┬───┬───┬───┤
│ E │ S │ O │ E │ S │ O │ H │ E │ O │
└───┴───┴───┴───┴───┴───┴───┴───┴───┘
E = Eden, S = Survivor, O = Old, H = Humongous
```

**Characteristics:**
- Heap divided into fixed-size regions (1MB to 32MB)
- Incremental collection with pause-time goals
- Concurrent marking for Old Generation
- Evacuation pause for compaction

#### ZGC
```
┌─────────────────────────────────────┐
│     Heap with Colored Pointers      │
├─────────────────────────────────────┤
│  Objects + Metadata in pointers    │
│     Concurrent collection           │
└─────────────────────────────────────┘
```

**Characteristics:**
- Non-generational by default (generational available since Java 21)
- Colored pointers for object tracking
- Fully concurrent collection
- Load barriers for consistency

### 2. Performance and Latency

| Metric | Parallel GC | G1GC | ZGC |
|--------|-------------|------|-----|
| **Throughput** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Average Latency** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Max Pauses** | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Predictability** | ⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Heap Scalability** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### 3. Resource Consumption

#### CPU Overhead
- **Parallel GC**: Low during execution, high spikes during GC
- **G1GC**: Moderate and constant, concurrent work
- **ZGC**: Higher baseline, but very stable

#### Memory Overhead
- **Parallel GC**: ~2-5% of heap
- **G1GC**: ~5-10% of heap (remembered sets, card tables)
- **ZGC**: ~10-20% of heap (colored pointers, metadata)

## Configuration and Tuning

### Parallel GC
```bash
# Basic configuration
-XX:+UseParallelGC

# Advanced tuning
-XX:MaxGCPauseMillis=200
-XX:GCTimeRatio=19
-XX:NewRatio=2
-XX:SurvivorRatio=8

# Monitoring (Java 9+)
-Xlog:gc*:gc.log:time
# Or for Java 8
-XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps
```

### G1GC
```bash
# Basic configuration
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# Advanced tuning
-XX:G1HeapRegionSize=16m
-XX:G1NewSizePercent=20
-XX:G1MaxNewSizePercent=40
-XX:G1MixedGCCountTarget=8
-XX:G1MixedGCLiveThresholdPercent=85

# Monitoring (Java 9+)
-Xlog:gc*:gc.log:time
# Or for Java 8
-XX:+PrintGC -XX:+PrintGCDetails
```

### ZGC
```bash
# Basic configuration
-XX:+UseZGC
# Note: -XX:+UnlockExperimentalVMOptions no longer needed since Java 17

# Advanced tuning
-XX:SoftMaxHeapSize=30g
-Xlog:gc*:gc.log:time

# Java 21+ - ZGC Generational (experimental)
-XX:+UseZGC -XX:+UnlockExperimentalVMOptions -XX:+UseZGenerationalGC
```

## Recommended Use Cases

### Use Parallel GC when:
✅ **Throughput is prioritized over latency**
- Batch applications
- Massive data processing
- Scientific computing
- ETL processes

✅ **Moderate heap size (< 4GB)**

✅ **Acceptable GC pauses (> 100ms)**

### Use G1GC when:
✅ **Balance between throughput/latency required**
- Web applications with moderate SLAs
- Business services with latency constraints
- Applications with 4GB-32GB heap

✅ **Predictable pauses important**
- SLAs with latency constraints (< 100ms)
- Interactive applications

✅ **Large heap with mixed allocation patterns**

### Use ZGC when:
✅ **Ultra-low latency critical**
- Trading systems
- Real-time applications
- Gaming servers
- High-frequency applications

✅ **Very large heap (> 32GB)**
- Big Data applications
- Massive in-memory caches
- Real-time analytics

✅ **Pauses < 10ms indispensable**

## Performance Metrics

### Key metrics to monitor

```java
// Example JVM monitoring
public class GCMetrics {
    
    @EventListener
    public void handleGCEvent(GarbageCollectionNotificationInfo info) {
        String gcName = info.getGcName();
        long duration = info.getGcInfo().getDuration();
        long beforeUsed = info.getGcInfo().getMemoryUsageBeforeGc()
            .values().stream().mapToLong(MemoryUsage::getUsed).sum();
        long afterUsed = info.getGcInfo().getMemoryUsageAfterGc()
            .values().stream().mapToLong(MemoryUsage::getUsed).sum();
        
        logger.info("GC {} - Duration: {}ms, Freed: {}MB", 
                   gcName, duration, (beforeUsed - afterUsed) / 1024 / 1024);
    }
}
```

### Recommended alert thresholds

| Collector | Max Pause | Frequency | Throughput |
|-----------|-----------|-----------|------------|
| **Parallel GC** | < 1000ms | < 5% of time | > 95% |
| **G1GC** | < 200ms | < 5% of time | > 90% |
| **ZGC** | < 10ms | Always | > 85% |

## Migration Between Collectors

### From Parallel GC to G1GC
```bash
# Before
-XX:+UseParallelGC -Xmx8g

# After - Progressive migration
-XX:+UseG1GC -Xmx8g -XX:MaxGCPauseMillis=200

# Monitor for 1-2 weeks
# Adjust if necessary
-XX:G1HeapRegionSize=16m
```

### From G1GC to ZGC
```bash
# Before
-XX:+UseG1GC -Xmx32g -XX:MaxGCPauseMillis=100

# After - Test in staging first
-XX:+UseZGC -Xmx32g
-XX:SoftMaxHeapSize=30g

# Intensive monitoring required
```

## Common Troubleshooting

### Parallel GC - Long pauses
```bash
# Diagnostics (Java 9+)
-Xlog:safepoint:gc-safepoint.log:time
-Xlog:gc*:gc.log:time

# Solutions
-XX:MaxGCPauseMillis=100  # Reduce target
-XX:NewRatio=1           # More young gen
```

### G1GC - Mixed GC issues
```bash
# Diagnostics
-Xlog:gc*:gc.log:time
-Xlog:gc+heap=debug:gc-heap.log:time

# Solutions
-XX:G1MixedGCCountTarget=16     # More mixed GC
-XX:G1OldCSetRegionThreshold=5  # Fewer regions per cycle
```

### ZGC - High allocation rate
```bash
# Diagnostics
-Xlog:gc*:gc.log:time
-Xlog:gc+heap=debug:gc-heap.log:time

# Solutions
-XX:SoftMaxHeapSize=28g  # More headroom
# Note: ZCollectionInterval and ZUncommitDelay have been removed
```

## Monitoring Tools

### GCViewer
```bash
# Export logs (Java 9+)
-Xlog:gc*:gc.log:time

# For Java 8
-Xloggc:gc.log -XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=10M

# Analysis with GCViewer
java -jar gcviewer.jar gc.log
```

### JProfiler / VisualVM
- Real-time monitoring
- Trend analysis
- Correlation with application metrics

### Prometheus + Grafana
```yaml
# JVM metrics
jvm_gc_pause_seconds
jvm_gc_memory_allocated_bytes_total
jvm_gc_memory_promoted_bytes_total
```

## Conclusion

GC choice heavily depends on application context:

- **Parallel GC** remains the best choice for throughput-oriented applications
- **G1GC** offers the best compromise for most applications
- **ZGC** is essential for latency-critical applications

Migration should always be thoroughly tested with real production workloads before deployment.
