# HikariCP Optimal Pool Sizing Based on Load: Load-Adapted Sizing Guide

## Introduction

Optimal HikariCP connection pool sizing is critical for application performance and resource efficiency. 
This guide provides practical methodologies for determining the ideal pool size based on actual load patterns, without over-provisioning resources or causing connection starvation.

## HikariCP Pool Configuration Parameters

### Key Configuration Settings
- **maximumPoolSize**: Maximum connections in the pool (default: 10)
- **minimumIdle**: Minimum idle connections maintained (default: same as maximumPoolSize)
- **connectionTimeout**: Max milliseconds to wait for connection (default: 30000)
- **idleTimeout**: Max idle time before connection removal (default: 600000)
- **maxLifetime**: Max connection lifetime in milliseconds (default: 1800000)
- **leakDetectionThreshold**: Connection leak detection timeout (default: 0)

### Configuration Impact on Pool Sizing
- **maximumPoolSize**: Primary sizing parameter - directly affects concurrent capacity
- **minimumIdle**: Ensures immediate connection availability, reduces acquisition latency
- **connectionTimeout**: Prevents thread blocking when pool is exhausted
- **idleTimeout + maxLifetime**: Controls connection lifecycle, affects pool dynamics

## Understanding Load Patterns

### Traffic Pattern Types

**Steady-State Applications**: Consistent, predictable load
- Examples: Internal business apps, data processing systems
- Strategy: Static sizing with 20-30% buffer

**Bursty Applications**: Sudden traffic spikes
- Examples: E-commerce sites, news platforms
- Strategy: Adaptive sizing with rapid scaling

**Cyclical Applications**: Predictable usage cycles
- Examples: Reporting systems, batch processors
- Strategy: Time-based adaptive sizing

## Core Sizing Formulas

### Little's Law Application
```
Pool Size = Request Rate × Average Database Operation Time
```
Example: 1000 req/sec × 0.05 sec = 50 connections

### CPU-Core Based Formula
```
Pool Size = (CPU Cores × 2) + 1
```
Conservative approach balancing processing capacity with I/O

### Utilization-Based Formula
```
Pool Size = Peak Concurrent Load / Target Utilization (70-80%)
```

## Load Analysis Process

### 1. Measure Current Performance
- Monitor peak concurrent database operations
- Track connection acquisition times
- Identify queue depths during high load
- Measure average connection hold times

### 2. Identify Load Characteristics
- **Peak Hours**: When does maximum load occur?
- **Traffic Growth**: What's the growth trend?
- **Spike Patterns**: How sudden are load increases?
- **Geographic Distribution**: Multiple time zones?

### 3. Calculate Optimal Size
- Use formulas as starting points
- Apply load testing to validate
- Consider safety margins (20-30%)
- Account for future growth (6-12 months)

## Performance Targets

### Healthy Metrics
- Connection wait time: < 10ms
- Pool utilization: 60-80% at peak
- Queue depth: 0-2 threads waiting
- Connection success rate: > 99.9%

### Warning Signs
- Wait times: 10-50ms
- Utilization: > 85% consistently
- Queue depth: > 5 threads
- Timeout errors increasing

## Sizing Decision Framework

### Small Pool Size (5-15 connections)
**When to use**: Low-traffic applications, microservices, development environments
**Benefits**: Minimal resource usage, fast startup
**Risks**: Easy to exhaust under load spikes

### Medium Pool Size (15-50 connections)
**When to use**: Standard web applications, moderate concurrent users
**Benefits**: Good balance of performance and resources
**Risks**: May not handle viral growth

### Large Pool Size (50+ connections)
**When to use**: High-traffic applications, batch processing, enterprise systems
**Benefits**: Handles large concurrent loads
**Risks**: Resource waste, database server pressure

## Optimization Strategies

### Dynamic Sizing Approach
1. Start with formula-based estimate
2. Monitor key metrics continuously
3. Adjust based on actual performance
4. Plan for seasonal variations
5. Test under realistic load conditions

### Load Testing Validation
- Gradually increase load while monitoring metrics
- Identify breaking points and optimal ranges
- Validate performance under spike conditions
- Test pool recovery after overload scenarios

## Common Mistakes to Avoid

- **Over-sizing**: Wasting database connections and memory
- **Under-sizing**: Causing connection starvation and timeouts
- **Static sizing**: Not adapting to changing load patterns
- **Ignoring database limits**: Exceeding database max connections
- **No growth planning**: Not considering future capacity needs

## Best Practices

1. **Start Conservative**: Begin with smaller pools and grow based on metrics
2. **Monitor Continuously**: Track utilization and performance trends
3. **Plan for Growth**: Size for 6-12 months future capacity
4. **Load Test Regularly**: Validate sizing under realistic conditions
5. **Consider Database Limits**: Ensure total pools don't exceed database capacity
6. **Document Decisions**: Record sizing rationale for future reference

## Conclusion

Optimal HikariCP pool sizing requires balancing performance needs with resource efficiency. 
Use mathematical formulas as starting points, validate with load testing, and continuously monitor to adapt to changing patterns. 
The goal is maintaining sub-10ms connection acquisition times while avoiding resource waste, ensuring your application scales efficiently with actual usage patterns.
