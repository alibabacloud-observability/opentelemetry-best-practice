# OpenTelemetry 指定透传 Header 格式

> OpenTelemetry 支持 tracecontext, baggage, b3, b3multi, jaeger 等多种透传格式，不同透传格式会在 HTTP 请求中设置不同格式的 Header。下面为您介绍如何设置 OpenTelemetry 的透传格式。

## 1. OpenTelemetry 支持的透传格式

默认情况下，OpenTelemetry 使用的是 tracecontext 和 baggage。

|  透传格式名称  |  格式  |  备注  |
| --- | --- | --- |
|  tracecontext  |  traceparent : {version}-{trace-id}-{parent-id}-{trace-flags}  |  [https://www.w3.org/TR/trace-context/](https://www.w3.org/TR/trace-context/)   |
|  baggage  |   |  [https://www.w3.org/TR/baggage/](https://www.w3.org/TR/baggage/)  |
|  b3  |  b3: {TraceId}-{SpanId}-{SamplingState}-{ParentSpanId}  |   |
|  b3multi  |  X-B3-TraceId: {TraceId} X-B3-SpanId: {SpanId} X-B3-ParentSpanId: {ParentSpanId} X-B3-Sampled: {SamplingState}  |   |
|  jaeger  |  uber-trace-id : {trace-id}:{span-id}:{parent-span-id}:{flags}  |   |
|  xray  |   |   |
|  ottrace  |   |   |
|  none  |   |   |

## 2. 设置透传格式

在启动应用时，有两种方式可以设置 OpenTelemetry TraceId 的透传格式：

1. 使用 otel.propagators 参数: `-Dotel.propagators=tracecontext,baggage`
    

*  支持同时指定多种格式，用逗号隔开
    
```
-javaagent:/path/to/opentelemetry-javaagent.jar
    -Dotel.resource.attributes=service.name=<service-name>
    -Dotel.exporter.otlp.headers=Authentication=<token>
    -Dotel.exporter.otlp.endpoint=<endpoint>
    -Dotel.metrics.exporter=none
    -Dotel.propagators=tracecontext,baggage,b3
```

2.  设置 OTEL\_PROPAGATORS 环境变量，再运行程序
    

*  支持同时指定多种格式，用逗号隔开
    
```
export OTEL_PROPAGATORS="b3"
```