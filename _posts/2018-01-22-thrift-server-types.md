### Thrift 服务端类型
- TSimpleServer:
- TThreadPoolServer:
- TNonblockingServer:
- THsHaServer:
- TThreadedSelectorServer

### Thrift 服务协议类型
- TBinartProtocol
- TCompactProtocol
- TJSONProtocol
- TSimpleJSONProtocol


#### 服务端编码步骤
- 实现服务处理接口
- 基于接口实现类创建 TProcessor
- 创建 TServerTransport
- 创建 TProtocol
- 基于 TProcessor, TServerTransport, TProtocol 创建 TServer
- 启动 Server
#### 客户端编码步骤
- 创建 Transport
- 基于 Transport 创建 TProtocol
- 基于 TProtocol 创建 Client
- 调用 Client 的相应方法

>**参考:**
[Thrift入门及Java实例演示](https://www.jianshu.com/p/10b7cf0a384e)
[Apache Thrift - 可伸缩的跨语言服务开发框架](https://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/)
