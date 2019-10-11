---
layout: post
title: "Metrics 快速入门"
date: "2018-12-16"
description: "Metrics 快速入门"
tag: [java, metrics]
---

### Metrics 介绍
Metrics 可以为已存在的项目添加度量指标, 简化指标统计重复代码, 并且支持各种 Reporter 将指标打到展示组件中

#### 依赖
```
<dependencies>
    <dependency>
        <groupId>io.dropwizard.metrics</groupId>
        <artifactId>metrics-core</artifactId>
        <version>${metrics.version}</version>
    </dependency>
</dependencies>
```

#### 简单示例
以下代码是统计方法每秒调用数的示例, 并上报到控制台
```
public class MeterDemo {
    private static final MetricRegistry metrics = new MetricRegistry();
    // A meter measures the rate of events over time (e.g., “requests per second”)
    private final Meter requests = metrics.meter(MetricRegistry.name(MeterDemo.class, "method", "requests"));
    static {
        ConsoleReporter reporter = ConsoleReporter.forRegistry(metrics)
                .convertRatesTo(TimeUnit.SECONDS)
                .convertDurationsTo(TimeUnit.MILLISECONDS)
                .build();
        reporter.start(1, TimeUnit.SECONDS);
    }

    public void method() {
        requests.mark();
    }

    public static void main(String[] args) throws InterruptedException {
        MeterDemo demo = new MeterDemo();
        for (int i = 0; i < 100; i++) {
            demo.method();
            TimeUnit.MILLISECONDS.sleep(300);
        }
    }
}
```
更多示例见 [Metrics 官网](https://metrics.dropwizard.io)

### 其他
如果展示组件支持统计功能, 例如 grafana; 在数据量较小的情况下, 可以考虑在应用中只上报事件, 使用展示组件统计; 如果数据量加大还是推荐使用 Metrics 的组件在应用中统计后再上报

>**参考:**
[Getting Started](https://metrics.dropwizard.io/4.0.0/getting-started.html)
