# 将 TraceId 自动写入 HTTP Response Header

> 默认情况下，TraceId 只会在保存在 HTTP Requst Header 中，如果您需要在 HTTP Response Header 中设置 TraceId，可参考本文方法，编写扩展程序以增强 OpenTelemetry Java Agent 的功能。

## 1. 前提条件
* 使用 OpenTelemetry Java Agent
* OpenTelemetry Java Agent 版本 >= 1.24.0

## 2. 编写 OpenTelemetry Java Agent 扩展

1. pom.xml 中添加依赖
```xml
  <dependencies>
      <dependency>
          <groupId>com.google.auto.service</groupId>
          <artifactId>auto-service</artifactId>
          <version>1.1.1</version>
      </dependency>

      <dependency>
          <groupId>io.opentelemetry.javaagent</groupId>
          <artifactId>opentelemetry-javaagent</artifactId>
          <version>1.28.0</version>
          <scope>compile</scope>
      </dependency>
  </dependencies>
```

2. 新建 AgentHttpResponseCustomizer 类，实现 HttpServerResponseCustomizer 接口

```java
package org.example;

import com.google.auto.service.AutoService;
import io.opentelemetry.javaagent.shaded.io.opentelemetry.api.trace.Span;
import io.opentelemetry.javaagent.shaded.io.opentelemetry.api.trace.SpanContext;
import io.opentelemetry.javaagent.shaded.io.opentelemetry.context.Context;
import io.opentelemetry.javaagent.bootstrap.http.HttpServerResponseCustomizer;
import io.opentelemetry.javaagent.bootstrap.http.HttpServerResponseMutator;

@AutoService(HttpServerResponseCustomizer.class)
public class AgentHttpResponseCustomizer implements HttpServerResponseCustomizer {
    @Override
    public <RESPONSE> void customize(Context context, RESPONSE response, HttpServerResponseMutator<RESPONSE> responseMutator) {
        SpanContext spanContext = Span.fromContext(context).getSpanContext();
        String traceId = spanContext.getTraceId();
        String spanId = spanContext.getSpanId();

        // 在 HTTP Response Header中设置 traceId 和 spanId, Header 字段名可以自定义
        responseMutator.appendHeader(response, "TraceId", traceId);
        responseMutator.appendHeader(response, "SpanId", spanId);
    }
}
```

## 3. 构建
* 将程序打包成 JAR 包，构建后存储在 target 目录下
```
mvn clean pacakage
```

## 4. 启动应用时加载 OpenTelemetry Java Agent 扩展
* 在原有启动参数上添加 otel.javaagent.extensions 参数

`-Dotel.javaagent.extensions=/path/to/opentelemetry-java-agent-extension.jar`

* 例如
```
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.extensions=path/to/opentelemetry-java-agent-extension.jar \
     -Dotel.exporter.otlp.headers=Authentication=<token> \
     -Dotel.exporter.otlp.endpoint=<endpoint> \
     -Dotel.metrics.exporter=none 
     -jar yourapp.jar
```