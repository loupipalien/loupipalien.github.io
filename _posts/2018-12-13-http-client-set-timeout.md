---
layout: post
title: "HttpClient 设置 Timeout"
date: "2018-12-13"
description: "Http Client 设置 Timeout"
tag: [java, http]
---

### HttpClient 的 Timeout
- ConnectionRequestTimeout
此字段设置从 Connection Manager (连接池) 中获取 Connection 超时时间, 单位毫秒, (V4.3 及以后版本才可使用); 即连接时尝试从连接池中获取, 若等待一定时间后没有获取到可用连接则超时

- ConnectionTimeout
此字段设置连接超时时间, 单位毫秒; 即与目标 URL 的建立连接超时时间, 在限定的时间内没有建立则超时
- SocketTimeout
此字段设置请求获取数据的超时时间 (即响应时间), 单位毫秒; 即建立连接后, 在限定时间内接口没有返回数据则超时

`org.apache.http.client.config.RequestConfig` 中注释表示以上三个参数默认值是 -1, 即使用系统默认值, 设置为 0 则表示一个无限的 Timeout

### HttpClient 设置 Timeout

#### V4.2 版本及以前
```
HttpClient httpClient=newDefaultHttpClient();
// 连接时间
httpClient.getParams().setParameter(CoreConnectionPNames.CONNECTION_TIMEOUT, 2000);
// 数据传输时间
httpClient.getParams().setParameter(CoreConnectionPNames.SO_TIMEOUT, 2000);
```

#### V4.3 版本及以后 (推荐)
```
CloseableHttpClient httpClient = HttpClients.createDefault();
// Get请求, 其他请求方式类似
HttpGet httpGet=newHttpGet("http://www.baidu.com");
// 设置请求和传输超时时间
httpGet.setConfig(RequestConfig.custom().setSocketTimeout(2000).setConnectTimeout(2000).build());
// 执行请求
httpClient.execute(httpGet);
```

> 参考:
[HttpClient超时设置详解](https://blog.csdn.net/u011191463/article/details/78664896)  
[HttpClient Timeout设置](https://my.oschina.net/dabird/blog/842433)
