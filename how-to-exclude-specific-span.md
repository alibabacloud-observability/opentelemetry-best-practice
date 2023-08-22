# 如何使用 OpenTelemetry 过滤指定的Span

## Java

OpenTelemetry Samplers 可用来过滤 Span。如果不希望修改业务代码，或者已经使用了 OpenTelemetry Java Agent 自动上报应用数据，那么可以参考方法一对 OpenTelemetry Java Agent 的功能进行扩展，编写自定义的 OpenTelemetry Sampler，通过在 Sampler 中定义过滤规则，实现无侵入的 Span 过滤。如果您已经使用了 OpenTelemetry Java SDK 手动上报应用数据，可以参考方法二在业务代码中编写自定义 Sampler。

### 1. 方法一：使用 OpenTelemetry Java Agent Extension（自动埋点 / 无侵入式）

#### 1.1 前提条件

*   已经使用 OpenTelemetry Java Agent 为应用程序自动埋点。如果您还未使用过 OpenTelemetry Java Agent，请参考 [通过OpenTelemetry Java Agent 自动上报Java应用数据](https://help.aliyun.com/document_detail/410559.html?spm=a2c4g.196640.0.0.22ba3e06dgKrgG#section-746-yt6-64f)。
    

#### 1.2 新建项目

*   新建一个空白的 maven 项目，用于创建和构建 Agent Extension，对 Agent 的功能进行扩展。
    

#### 1.3 在 pom.xml 中添加依赖

*   注意：为保证依赖互相兼容，请将各项 opentelemetry 依赖设置为同一版本，且与您使用的 OpenTelemetry Java Agent 版本保持一致。
    

```xml
    <dependency>
        <groupId>com.google.auto.service</groupId>
        <artifactId>auto-service</artifactId>
        <version>1.1.1</version>
    </dependency>
    
    <dependency>
        <groupId>io.opentelemetry.javaagent</groupId>
        <artifactId>opentelemetry-javaagent</artifactId>
        <version>1.28.0</version>
        <!--这里要设置为compile的-->
        <scope>compile</scope>
    </dependency>
    
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk-trace</artifactId>
        <version>1.28.0</version>
    </dependency>
    
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk-extension-autoconfigure</artifactId>
        <version>1.28.0</version>
    </dependency>
    
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-semconv</artifactId>
        <version>1.28.0-alpha</version>
    </dependency>
```

#### 1.4 新建 SpanFilterSampler 类（类名可自定义）

*   需要实现 io.opentelemetry.sdk.trace.samplers.Sampler接口，并实现 shouldSample 方法和 getDescription 方法。
    

*   **shouldSample 方法**
    
    *   可以在方法中自定义过滤条件。对于要过滤的 Span，返回 `SamplingResult.create(SamplingDecision.DROP)`，需要保留并继续上报的 Span 则返回 `SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE)`。
        
*   **getDescription 方法**
    
    *   返回自定义 Sampler 的名称。
        

*   在下面示例代码中，SpanFilterSampler 会过滤名称为“spanName1”或“spanName2”的Span，以及 attributes.http.target 为“/api/checkHealth”或“"/health/checks"”的所有 Span。
    
```java
    package org.example;
    
    import io.opentelemetry.api.common.Attributes;
    import io.opentelemetry.api.trace.SpanKind;
    import io.opentelemetry.context.Context;
    import io.opentelemetry.sdk.trace.data.LinkData;
    import io.opentelemetry.sdk.trace.samplers.Sampler;
    import io.opentelemetry.sdk.trace.samplers.SamplingDecision;
    import io.opentelemetry.sdk.trace.samplers.SamplingResult;
    import io.opentelemetry.semconv.trace.attributes.SemanticAttributes;
    
    import java.util.*;
    
    public class SpanFilterSampler implements Sampler {
        /*
         * 过滤 Span名称 在 EXCLUDED_SPAN_NAMES 中的所有 Span
         */
        private static List<String> EXCLUDED_SPAN_NAMES = Collections.unmodifiableList(
                Arrays.asList("spanName1", "spanName2")
        );
    
        /*
         * 过滤 attributes.http.target 在 EXCLUDED_HTTP_REQUEST_TARGETS 中的所有 Span
         */
        private static List<String> EXCLUDED_HTTP_REQUEST_TARGETS = Collections.unmodifiableList(
                Arrays.asList("/api/checkHealth", "/health/checks")
        );
    
        @Override
        public SamplingResult shouldSample(Context parentContext, String traceId, String name, SpanKind spanKind, Attributes attributes, List<LinkData> parentLinks) {
            String httpTarget = attributes.get(SemanticAttributes.HTTP_TARGET) != null ? attributes.get(SemanticAttributes.HTTP_TARGET) : "";
    
            if (EXCLUDED_SPAN_NAMES.contains(name) || EXCLUDED_HTTP_REQUEST_TARGETS.contains(httpTarget)) { // 根据条件进行过滤
               return SamplingResult.create(SamplingDecision.DROP);
           } else {
               return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
           }
        }
    
        @Override
        public String getDescription() {
            return "SpanFilterSampler"; // SpanFilterSampler可以替换为自定义的名称
        }
    }
```

#### 1.5 新建 SpanFilterSamplerProvider 类（类名可自定义）

*   需要实现io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSamplerProvider接口，实现 createSampler 方法和 getName 方法
    

*   **createSampler 方法**
    
    *   创建并返回您自定义的 Sampler 实例。
        
*   **getName 方法**
    
    *   获取自定义的 Sampler 名称，OpenTelemetry Java Agent 通过该名称找到这个 Sampler。
        
```java
    package org.example;
    
    import com.google.auto.service.AutoService;
    import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
    import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSamplerProvider;
    import io.opentelemetry.sdk.trace.samplers.Sampler;
    
    @AutoService(ConfigurableSamplerProvider.class)
    public class SpanFilterSamplerProvider implements ConfigurableSamplerProvider {
        @Override
        public Sampler createSampler(ConfigProperties configProperties) {
            return new SpanFilterSampler();
        }
    
        @Override
        public String getName() {
            return "SpanFilterSampler"; // SpanFilterSampler可以替换为自定义的名称
        }
    }
```  

#### 1.6 构建

*   将程序打包成 JAR 包，构建后存储在 target 目录下
    
```
mvn clean pacakage
```
    

#### 1.7 启动应用时加载 OpenTelemetry Java Agent 扩展

有两种加载方式：

*   方法一：在原有 VM 参数上添加 otel.traces.sampler 参数
```  
    -Dotel.traces.sampler=<your-sampler-name> // 将 <your-sampler-name> 替换为您自定义的 Sampler 名称，也就是 getName 方法的返回值
```

*   完整启动命令示例：
    
```
    java -javaagent:path/to/opentelemetry-javaagent.jar \
         -Dotel.javaagent.extensions=path/to/opentelemetry-java-agent-extension.jar \
         -Dotel.traces.sampler=<your-sampler-name> \ 
         -Dotel.exporter.otlp.headers=Authentication=<token> \
         -Dotel.exporter.otlp.endpoint=<endpoint> \
         -Dotel.metrics.exporter=none 
         -jar yourapp.jar
```

*   方法二：设置 OTEL\_TRACES\_SAMPLER 环境变量
```
    export OTEL_TRACES_SAMPLER="<your-sampler-name>" // 将 <your-sampler-name> 替换为您自定义的 Sampler 名称，也就是 getName 方法的返回值
```

*   完整启动命令示例：
    
```
    export OTEL_JAVAAGENT_EXTENSIONS="path/to/opentelemetry-java-agent-extension.jar"
    export OTEL_TRACES_SAMPLER="<your-sampler-name>"
    export OTEL_EXPORTER_OTLP_HEADERS="Authentication=<token>"
    export OTEL_EXPORTER_OTLP_ENDPOINT="<endpoint>"
    export OTEL_METRICS_EXPORTER="none"
    
    java -javaagent:path/to/opentelemetry-javaagent.jar \
         -jar yourapp.jar
```

### 2. 方法二：使用 OpenTelemetry Java SDK （手动埋点）

#### 2.1 前提条件

*   已经使用 OpenTelemetry Java SDK 为应用程序手动埋点。如果您还未使用过 OpenTelemetry Java SDK，请参考 [通过OpenTelemetry Java SDK 手动埋点并上报Java应用数据](https://help.aliyun.com/document_detail/410559.html#section-hvi-cqt-mpa)
    

#### 2.2 在您的业务代码中创建自定义 Sampler 类

*   需要实现 io.opentelemetry.sdk.trace.samplers.Sampler 接口，并实现 shouldSample 方法和 getDescription 方法。
    

*   **shouldSample 方法**
    
    *   可以在方法中自定义过滤条件。对于要过滤的 Span，返回 `SamplingResult.create(SamplingDecision.DROP)`，需要保留并继续上报的 Span 则返回 `SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE)`。
        
*   **getDescription 方法**
    
    *   返回自定义 Sampler 的名称。
        

*   在下面示例代码中，SpanFilterSampler 会过滤名称为“spanName1”或“spanName2”的Span，以及 attributes.http.target 为“/api/checkHealth”或“"/health/checks"”的所有 Span。
    
```java
    package org.example;
    
    import io.opentelemetry.api.common.Attributes;
    import io.opentelemetry.api.trace.SpanKind;
    import io.opentelemetry.context.Context;
    import io.opentelemetry.sdk.trace.data.LinkData;
    import io.opentelemetry.sdk.trace.samplers.Sampler;
    import io.opentelemetry.sdk.trace.samplers.SamplingDecision;
    import io.opentelemetry.sdk.trace.samplers.SamplingResult;
    import io.opentelemetry.semconv.trace.attributes.SemanticAttributes;
    
    import java.util.*;
    
    public class SpanFilterSampler implements Sampler {
        /*
         * 过滤 Span名称 在 EXCLUDED_SPAN_NAMES 中的所有 Span
         */
        private static List<String> EXCLUDED_SPAN_NAMES = Collections.unmodifiableList(
                Arrays.asList("spanName1", "spanName2")
        );
    
        /*
         * 过滤 attributes.http.target 在 EXCLUDED_HTTP_REQUEST_TARGETS 中的所有 Span
         */
        private static List<String> EXCLUDED_HTTP_REQUEST_TARGETS = Collections.unmodifiableList(
                Arrays.asList("/api/checkHealth", "/health/checks")
        );
    
        @Override
        public SamplingResult shouldSample(Context parentContext, String traceId, String name, SpanKind spanKind, Attributes attributes, List<LinkData> parentLinks) {
            String httpTarget = attributes.get(SemanticAttributes.HTTP_TARGET) != null ? attributes.get(SemanticAttributes.HTTP_TARGET) : "";
    
            if (EXCLUDED_SPAN_NAMES.contains(name) || EXCLUDED_HTTP_REQUEST_TARGETS.contains(httpTarget)) { // 根据条件进行过滤
                return SamplingResult.create(SamplingDecision.DROP);
            } else {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }
        }
    
        @Override
        public String getDescription() {
            return "SpanFilterSampler"; // SpanFilterSampler可以替换为自定义的名称
        }
    }
```    
    

#### 2.3 在创建 SdkTracerProvider 实例时设置自定义 Sampler

*   在创建 SdkTracerProvider 实例时，调用 `setSampler(new SpanFilterSampler())`，即可完成配置。
    
```java
    ...
    
    
    SdkTracerProvider sdkTracerProvider = SdkTracerProvider.builder()
                    .setSampler(new MySampler()) // 添加这一行
                    .addSpanProcessor(BatchSpanProcessor.builder(OtlpGrpcSpanExporter.builder()
                            .setEndpoint("<endpoint>")
                            .addHeader("Authentication", "<token>")
                            .build()).build())
                    .setResource(resource)
                    .build();
    
    
    ...

```

#### 2.4   启动应用

## NodeJS

### 前提条件

*   已经使用 OpenTelemetry JavaScript API 为应用程序埋点。如果您还未使用过 OpenTelemetry JavaScript API，请参考 [通过OpenTelemetry上报Node.js应用数据](https://help.aliyun.com/document_detail/425195.html#section-z7s-fqo-y8m) 和示例代码 [opentelemetry-nodejs-demo](https://github.com/alibabacloud-observability/nodejs-demo)。
    

### 1. 方法一：在 Span 创建阶段过滤

#### 1.1 创建 HttpInstrumentation 时设置 ignoreIncomingRequestHook

*   创建 HttpInstrumentation 时可以设置 HttpInstrumentationConfig 参数，参数包括 ignoreIncomingRequestHook，该参数允许用户传入一个自定义方法的方法，在请求被处理前判断是否需要创建埋点。**需要注意的是，此方法的作用只是不创建 Span，而不会影响该请求的执行。**
    
*   下面代码中展示了如何过滤 request.url=/api/checkHeal 的请求：
    
```javascript
    ...
    
    // 要被替换的内容
    // registerInstrumentations({
    //   tracerProvider: provider,
    //   instrumentations: [new HttpInstrumentation(), ExpressInstrumentation],
    // });
    
    const httpInstrumentation = new HttpInstrumentation({
    
      // 添加一个自定义的ignoreIncomingRequestHook
      ignoreIncomingRequestHook: (request) => {
        //忽略request.url为/api/checkHeal的请求
        if (request.url === '/api/checkHealth') {
            return true;
        }
        return false;
      },
      
    });
    
    registerInstrumentations({
      tracerProvider: provider,
      instrumentations: [httpInstrumentation, ExpressInstrumentation],
    });
    
    ...
```

#### 1.2 启动应用

### 2. 方法二：在 Span 上报阶段过滤

#### 2.1 创建自定义Sampler类

*   创建一个实现了Sampler接口的自定义Sampler类。该接口定义了是否采样的规则。例如：
    
```javascript
    const opentelemetry = require('@opentelemetry/api');
    
    class SpanFilterSampler {
      shouldSample(spanContext, parentContext) {
        // 在此处实现您的自定义采样逻辑
      }
    }

```

#### 2.2 创建 NodeTracerProvider 实例时设置自定义 Sampler

```javascript
    ...
    const provider = new NodeTracerProvider({
      sampler: new SpanFilterSampler(), // 添加这一行代码，设置自定义 Sampler
      resource: new Resource({
        [SemanticResourceAttributes.HOST_NAME]: require("os").hostname(), // your host name
        [SemanticResourceAttributes.SERVICE_NAME]: "<your-service-name>",
      }),
    });
    ...
```