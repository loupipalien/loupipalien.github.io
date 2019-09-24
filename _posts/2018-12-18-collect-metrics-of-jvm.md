---
layout: post
title: "采集 JVM 的 Metrics"
date: "2018-12-18"
description: "采集 JVM 的 Metrics"
tag: [java, metrics]
---

### metrics-jvm
这个模块包含了一系列可重用的 guage 和 metric sets, 允许方便的获取 JVM 的指标; 支持的 metric 包括
- 对所有执行的垃圾回收期统计次数和耗时
- 对所有内存池统计使用, 包括非堆内存
- 线程状态分解, 包括死锁
- Buffer 池大小和使用

#### 依赖
```
<dependencies>
    <dependency>
        <groupId>io.dropwizard.metrics</groupId>
        <artifactId>metrics-core</artifactId>
        <version>${metrics.version}</version>
    </dependency>
    <dependency>
        <groupId>io.dropwizard.metrics</groupId>
        <artifactId>metrics-jvm</artifactId>
        <version>${metrics.version}</version>
    </dependency>
</dependencies>
```

#### 使用
```
private void buildJvmMetrics(MetricRegistry registry) {
    registerAll("jvm.buffer", new BufferPoolMetricSet(ManagementFactory.getPlatformMBeanServer()), registry);
    registerAll("jvm.class", new ClassLoadingGaugeSet(), registry);
    register("jvm.uptime", (Gauge) () -> ManagementFactory.getRuntimeMXBean().getUptime(), registry);
    register("jvm.time", (Gauge<Long>) () -> new CpuTimeClock().getTime(), registry);
    register("jvm.fd", new FileDescriptorRatioGauge(), registry);
    registerAll("jvm.gc", new GarbageCollectorMetricSet(), registry);
    registerAll("jvm.attribute", new JvmAttributeGaugeSet(), registry);
    registerAll("jvm.memory", new MemoryUsageGaugeSet(), registry);
    registerAll("jvm.thread", new ThreadStatesGaugeSet(ManagementFactory.getThreadMXBean(), new ThreadDeadlockDetector()), registry);
}

private static void registerAll(String prefix, MetricSet metricSet, MetricRegistry registry) {
    for (Map.Entry<String, Metric> entry : metricSet.getMetrics().entrySet()) {
        if (entry.getValue() instanceof MetricSet) {
            registerAll(prefix + "." + entry.getKey(), (MetricSet) entry.getValue(), registry);
        } else {
            register(prefix + "." + entry.getKey(), entry.getValue(), registry);
        }
    }
}

private static void register(String name, Metric metric, MetricRegistry registry) {
    registry.register(name, metric);
}
```

>**参考:**
[JVM Instrumentation](https://metrics.dropwizard.io/4.0.0/manual/jvm.html)  
[Getting metrics from the Java Virtual Machine](https://jansipke.nl/getting-metrics-from-the-java-virtual-machine-jvm/)
