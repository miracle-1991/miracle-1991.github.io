---
title: Observability (可观测)
date: 2023-12-12 12:00:00 +0800
categories: [observable]
tags: [apisix, tracing, metric, backend]
author: miracle
mermaid: true
---

## 背景

![threeIndicator](/assets/img/tracing/threeIndicate.png)

随着微服务的兴起，项目开始从大型的单体结构逐步演变成职责分明的多个微服务结构。微服务之间相互调用，网络通信频繁，这使得排查一个问题变得越发困难。
可观测系统是随之而来的一个产物，该系统可以监控整个微服务系统的各项业务、性能指标，对于一些异常情况可以快速告警；对于客户端发来的每一项请求，可以
记录响应过程中经过了哪些服务，每个服务的耗时等等。


## 什么是可观测系统

可观测系统能让开发者从一个外部的、直观的视角看待微服务系统，从而更容易地掌控分布式系统。

可观测系统有三大指标需要搜集：

* trace: trace记录了请求被处理的整个过程，还记录了各个服务处理的耗时、异常等等，让我们对整个链路有直观的了解
* metric：可以实时显示服务的业务和性能指标，比如CPU、内存利用率、QPS、Latency等等，让我们可以从一个直观的、可视化的角度查看系统的状态，甚至，通过与告警系统相结合，我们可以及时获知系统的异常情况
* log: 系统日志、业务日志，记录服务的详细处理过程

## 链路追踪实践

trace 也可以称作链路追踪，可以将一次分布式请求还原成调用链路进行展示，从该链路的可视化界面中，开发者可以直观的看到
整个调用过程，从而快速定位。开发者还能直观的了解到整个过程中哪个部分最耗时，从而着重针对该高耗时的处理过程进行优化。

![tracingExample](/assets/img/tracing/tracingExample.png)

我在[Tracing Of Envoy](https://miracle-1991.github.io/posts/envoyTracing/)中曾经讲述过Envoy中是怎么实现Tracing的，当时选择了Jaeger接收
上报的Span。由于当时做调研时Envoy支持的方式并不多，基本只有Zipkin比较成熟，免费，也兼容Jaeger，所以选择了使用Zipkin进行上报。

但是在后端中，我们的选择更多一些,主要可以分为：
* Zipkin SDK
* Jaeger SDK
* Skywalking
* cat
* pinpoint
* lightstep
* ......

### 有没有统一的标准 ?
#### 标准1: [Opentracing](https://opentracing.io/)
众多的分布式追踪系统通过使用不兼容 API 的应用程序级检测进行实现，导致开发不通用。

而OpenTracing是CNCF（Cloud Native Computing Foundation projects）的项目，
它是一个与厂商无关的API，并提供了一种规范，可以帮助开发人员轻松的在他们的代码上集成tracing。

官方提供了Go, JavaScript, Java, Python, Ruby, PHP, Objective-C, C++, C#等语言的支持。
它是开发的不属于任何一家公司。

事实上有很多公司正在支持OpenTracing，例如：Zipkin和Jaeger都遵循OpenTracing协议。

![OpenTracing](/assets/img/tracing/openTracing.png)

#### 标准2: [OpenCensus](https://opencensus.io/ OpenCensus)
OpenCensus是Google开源的一套标准。
作为最早提出Tracing概念的公司，OpenCensus也是Google Dapper的社区版本；
OpenSensus是允许你采集应用程序Metrics和分布式Traces，并且支持各种语言的库 通过agent直接发送遥测数据到指定的exporter（jaeger、zipkin等）

![OpenCensus](/assets/img/tracing/openCensus.png)

#### 两个标准的区别与合并
OpenCensus把Metrics包括进来了，不仅可以采集traces，还支持采集metrics，还有一点不同OpenCensus并不是单纯的规范制定，他还把包括数据采集的Agent、Collector

![compare](/assets/img/tracing/compareOpenTracingAndOpenCensus.png)

OpenCensus与OpenTracing为了将两者的优点进行整合，宣布将进行合并，两者的合并就是后来赫赫有名：OpenTelemetry

![merge](/assets/img/tracing/mergeOpenTracingAndOpenCencus.png)

###### OpenTelemetry
OpenTelemetry与厂商、平台无关，不提供与可观测性相关的后端服务。 可根据用户需求将可观测类数据导出到存储、查询、可视化等不同后端，如 Prometheus、Jaeger 、云厂商服务中。

OpenTelemetry不是像Jaeger、Skywalking、Prometheus这框架具备存储，查询，dashboard的服务。相反，它支持将数据导出到各种开源和商业后端。它提供了一个可插拔的体系结构，因此可以轻松添加附加的技术协议和格式。

该项目得到了云提供商、厂商和最终用户的广泛行业支持和采用；

OpenTelemetry与厂商支持11种主流语言（c++、.net、erlang、go、java、js、php、py、ruby、rust、swift）,
其中[opentelemetry-go](https://github.com/open-telemetry/opentelemetry-go)是go语言的opentelemetry实现，目前go的metrics到了贝塔版本，log还没有开发完成，不过trace是比较成熟的了


![openTelemetry](/assets/img/tracing/openTelemetry.png)

###### opentelemetry collector（简称otel collector）

从上图中能看到，要想使用opentelemetry, 需要单独部署一个collector。

该collector作为中间层，能接收telemetry数据，处理数据然后将数据发送出去。
正常数据通过otel sdk到collector，经过collector处理后，再发送到下游。

当然，也可以直接将数据发送下游而不经过collector，用collector的好处是可以进行数据加工，可以以sidecar的方式部署和业务容器一起部署，这样业务只需要把业务数据发送出去，而不用关心网络问题、发送失败重试等。

### 服务中使用Opentelemetry

由于我的服务是在docker-compose中部署的，因此，在服务启动之前，首先应该启动
* Collector: 作为中间层，用于接收Opentelemetry SDK上报的数据，转发到Span接收者
* Jaeger: 接收Span，并提供链路的展示

因此首先启动collector和jaeger两个容器：
```
...
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./opentelemetry_conf/otel-collector-config.yaml:/etc/otel-collector-config.yaml
    networks:
      - apisix
    depends_on:
      - jaeger
      - prometheus
    ports:
      - "1888:1888"   #pprof expand
      - "8888:8888"   #receive prometheus data
      - "8889:8889"   #output prometheus data
      - "13133:13133" #health check expand
      - "4317:4317"   #OTLP grpc receiver
      - "4318:4318"   #OTLP http receiver
      - "55679:55679" #zpages extension
      - "55680:55680" #OTLP protocol data

  jaeger:
    image: jaegertracing/all-in-one:latest
    restart: always
    ports:
      - "16686:16686" # jaeger UI port
      - "14268:14268" # receive data from OTEL Collector
      - "14250:14250" # receive data from OTEL Collector
    networks:
      apisix:
...
```

在启动时，collector需要提供相应的配置文件([config.yaml](https://github.com/miracle-1991/apiGateWay/blob/master/server/opentelemetry_conf/otel-collector-config.yaml))
```
receivers: #这部分配置了OpenTelemetry collector从哪些接收端收集数据。如：`otlp`（OpenTelemetry 标准的收集器）通过 gRPC 和 HTTP 获取数据。
  otlp:
    protocols:
      grpc:
      http:  #receive data from apisix or server, default port 4318

processors: #配置了预处理这些收集到的数据的一系列流程。如：`batch` 这个处理器会将多个数据点打包到一个批次，然后再将批次作为一个整体进行处理。
  batch:

exporters: #配置了OpenTelemetry collector将处理后的数据输出到何处。如：`otlp`, `prometheus`, `debug`, 和 `logging`，他们分别会将数据发送到配置的 `endpoint` 。例如， `otlp` exporter 会将数据发送到 `jaeger:4317`。
  otlp:
    endpoint: jaeger:4317 # send data to jaeger or zipkin
    tls:
      insecure: true

  debug:

  prometheus:
    endpoint: "0.0.0.0:8889"

  logging:
    loglevel: debug

extensions: #扩展为 collector 提供一些额外的功能，例如：运行状况检查 (`health_check`)，性能剖析 (`pprof`)，页面状态检查 (`zpages`)。
  health_check:
  pprof:
    endpoint: :1888
  zpages:
    endpoint: :55679

service:
  extensions: [pprof, zpages, health_check]
  pipelines: # 配置了 collector 启动时应激活哪些扩展和管道。管道（pipelines）由一个或多个收集器、处理器和输出器组成，数据通过管道从接收到输出。
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug,otlp]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug,prometheus]
```

然后在服务启动之前先进行注册([Register](https://github.com/miracle-1991/apiGateWay/blob/master/server/echo/observable/trace/trace.go)),在注册的过程中设置好
collector的地址、采样频率：
```
...
ctx, cancel := context.WithTimeout(ctx, time.Second)
	defer cancel()
	conn, err := grpc.DialContext(ctx,
	  config.OTEL_ADDR, //Collector的地址
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithBlock(),
	)
	if err != nil {
		return fmt.Errorf("failed to create gRPC connection to collector: %w", err)
	}

	// Set up a trace exporter
	traceExporter, err := otlptracegrpc.New(ctx, otlptracegrpc.WithGRPCConn(conn))
	if err != nil {
		return fmt.Errorf("failed to create trace exporter: %w", err)
	}

	// Register the trace exporter with a TracerProvider, using a batch
	// span processor to aggregate spans before export.
	bsp := sdktrace.NewBatchSpanProcessor(traceExporter)
	tracerProvider := sdktrace.NewTracerProvider(
		sdktrace.WithSampler(sdktrace.AlwaysSample()),  //全采样
		sdktrace.WithResource(res),
		sdktrace.WithSpanProcessor(bsp),
	)
...
```
然后就可以很方便的使用Tracing:
```
func (i *IMPL) Hello(ctx context.Context) (int, string, error) {
	ctx, span := trace.Tracer.Start(ctx, "hello") //生成新Span,或者继承父Span生成子Span
	defer span.End()  //上报Span

	return config.OK, "hello world", nil
}
```

### 网关中使用Opentelemetry

作为一个完整流程，一个请求首先是从客户端/浏览器发到网关，然后网关将其转发到后端服务。因此，从后端的视角来看，一个
请求链路的起始就是网关。因此很有比较在网关中对请求进行链路追踪，从而才能够发现该请求从哪里开始、分配到了哪个节点，最后又怎么结束。

由于我使用的是apisix作为网关，因此首先应该在启动apisix之前使其支持opentelemetry插件：
```
...
plugins:
  - opentelemetry #开启opentelemetry插件

plugin_attr:
  opentelemetry: #为opentelemetry插件进行初始化设置
    trace_id_source: x-request-id
    resource:
      service.name: APISIX
    collector:
      address: otel-collector:4318  # OTLP HTTP Receiver address
      request_timeout: 3
```
如此，就完成了插件的配置，apisix中的请求也会自动上报到collector

### Tracing效果展示

在apisix上搭建路由如下所示:

![postman](/assets/img/tracing/postmanToTracing.png)

发起curl请求:

```
curl --location 'http://127.0.0.1:9080/hashfill' \
--header 'Content-Type: application/json' \
--data '{
    "boundaryName": "chengdu",
    "precision": 6,
    "boundary": {
        "polygons": [
            {
                "vertices": [
                    {"lat": 30.725932779733828,"lon": 104.04810399201176},
                    {"lat": 30.718933590173382,"lon": 104.10548984714013},
                    {"lat": 30.712244330215963,"lon": 104.13622682435815},
                    {"lat": 30.702910389863735,"lon": 104.15873257503482},
                    {"lat": 30.66954989275946,"lon": 104.16300567773708},
                    {"lat": 30.611560542033516,"lon": 104.14115260383586},
                    {"lat": 30.60368261457832,"lon": 104.13460495126948},
                    {"lat": 30.597337315209458,"lon": 104.12197165037554},
                    {"lat": 30.594520110636584,"lon": 104.09585145457078},
                    {"lat": 30.597848397395246,"lon": 104.08054653476084},
                    {"lat": 30.59897710829587,"lon": 104.04449447431418},
                    {"lat": 30.626768854130056,"lon": 104.0006597543566},
                    {"lat": 30.652212454847685,"lon": 103.98381719828197},
                    {"lat": 30.67412880125752,"lon": 103.99050661667191},
                    {"lat": 30.701291076077624,"lon": 104.00698694578595},
                    {"lat": 30.70704013040491,"lon": 104.02065715840023}
                ]
            }
        ]
    }
}'
```
发起请求后，能够在[jaeger ui](http://127.0.0.1:16686/)上看到整个链路：

![jaegerui](/assets/img/tracing/jaeger.png)

## 监控实践

监控可以说是每个互联网公司必备的观测工具了。通过监控，我们可以直观的看到业务或系统中的各项指标，并根据指标配置相应的告警，及时对业务或者系统
进行响应(oncall)，对于业务的稳定性至关重要。

而社会上的监控系统可以说已经相当成熟，可以在业务中很方便的接入，有开源免费的prometheus+grafana，也有收费但是非常方便使用的DataDog。此处，我使用
开源的prometheus+grafana演示监控系统的使用。

首先，介绍一下prometheus的架构:

![prometheus-structure](/assets/img/metric/prometheus-structrue.png)

* **搜集数据**： prometheus其实支持推和拉两种搜集方式：
  * Push: 服务将自己的监控主动push到prometheus server：
    * 优点：适合监控短生命周期的服务，或者位于网络环境复杂和网络稳定性差的区域的服务
    * 不足：如果推送过于频繁，会给 Prometheus 服务器带来很大压力；而且推模式下，一旦 Prometheus 服务器出现问题，所有的监控数据都将丢失，风险较高
  * Pull(默认方式): 服务将监控数据先保存到本地，然后通过类似于/metrics的endpoint将这些数据暴露出去，prometheus server根据配置的采样间隔定期去请求/metrics拉数据：
    * 优点：配置灵活，Prometheus 只需要知道被监控端点的位置；方便于版本更新与问题排查；对网络中断有较好的容忍度。
    * 不足：如果被监控的实例非常多，会给 Prometheus 服务器带来很大压力；另外，瞬时故障可能不容易被发现。
* **服务发现**：prometheus支持多种服务发现,比如DNS、K8s、Consul
* **存储**: prometheus默认使用本地磁盘的时间序列数据库存储数据，它将数据分成块(blocks)，每个块包含两个小时的时间序列数据。每个块都可能会被压缩和删除。prometheus默认会存储15天内的数据，如果希望长期保存，还需要使用prometheus的远程存储功能集成其他存储系统，比如S3
* **查询**：Prometheus 提供了一种非常强大的数据查询语言，称之为 PromQL（Prometheus Query Language）。你可以使用 PromQL 来对 Prometheus 存储的时间序列数据进行查询和分析。下面是一些 PromQL 的基础查询例子：
  * **查询单个时间序列：** 直接输入该时间序列的 metric 名字，就可以查询到该 metric 的所有时间序列数据。例如：
    ```promql
    http_requests_total
    ```
    这将查询名为 `http_requests_total` 的 metric 对应的所有时间序列。
  * **基于标签筛选时间序列：** 在 metric 名后添加一对大括号 `{}`，在括号内部填入标签筛选条件。例如：

     ```promql
     http_requests_total{method="GET"}
     ```
    这将筛选出所有 method 标签值为 "GET" 的 `http_requests_total` 时间序列。
  * **对时间序列进行聚合：** PromQL 支持多种聚合操作符，如 `sum`、`avg`、`min`、`max`、`count` 等。例如：

     ```promql
     sum(http_requests_total)
     ```
    这将计算所有 `http_requests_total` 时间序列的总和。

  PromQL 的表达式可以是非常复杂的，支持多种函数、操作符以及时间范围选择等。你可以在 [Prometheus 官方文档](https://prometheus.io/docs/prometheus/latest/querying/basics/) 中学习更多关于 PromQL 的用法。

prometheus还可以通过与grafana进行集成，从而直观地在grafana的UI上显示监控的数据。
而告警系统比如PagerDuty可以通过与prometheus的webhook集成，从而及时对prometheus中的告警进行响应，将告警消息以电话的方式通知给oncall人员。

### 业务中使用prometheus
OpenTelemetry（OpenTelemetry Protocol，简称OTEL）为多种监测后端系统提供了 Metrics 导出器，允许将收集到的度量数据发送至指定的存储和分析系统。以下是一些由OpenTelemetry官方或社区提供的Metrics导出器：

1. **Prometheus Exporter：** Prometheus 是一个广泛使用的开源监控和告警工具包。这个导出器可将度量数据以 Prometheus 可抓取的格式提供。

2. **OTLP Exporter：** OTLP 是 OpenTelemetry Protocol 的缩写。其允许使用 gRPC 或 HTTP 协议将度量数据发送到任何支持 OTLP 的后端。

3. **InfluxDB Exporter**：此导出器可以将度量数据发送到 InfluxDB，一款开源时序数据库。

此处我们使用Prometheus导出器，将监控数据上报到prometheus
```
var meter otelMetric.Meter
var funcDurationHistogram otelMetric.Float64Histogram

func Init() error {
	exporter, err := prometheus.New() //创建 Prometheus 导出器（Exporter,导出器定义度量数据会发送到哪里。在这个例子中，度量数据将会被发送到 Prometheus。
	if err != nil {
		return errors.New(fmt.Sprintf("failed to new prometheus: %v", err))
	}

	ctx := context.Background()
	res, err := resource.New(ctx, //创建 Resource： 一个 Resource 代表了 OpenTelemetry 观测数据的源头（例如，一个特定的微服务或者主机）。这里，Resource 的服务名称被设置为了服务名
		resource.WithAttributes(
			semconv.ServiceName(config.SERVICENAME),
		),
	)
	if err != nil {
		return errors.New(fmt.Sprintf("failed to new resource: %v", err))
	}

	meterProvider := metric.NewMeterProvider( //设置 OpenTelemetry MeterProvider： MeterProvider 控制着度量数据的收集。本段代码将它配置为使用刚创建的 Resource 和 Exporter。
		metric.WithResource(res),
		metric.WithReader(exporter),
	)

	otel.SetMeterProvider(meterProvider)
	meter = otel.Meter(config.SERVICENAME)
	funcDurationHistogram, err = meter.Float64Histogram( //创建 Histogram（直方图）度量： Histogram 是一个统计工具，用于测量数据分布。这段代码创建的 Histogram 会测量函数调用的时长。
		"echo_api_duration",
		otelMetric.WithDescription("duration of func call"),
		otelMetric.WithUnit("ms"),
	)
	if err != nil {
		return errors.New(fmt.Sprintf("failed to create histogram, error: %v", err))
	}

	return nil
}
```

为了在业务中方便的使用监控，还可以提供一些封装好的上报接口供业务使用，比如Count1,Count,Duration...，此处提供Duration的参考实现:
```
func Duration(ctx context.Context, funName string, start time.Time) {
	duration := time.Since(start)
	label := attribute.Key("func").String(funName)
	funcDurationHistogram.Record(ctx, float64(duration.Nanoseconds()/1000000), otelMetric.WithAttributes(label))
}
```
然后业务中就可以很简便地使用这种方式进行上报：
```
func (i *IMPL) FillGeoHash(ctx context.Context, request *echo.FillGeoHashRequest) (*echo.FillGeoHashResponse, error) {
	ctx, span := trace.Tracer.Start(ctx, "http-service-fillgeohash")
	defer span.End()
	defer metric.Duration(ctx, "http-service-fillgeohash", time.Now()) //上报监控数据

	hashes, err := utils.FillGeoHash(ctx, request)
	if err != nil {
		return nil, err
	}
	return hashes, nil
}
```
一切准备好之后，还需要在服务启动之前，提供好endpoint供prometheus sever主动pull数据:
```
...
	r.GET("/metrics", gin.WrapH(promhttp.Handler()))
...
```
当服务启动后，访问服务的/metrics endpoint应该能看到缓存在服务节点内存中的监控数据:

![echoMetric](/assets/img/metric/echoMetricData.png)

### apisix中上报监控数据
apisix已经对prometheus提供了比较好的支持，只需要简单的配置。首先配置apisix的config.yml，在启动时允许使用prometheus插件，并配置endpoint的端口:
```
plugins:
  - prometheus #启动apisix时启动prometheus插件

plugin_attr:
  prometheus:
    export_addr:
      ip: "0.0.0.0"
      port: 9091 //在0.0.0.0:9091提供endpoint供prometheus访问监控数据
```

然后在路由中使用prometheus:
```
{
  "uri": "/*",
  "name": "test",
  "methods": [
    "GET",
    "POST",
    "PUT",
    "DELETE",
    "PATCH",
    "HEAD",
    "OPTIONS",
    "CONNECT",
    "TRACE",
    "PURGE"
  ],
  "plugins": {
    "prometheus": { #在该条路由中使用prometheus
      "_meta": {
        "disable": false
      }
    }
  },
  "upstream": {
    "timeout": {
      "connect": 6,
      "send": 6,
      "read": 6
    },
    "type": "roundrobin",
    "scheme": "http",
    "discovery_type": "consul",
    "pass_host": "pass",
    "service_name": "echo_http",
    "keepalive_pool": {
      "idle_timeout": 60,
      "requests": 1000,
      "size": 320
    }
  },
  "status": 1
}
```
当client访问该条路由时，apisix会主动将监控数据统计下来，供prometheus拉取

### prometheus采集配置
导出器配置好后，prometheus还需要知道导出器的位置以及导出器提供的endpoint
```
scrape_configs:
  - job_name: echo-exporter #业务服务的导出器：通过consul发现echo_http服务，访问其/metrics endpoint获取监控数据
    metrics_path: /metrics
    scheme: http
    consul_sd_configs:
      - server: consul:8500
        services:
          - echo_http
  - job_name: "apisix" #网关的导出器: 通过http://apisix:9091访问apisix，访问/apisix/prometheus/metrics获取监控数据
    scrape_interval: 5s
    metrics_path: "/apisix/prometheus/metrics"
    static_configs:
      - targets: ["apisix:9091"]
```

### grafana中使用prometheus
grafana可以以单独的服务启动，启动后，访问其UI，以管理员角色登陆后，可以设置监控数据的来源:

![usePrometheusAsDataSource](/assets/img/metric/grafanaUsePrometheusAsDataSource.png)

然后就可以在Dashboard中使用上报的数据了：

![useMetricData](/assets/img/metric/useMetricDataInGrafana.png)

### 效果展示

![apisixDashboard](/assets/img/metric/grafanaApisix.png)
