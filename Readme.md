# 🐱 基于 Docker 接入 Cat 用户指南

Cat 支持通过两种方式接入：

1. **[OpenTelemetry (OTel) 接入](#一opentelemetry-接入)**
2. **[Cat Agent 接入](#二cat-agent-接入)**
3. **[功能对比](#三功能对比表)**

两种方式都可以收集应用的监控数据并上报到 Cat 平台，但在功能和接入深度上有一定差异。

---

## 一、OpenTelemetry 接入

### 1. 准备工作
- Docker 镜像中需要包含 OTel Java Agent。
- 确保 Cat 服务已启动，并能被应用容器访问。

### 2. Dockerfile 示例

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app
COPY build/libs/app.jar .
ADD --chmod=644 https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v$OTEL_JAVA_AGENT_VERSION/opentelemetry-javaagent.jar /usr/src/app/opentelemetry-javaagent.jar

# 设置 JAVA_TOOL_OPTIONS，注入 OTel Agent
ENV JAVA_TOOL_OPTIONS="-javaagent:/usr/src/app/opentelemetry-javaagent.jar"

CMD ["java", "-jar", "app.jar"]
```

### 3.compose.yaml
```yaml
  services:
      [serviceName]:
        environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${CAT_HOST}:${Protocol port using 4317 for grpc|4318 for http}
      - OTEL_EXPORTER_OTLP_PROTOCOL={remote call protocol grpc | http}
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES=otel.resource.attributes=service.namespace={上报的domain name},cat.token={租户授权Token}
      - OTEL_LOGS_EXPORTER=otlp 
      - OTEL_SERVICE_NAME=ad
```

### 4. 特点
- 标准化，兼容 OTel 生态（Prometheus、Grafana 等）。
- 无需额外代码改造即可收集常见指标和链路。

---

## 二、Cat Agent 接入

### 1. 准备工作
- Docker 镜像中需要包含 Cat Agent Jar。
- 应用可通过环境变量加载 Agent。

### 2. Dockerfile 示例

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app
COPY build/libs/app.jar .
COPY cat-agent.jar /usr/src/app/cat-agent.jar

# 设置 JAVA_TOOL_OPTIONS，注入 Cat Agent
ENV JAVA_TOOL_OPTIONS="-javaagent:/usr/src/app/cat-agent.jar   -Dotel.resource.attributes=service.namespace={上报的domain name},service.name={服务名},cat.token={租户授权Token},cat.endpoints={云平台地址}"

CMD ["java", "-jar", "app.jar"]
```

### 3. 特点
- 除了 OTel 的基础能力外，还提供 Cat 的 **增强功能**：
    - 更细粒度的埋点和业务事件追踪
    - 应用级别的日志聚合
    - SQL/Cache/RemoteCall 深度分析
    - 更强的链路与业务结合（如消息队列、业务事务）

---

## 三、功能对比表

| 能力维度               | OpenTelemetry 接入        | Cat Agent 接入 |
|--------------------|-------------------------|---------------|
| **链路监控（Tracing）**  | ✅                       | ✅             |
| **指标采集 (Metrics)** | ✅                       | ✅             |
| **日志采集 (Logging)** | ✅                       | ✅ |
| **SQL 调用监控**       | 基于通用插件支持                | ✅ 深度集成    |
| **缓存监控**           | ❌                       | ✅             |
| **消息队列追踪**         | 需额外配置                   | ✅ 内置支持    |
| **生态兼容性**          | ✅ (Prometheus, Grafana) | ⚠️ 以 Cat 为主 |
| **扩展能力**           | 社区标准插件                  | ✅ Cat 特有插件 |

---

## 四、选择建议

- 如果你只需要 **基础的链路追踪和指标采集**，并希望复用已有的 OTel 生态 → **选择 OpenTelemetry**。
- 如果你需要 **更强的业务监控、SQL/缓存/远程调用分析**，以及与 Cat 的深度集成 → **选择 Cat Agent**。  
