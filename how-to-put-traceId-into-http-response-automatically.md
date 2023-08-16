# 将 TraceId 自动写入 HTTP Response Header (Java & NodeJs)

# Java

> 默认情况下，TraceId 只会在保存在 HTTP Requst Header 中，如果您需要在 HTTP Response Header 中设置 TraceId，可参考本教程，使用 OpenTelemetry Java Agent Extension(扩展)以增强 OpenTelemetry Java Agent 的功能。

本教程提供了两种方法：

* 方法一：开箱即用。直接使用我们已经打包好的 OpenTelemetry Java Agent 扩展，简单快捷。

* 方法二：自行实现 OpenTelemetry Java Agent 扩展。如果我们的扩展不满足您的需求，可以参考此方法实现 OpenTelemetry Java Agent 扩展并打成 JAR 包。

## 1. 方法一：开箱即用

* 我们已经实现了简单的 OpenTelemetry Java Agent 扩展，在 HTTP Response Header 中自动添加“TraceId”和“SpanId”字段，只需要在启动参数加载 JAR 包即可。下载地址：[ot-java-agent-extension-1.28.0.jar](https://github.com/alibabacloud-observability/opentelemetry-best-practice/blob/main/opentelemetry-javaagent-extension/ot-java-agent-extension-1.28.0.jar) 

* 使用方法：
  * 在原有启动参数上添加 otel.javaagent.extensions 参数 `-Dotel.javaagent.extensions=/path/to/opentelemetry-java-agent-extension.jar`

  * 一个完整的启动命令示例：
```
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.extensions=path/to/opentelemetry-java-agent-extension.jar \
     -Dotel.exporter.otlp.headers=Authentication=<token> \
     -Dotel.exporter.otlp.endpoint=<endpoint> \
     -Dotel.metrics.exporter=none 
     -jar your-app.jar
```

## 2. 方法二：自行实现 OpenTelemetry Java Agent 扩展

### 1. 前提条件
* 使用 OpenTelemetry Java Agent
* OpenTelemetry Java Agent 版本 >= 1.24.0

### 2. 编写 OpenTelemetry Java Agent 扩展
1. 新建一个项目
   
2. 在 pom.xml 中添加依赖
* 注意：opentelemetry-javaagent 的版本需要与您使用的 OpenTelemetry Java Agent 版本一致
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

3. 新建 AgentHttpResponseCustomizer 类，实现 HttpServerResponseCustomizer 接口

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

### 3. 构建 OpenTelemetry Java Agent 扩展
* 将程序打包成 JAR 包，构建后存储在 target 目录下
```
mvn clean pacakage
```

### 4. 启动应用时加载 OpenTelemetry Java Agent 扩展
* 在原有启动参数上添加 otel.javaagent.extensions 参数

`-Dotel.javaagent.extensions=/path/to/opentelemetry-java-agent-extension.jar`

* 一个完整的启动命令示例：
```
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.javaagent.extensions=path/to/opentelemetry-java-agent-extension.jar \
     -Dotel.exporter.otlp.headers=Authentication=<token> \
     -Dotel.exporter.otlp.endpoint=<endpoint> \
     -Dotel.metrics.exporter=none 
     -jar yourapp.jar
```


# NodeJS

## 1. 下载 OpenTelemetry Node.js demo（可选）

* demo地址：[node.js-demo](https://github.com/alibabacloud-observability/nodejs-demo)

* 如果您的 NodeJS 应用已经使用 OpenTelemetry 接入，可跳过步骤1，直接参考步骤2修改 NodeJS 应用中的代码。


## 2. 修改 demo 中 HttpInstrumentation 的创建代码

* 创建 HttpInstrumentation 时可以设置 HttpInstrumentationConfig 参数，参数包括 responseHook，该参数允许用户传入一个自定义方法的方法，在响应被处理之前添加自定义内容，例如在 HTTP Header 中添加 TraceId。


```javascript
...

// 要被替换的内容
// registerInstrumentations({
//   tracerProvider: provider,
//   instrumentations: [new HttpInstrumentation(), ExpressInstrumentation],
// });




const httpInstrumentation = new HttpInstrumentation({
  // 添加一个自定义的responseHook
  responseHook: (span, response) => {

    // 从当前上下文中获取traceId, spanId
    const traceId = span.spanContext().traceId;
    const spanId = span.spanContext().spanId;

    // 将traceId, spanId添加到响应标头中
    response.setHeader('TraceId', traceId);
    response.setHeader('SpanId', spanId);
    // 返回响应对象
    return response;
  },
});


registerInstrumentations({
  tracerProvider: provider,
  instrumentations: [httpInstrumentation, ExpressInstrumentation],
});

...
```
