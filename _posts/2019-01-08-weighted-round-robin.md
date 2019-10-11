---
layout: post
title: "加权轮询算法"
date: "2019-01-08"
description: "加权轮询算法"
tag: [algorithm]
---

### 加权轮询
加权轮询调度设计用于更好的处理不同处理能力的服务器; 每个服务器可以分配一个权重, 用一个整数值表示处理能力; 高权重的服务器优于低权重的服务器接受新的连接, 高权重的服务器比低权重的服务器会获得更多的连接, 权重相同的服务器会获得相同的连接; 加权轮询调度的伪代码如下
```
# 假定这里有一个服务器集合 S = {S0, S1, …, Sn-1}; W(Si) 表示 Si 的权重
# i 表示上一次选择的服务器, i 的初始值为 -1
# cw 是调度的当前权重, 并且 cw 的初始值为 0
# max(S) 是所有服务器的最大权重
# gcd(S) 是所有服务器权重的最大公约数
while (true) {
    i = (i + 1) mod n;  // 遍历 S
    if (i == 0) {  // 遍历 S 一次, 则当前权重减去最大公约数
        cw = cw - gcd(S);
        if (cw <= 0) {  // 获取最大权重
            cw = max(S);
            if (cw == 0)
            return NULL;
        }
    }
    if (W(Si) >= cw)
        return Si;
```
例如, 真实服务器 A, B, C 的权重是 4,, 3, 2, 在一个调度周期内 (mod (sum(Wi) / gcd(S))) 调度队列将会是 AABABCABC  
优化的加权轮询调度实现在 [IPVS](http://kb.linuxvirtualserver.org/wiki/IPVS) 的规则修改后, 调度队列会根据服务器的权重生成; 网络连接使用轮询方式的调度队列顺序连接到不同真实服务器  
当真实服务器的处理能力不同时, 加权轮询调度要优于 [轮询调度](http://kb.linuxvirtualserver.org/wiki/Round-Robin_Scheduling); 但是, 如果请求的负载非常高也可能导致真实服务器之间的动态负载不均衡; 短时间内, 获取响应的大量请求可能连接同一台真实服务器  
事实上, [轮询调度](http://kb.linuxvirtualserver.org/wiki/Round-Robin_Scheduling) 是加权轮询调度的一个特例, 即所有服务器的权重都相等

### Java 实现
```
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

/**
 * 加权轮询调度算法
 */
public class WeightRoundRobin {
    public static class Server {
        private String address;
        private int weight;

        public Server(String address, int weight) {
            this.address = address;
            this.weight = weight;
        }

        @Override
        public String toString() {
            return "Server{address='" + address + ", weight=" + weight + '}';
        }
    }

    private List<Server> servers;
    private int currentIndex = -1;
    private int currentWeight = 0;
    private int maximumWeight;
    private int greatestCommonDivisor;
    private int serverCount;

    public WeightRoundRobin(List<Server> servers) {
        this.servers = servers;
        List<Integer> list = servers.stream().map(server -> server.weight).collect(Collectors.toList());
        maximumWeight = computeMaximum(list);
        greatestCommonDivisor = computeGreatestCommonDivisor(list);
        serverCount = list.size();
    }

    private int computeMaximum(List<Integer> list) {
        if (list == null && list.isEmpty()) {
            throw new IllegalArgumentException("List must not null or empty.");
        }

        int max = list.get(0);
        for (int i = 1; i < list.size(); i++) {
            max = Math.max(max, list.get(i));
        }
        return max;
    }

    private int computeGreatestCommonDivisor(int a, int b) {
        return a % b == 0 ? b : computeGreatestCommonDivisor(b, a % b);
    }

    private int computeGreatestCommonDivisor(List<Integer> list) {
        if (list == null && list.isEmpty()) {
            throw new IllegalArgumentException("List must not null or empty.");
        }

        int gcd = list.get(0);
        for (int i = 1; i < list.size(); i++) {
            gcd = computeGreatestCommonDivisor(gcd, list.get(i));
        }
        return gcd;
    }

    public Server next() {
        while (true) {
            currentIndex = (currentIndex + 1) % serverCount;
            if (currentIndex == 0) {
                currentWeight = currentWeight - greatestCommonDivisor;
                if (currentWeight <= 0 ) {
                    currentWeight = maximumWeight;
                    if (currentWeight == 0) {
                        return null;
                    }
                }
            }
            Server server = servers.get(currentIndex);
            if (server.weight >= currentWeight) {
                return server;
            }
        }
    }

    public static void main(String[] args) {
        List<Server> servers = new ArrayList<>();
        servers.add(new Server("192.168.0.1", 1));
        servers.add(new Server("192.168.0.2", 2));
        servers.add(new Server("192.168.0.3", 4));
        servers.add(new Server("192.168.0.4", 8));
        // weight round robin
        WeightRoundRobin wrr = new WeightRoundRobin(servers);
        for (int i = 0; i < 100; i++) {
            System.out.println(wrr.next());
        }
    }
}
```

>**参考:**
[Weighted Round-Robin Scheduling](http://kb.linuxvirtualserver.org/wiki/Weighted_Round-Robin_Scheduling)
