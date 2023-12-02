---
title: Tracing Of Envoy
date: 2023-10-19 10:00:00 +0800
categories: [service_mesh]
tags: [envoy, tracing]
author: miracle
mermaid: true
---

***
## 背景
### Opentracing的基本概念
Opentracing是一个分布式追踪标准，与平台和语言无关，统一接口，方便接入不同的分布式追踪系统。
Jaeger也是支持Opentracing标准的项目之一。
学习jaeger有必要了解Opentracing规范。 jaeger的文档资料：
[opentracing中文连接地址](https://wu-sheng.gitbooks.io/opentracing-io/content)

#### 基本概念：
* Trace：表示对一次请求完整调用链的跟踪，可以认为是多个span的有向无环图
* __Span__：一个span代表系统中具有开始时间和执行时长的逻辑运行单元。span之间通过嵌套或顺序排列简历逻辑因果关系
* Operation Name：每个span都有一个操作名称，这个名称要求具有可读性(例如: 一个RPC方法的名称，一个函数的名称，
或一个大型计算过程中的子任务或阶段)。operation name应该是抽象的、通用的标识和明确的、具有统计意义的名称。更具体的子类型的描述请使用tag
* Inter-Span References：一个span可以和一个或多个span间存在因果关系。OpenTracing定义了两种关系ChildOf和FollowsFrom。
这两种引用类型代表了子节点和父节点间的直接因果关系。
* Log：每个span可以进行多次Log操作，每次Log操作都需要一个带时间戳的时间名称，以及可选的任意大小的存储结构
* Tag：每个span可以有多个简直对形式的Tag，Tag没有时间戳。Tag主要用来对span进行简单的注解和补充
* SpanContext：每个span必须提供方法访问SpanContext。SpanContext代表跨越进程边界，传递到下级span的状态。
例如包含元组<trace_id, span_id, sampled>
* Baggage：是存储在SpanContext中的一个键值对(SpanContext)集合，它会在一条追踪链路上的所有span内全局传输，
包含这些span对应的SpanContext。Baggage功能强大，消耗也很大。因为Baggage全局传输，如果包含的数据量太大或元素太多，
将会降低系统的吞吐量或增加RPC的延迟

##### Golang Jaeger Example:
本项目使用golang实现了一个tracing的example，并将tracing信息上报到jaeger: [go-tracing](https://github.com/miracle-1991/envoy-study/tree/main/tracing/go-tracing)
上报过程如下:

![tracing上报到jaeger](/assets/img/envoy/tracing/jaeger-architecture.png)

其中的组件如下:
* Client: Jager客户端，是OpenTracing API的具体语言实现；
* Agent: 监听在UDP端口的守护进程，用于接受Client发送的追踪数据，并将数据批量发送给Collector；
* Collector: 接收Agent的数据，验证数据并建立索引，最后异步写入到后端存储；
* DataBase：后端存储组件，支持内存、ES、Kafka等方式；
* Query: 用于接收查询请求，从数据库中检索数据并通过UI展示

按照example的[运行说明](https://github.com/miracle-1991/envoy-study/tree/main/tracing/go-tracing/README.md)运行后，就能在[Jaeger-UI](http://localhost:16686/)中看到运行的结果。
## Envoy Tracing
### 官方Jaeger Example
golang实现的tracing需要自己使用jaeger-client上报span，而envoy将这个上报的过程集成到了envoy内部，开发者不再需要
在代码中添加这些侵入性的代码，所有的tracing由envoy自行完成。
envoy官方的例子中已经提供了jaeger的测试例子：
[jaeger-tracing](https://github.com/envoyproxy/envoy/tree/main/examples/jaeger-tracing)
在[jaeger-tracing](https://github.com/envoyproxy/envoy/tree/main/examples/jaeger-tracing)下执行以下命令即可运行该demo,
并能在[Jaeger-UI](http://localhost:16686/)看到上报的结果:
```
docker-compose pull
docker-compose up --build -d
curl -v http://localhost:8000/trace/1
```
该demo的运行过程如下所示:

![example-jaeger-tracing](/assets/img/envoy/tracing/envoy-example-jaeger-tracing.png)

* front:接受curl请求，并向service1发起http请求
* service1: 转发http请求，转发过程中透传各种header
* service2: 接收http请求，并做出反应
* envoy: 解析http headers，创建root span或者child span，并异步上报span到jaeger
### Grpc Example
本项目中也实现了一个tracing的Example，与官方的区别是service之间的通信是通过grpc，而不是http。
example代码位置[grpc envoy tracing](https://github.com/miracle-1991/envoy-study/tree/main/tracing/envoy-tracing/README.md)

### envoy内部tracing过程
在envoy内部，tracing的主要过程如下:

![envoy的请求处理过程](/assets/img/envoy/tracing/filter-work.png)

envoy的内部处理过程：

![tracing内部过程](/assets/img/envoy/tracing/tracing_internal_process.png)

* Listener Filter:侦听器过滤器(tls)
* Connection:建立连接
* TCP Filter Manager: TCP管理器，管理L4过滤器（读取原始套接字），分为read/write filters
* HTTP Codec: HTTP编解码（L4）
* HTTP Conn Manager: HTTP管理器， 管理L7过滤器（读取header/body）

http conn manager使用tracing:
* startSpan: request时，http conn manager会解析header,解析过程中会调用startSpan，创建rootSpan或者childSpan
  * [http conn manager解析header时startSpan](https://github.com/envoyproxy/envoy/blob/main/source/common/http/conn_manager_impl.cc)
* injectContext: 当建立好与service的连接并发起真实请求时，会将创建好的span插入到header中
  * [onPoolReady中injectContext](https://github.com/envoyproxy/envoy/blob/main/source/common/router/upstream_request.cc)
* finishSpan：当请求结束时，会调用finishSpan，将待上传的span缓存起来，异步发送给Jaeger
  * [completeRequest中调用finalizeDownstreamSpan，finalizeDownstreamSpan进而调用finishSpan](https://github.com/envoyproxy/envoy/blob/main/source/common/tracing/http_tracer_impl.cc)

startSpan、finishSpan、injectContext等主要接口定义在[envoy/envoy/tracing](https://github.com/envoyproxy/envoy/tree/main/envoy/tracing)中
。envoy/tracing定义了tracing组件需要实现的接口，这些接口都是纯虚接口:

![envoy/envoy/tracing interface definition](/assets/img/envoy/tracing/envoy-tracing-functions.png)

主要的接口说明:
* [startSpan](https://github.com/envoyproxy/envoy/tree/main/envoy/tracing/trace_driver.h)：创建一个新的spanContext，可以是rootSpan或者childSpan
  * *params1* [config](https://github.com/envoyproxy/envoy/tree/main/envoy/tracing/trace_config.h): yaml中的配置，用于读取用户指定的配置值
  * *params2* [RequestHeaderMap](https://github.com/envoyproxy/envoy/blob/main/envoy/http/header_map.h): http请求的headers，该文件实现了TraceContext接口，可以方便地读取/设置header
  * *return* [Span]((https://github.com/envoyproxy/envoy/tree/main/envoy/tracing/trace_driver.h): 创建的spanContext，提供了以下接口用于方便地读取/设置值：
    * setTag：可以修改span标签
    * injectContext：将context注入到http header中
    * finishSpan: 结束span，将span缓存，异步发送给jaeger


[envoy/tracing]((https://github.com/envoyproxy/envoy/tree/main/envoy/tracing)主要提供接口定义，具体的实现位于source目录中。空实现以及扩展实现如下图所示:

![tracing的空实现以及扩展实现](/assets/img/envoy/tracing/tracing-null-impl.png)


* [envoy/source/common/tracing](https://github.com/envoyproxy/envoy/tree/main/source/common/tracing)：提供了空实现，并管理扩展实现
  * [custom_tag_lib](https://github.com/envoyproxy/envoy/blob/main/source/common/tracing/custom_tag_impl.h): 主要定义了如何读取标签值并将标签设置到span中

![读取的各种实现](/assets/img/envoy/tracing/common-tracing-custom_tag_lib.png)

  * [http_tracer_manager_lib](https://github.com/envoyproxy/envoy/blob/main/source/common/tracing/tracer_manager_impl.h): 管理http_tracer，使用hash map保存了config与tracer的对应关系，向外提供获取tracer的接口
  * [http_tracer_lib](https://github.com/envoyproxy/envoy/blob/main/source/common/tracing/tracer_impl.h): 主要定义了三部分
    * HttpTracerUtility: 全部是静态函数，路由时结束时可以被调用，上报tracing log
    * HttpNullTracer: 空实现,默认Tracer
    * HttpTracerImpl: 调用默认或者扩展的Driver实现的startSpan接口
* [envoy/source/extensions/tracers](https://github.com/envoyproxy/envoy/tree/main/source/extensions/tracers): 提供了各种扩展实现
  * [dynamic_ot](https://www.aftership.com/zh-hans/couriers/dynamic-logistics): Dynamic Logistics是泰国企业的物流和履行服务提供商
  * [lightstep](https://lightstep.com/): 公司由前Google工程师Ben Sigelman于2015年成立（创始人曾经是Dapper的开发者，专注于分布式链路追踪），LightStep的使命是削减软件的规模和复杂性，帮助公司能够持续保持对其系统的控制。
  * [opencensus](https://opencensus.io/): Google 开源的一个用来收集和追踪应用指标的第三方库。OpenCensus 能够提供了一套统一的测量工具：跨服务捕获跟踪跨度（span）、应用级别指标以及来自其他应用的元数据
  * [skywalking](https://dubbo.apache.org/zh/docs/v2.7/admin/ops/skywalking/):  专门为微服务架构和云原生架构系统而设计并且支持分布式链路追踪的APM系统。Apache Skywalking 通过加载探针的方式收集应用调用链路信息，并对采集的调用链路信息进行分析，生成应用间关系和服务间关系以及服务指标。
  * [xray](https://aws.amazon.com/cn/xray/): 亚马逊提供的分析与调试分布式生产应用程序
  * [zipkin](https://zipkin.io/): 开源的分布式追踪系统。在微服务架构下，它用于帮助收集排查潜在问题的时序数据。它同时管理数据收集和数据查询

##### Zipkin扩展实现
  zipkin作为开源的分布式追踪系统，兼容jaeger，应用广泛，envoy官方的example就以zipkin为例。因此，此处针对zipkin的实现进行分析。
##### Zipkin的注册与查找

![zipkin的注册与查找](/assets/img/envoy/tracing/zipkin-register-find.png)
* [REGISTER_FACTORY](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/config.cc)：envoy在main执行之前，zipkin通过该宏定义将自身注册到[全局map](https://github.com/envoyproxy/envoy/blob/main/envoy/registry/registry.h)中；
* [getOrCreateHttpTracer](https://github.com/envoyproxy/envoy/blob/main/source/common/tracing/tracer_manager_impl.cc): envoy启动时，会读取yaml中，通过yaml中配置的名字找到能创建tracer的factory;
  * [http conn manager config 读取配置时创建tracer](https://github.com/envoyproxy/envoy/blob/main/source/extensions/filters/network/http_connection_manager/config.cc)
```
 if (config.has_tracing()) {
    http_tracer_ = http_tracer_manager.getOrCreateHttpTracer(getPerFilterTracerConfig(config));
    ....
 }
```

在找到[ZipkinTracerFactory](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/config.h)后，[getOrCreateHttpTracer](https://github.com/envoyproxy/envoy/blob/main/source/common/tracing/tracer_manager_impl.cc)
就能创建出http tracer对象，供[http conn manager](https://github.com/envoyproxy/envoy/blob/main/source/common/http/conn_manager_impl.cc)使用：
```
  active_span_ = connection_manager_.tracer().startSpan(
      *this, *request_headers_, filter_manager_.streamInfo(), tracing_decision);
```

[http tracer](https://github.com/envoyproxy/envoy/blob/main/source/common/tracing/http_tracer_impl.cc)在startSpan时，实际上是调用的[zipkin的Driver](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/zipkin_tracer_impl.h)实现的startSpan,该函数逻辑如下:

![zipkin driver startSpan](/assets/img/envoy/tracing/zipkin_startSpan.png)

* [extractSpanContext](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/span_context_extractor.h): 主要用于提取http头中的以下header:
  * x-b3-traceid: traceid
  * x-b3-spanid: spanid
  * x-b3-sampled: 采样标志
  * x-b3-parentspanid: parentid
* [Tracer::startSpan](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/tracer.cc): 用于创建root span和child span。创建的过程中主要是生成新的traceid、spanid和parentid,并保存到[ZipkinSpan](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/zipkin_tracer_impl.h)中，在连接建立好后被使用。

[ZipkinSpan](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/zipkin_tracer_impl.h)实现了[Tracing::Span](https://github.com/envoyproxy/envoy/blob/main/envoy/tracing/trace_driver.h),
最主要实现了以下几个接口：
* [injectContext](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/zipkin_tracer_impl.cc): 在与upstram的连接真实建立后才会被router调用,将startSpan生成的traceID、spanID、parentID设置到http header中
* [finishSpan](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/zipkin_tracer_impl.cc): 在请求结束时会被调用，最终的实现为[Span::finish](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/zipkin_core_types.cc),主要是记录以下请求的耗时，并最终调用[reportSpan](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/zipkin_tracer_impl.h)进行上报。


zipkin实现的上报主要实现于[ReporterImpl::reportSpan](https://github.com/envoyproxy/envoy/blob/main/source/extensions/tracers/zipkin/zipkin_tracer_impl.cc)
主要逻辑如下:

![zipkin_flush_span](/assets/img/envoy/tracing/zipkin_flush_span.png)

最终，zipkin的总体过程可总结如下:

![zipkin总体过程](/assets/img/envoy/tracing/zipkin-sum.png)

附：上报的span
```
[{
    "duration": 11582,
    "timestamp": 1652170632495925,
    "traceId": "7fe0fc996f632e6e",
    "id": "7fe0fc996f632e6e",
    "annotations": [{
        "value": "ss",
        "timestamp": 1652170632508321
    }],
    "name": "localhost:8000",
    "tags": {
        "guid:x-request-id": "ad007532-d550-9882-a91b-4c6b9cd872d7",
        "user_agent": "grpc-go/1.45.0",
        "node_id": "load-balancer",
        "peer.address": "172.23.0.1",
        "request_size": "12",
        "http.method": "POST",
        "grpc.status_code": "0",
        "upstream_cluster.name": "string_server",
        "grpc.authority": "localhost:8000",
        "response_flags": "-",
        "http.status_code": "200",
        "http.url": "http://localhost:8000/pb.StringService/Concat",
        "grpc.timeout": "9990195u",
        "grpc.content_type": "application/grpc",
        "downstream_cluster": "-",
        "grpc.path": "/pb.StringService/Concat",
        "component": "proxy",
        "response_size": "34",
        "upstream_cluster": "string_server",
        "http.protocol": "HTTP/2"
    },
    "kind": "SERVER",
    "localEndpoint": {
        "port": 0,
        "ipv4": "172.23.0.3",
        "serviceName": "load-balancer"
    }
}]
```
