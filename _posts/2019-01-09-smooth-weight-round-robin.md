---
layout: post
title: "平滑的基于权重的轮询算法"
date: "2019-01-09"
description: "平滑的基于权重的轮询算法"
tag: [algorithm]
---

### 平滑的加权轮询
所谓平滑, 即在一段时间内, 不仅服务器被选择的次数分布和权重一致, 而且调度算法还比较均匀的选择服务器, 而不会几种一段时间之内值选择某一个权重较高的服务器; 如果使用基于权重的轮询算法可能会造成某个服务几种被调用而压力过大  
例如: `{a:5, b:1, c:1}` 的一组服务器, 平滑的基于权重的轮询算法的调度序列为 `{a, a, b, a, c, a, a}`, 这明显比基于权重的轮询算法的调度序列 `{a, a, a, a, a, b, c}` 更为平滑, 不会对 `a` 服务器造成集中访问

#### 算法说明
在每一次的选择后, 为每个可选节点的当前权重增加它的权重; 选取最大当前权重的节点, 并且将它的当前权重减去所有节点权重的总和  
`{a:5, b:1, c:1}` 的权重会生成以下当前权重的序列

| sequence | current weight before select | selected server | current weight after select |
| :--- | :--- | :--- | :--- |
| 0 | - | initial | `{a:0, b:0, c:0}` |
| 1 | `{a:5, b:1, c:1}` | a | `{a:-2, b:1, c:1}` |
| 2 | `{a:3, b:2, c:2}` | a | `{a:-4, b:2, c:2}` |
| 3 | `{a:1, b:3, c:3}` | b | `{a:1, b:-4, c:3}` |
| 4 | `{a:6, b:-3, c:4}` | a | `{a:-1, b:-3, c:4}` |
| 5 | `{a:4, b:-2, c:5}` | c | `{a:4, b:-2, c:-2}` |
| 6 | `{a:9, b:-1, c:-1}` | a | `{a:2, b:-1, c:-1}` |
| 7 | `{a:7, b:0, c:0}` | a | `{a:0, b:0, c:0}` |

#### Java 实现
```
import java.util.ArrayList;
import java.util.List;

/**
 * 平滑的加权轮询调度算法
 */
public class SmoothWeightRoundRobin {
    public static class Server {
        private String address;
        private int weight;
        private int currentWeight;

        public Server(String address, int weight) {
            this.address = address;
            this.weight = weight;
        }

        @Override
        public String toString() {
            return "Server{address='" + address + ", weight=" + weight + ", currentWeight=" + currentWeight + '}';
        }
    }

    private List<Server> servers;
    private int totalWeight;

    public SmoothWeightRoundRobin(List<Server> servers) {
        this.servers = servers;
        totalWeight = servers.stream().map(server -> server.weight).reduce((a,b) -> a + b).get();
    }

    public Server next() {
        Server best = null;
        for (Server server : servers) {
            server.currentWeight += server.weight;
            if (best == null || best.currentWeight < server.currentWeight) {
                best = server;
            }
        }
        if (best != null) {
            best.currentWeight -= totalWeight;
        }
        return best;
    }

    public static void main(String[] args) {
        List<Server> servers = new ArrayList<>();
        servers.add(new Server("192.168.0.1", 1));
        servers.add(new Server("192.168.0.2", 2));
        servers.add(new Server("192.168.0.3", 4));
        servers.add(new Server("192.168.0.4", 8));
        // smooth weight round robin
        SmoothWeightRoundRobin swrr = new SmoothWeightRoundRobin(servers);
        for (int i = 0; i < 100; i++) {
            System.out.println(swrr.next());
        }
    }

}
```

#### 其他
如果服务器的权重差别较大, 出于平滑的考虑, 避免短时间内对服务器造成冲击, 可选择平滑的加权轮询算法 (Nginx 轮询); 如果服务器的权重差别不是很大, 可考虑使用加权轮询算法 (Lvs 轮询); 两者就算法复杂度来说是相等的, 性能与实现略有关系, 但实际上两者在服务器数量不大的情况下, 每次调度都应该在 100 纳秒以内, 使用哪一种都不会成为系统的瓶颈

>**参考:**
[平滑的基于权重的轮询算法](https://colobu.com/2016/12/04/smooth-weighted-round-robin-algorithm/)
[Upstream: smooth weighted round-robin balancing](https://github.com/phusion/nginx/commit/27e94984486058d73157038f7950a0a36ecc6e35)
[平滑的加权轮询算法](http://lufred.github.io/2018/01/23/smooth_weighted_round_robin_balancing/)
