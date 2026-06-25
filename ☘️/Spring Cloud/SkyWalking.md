# SkyWalking
> 相关笔记：[[Spring Cloud|微服务 知识总结]]


## 链路追踪

微服务拆分后，一次请求可能经历下面的过程：

1. 用户请求进入 Gateway
2. Gateway 调用商品服务、订单服务或库存服务
3. 订单服务继续调用支付服务
4. 服务访问 MySQL、Redis 或消息队列
5. 部分调用串行执行，部分调用并行执行

这些服务可能由不同团队开发，部署在不同服务器上，甚至使用不同语言。只看单个服务的日志是很难追溯到问题的，比如

- 请求具体经过了哪些服务
- 服务之间是什么依赖关系
- 各个接口的性能怎么样

这就需要链路追踪。<mark class="hltr-green-light">链路追踪会记录一次请求在多个服务之间的流转路径、耗时和状态，把分散在各个服务里的调用信息集中展示。它的核心是通过唯一的 `Trace ID` 串联跨服务调用，用 `Span` 表示一次具体处理单元，再通过 `Context` 传递让链路可以连续下去。</mark>

这里用快递包裹的物流追踪来类比链路追踪。看物流轨迹时，我们会关心包裹经过了哪些转运节点、每个节点花了多少时间、派送过程有没有异常。链路追踪也是类似的思路：给复杂的网络请求画一张“流程图”，记录一次操作经过了哪些服务、每个环节耗时多少、是否出现异常。

![[Pasted image 20260615103705.png|503]]

| 链路追踪     | 快递物流                            |
| -------- | ------------------------------- |
| Trace    | 整个物流链路                          |
| Trace ID | 快递单号                            |
| Span     | 包裹每经过一个转运中心产生的一条记录，包含到达时间、状态等信息 |

链路追踪的作用主要有四个：

- **保障系统可用性**：结合 CPU、内存、成功率等指标，提前发现异常并触发告警
- **优化性能和资源使用**：通过完整 Trace 看清每个服务节点耗时，找到慢调用
- **提升故障排查效率**：用 Trace ID 把跨服务日志、错误信息串成统一视图
- **理解服务依赖与拓扑**：自动展示服务间调用关系、调用频率和成功率，辅助架构分析

<mark class="hltr-orange">链路追踪适合找调用关系和慢点，但不完全等同于日志。</mark> Trace 说明“哪一步慢、哪一步错”，日志继续解释“当时的参数和业务状态是什么”。如果还要定位到具体方法和代码行，再进入性能剖析。

## 常见方案

链路追踪本身是一类问题，不是 SkyWalking 独有的能力。常见方案大概有这些：

- **Zipkin**：Twitter 开源的分布式链路追踪系统，设计思路来自 Google Dapper 论文，常用于请求链路采集、存储和查询
- **CAT**：美团点评开源的分布式实时监控系统，理念来源于 eBay 的 CAL 系统，报表和监控能力比较完整，但通常需要代码埋点
- **SkyWalking**：Apache 顶级项目，面向微服务和分布式架构，支持 Agent 自动采集，UI 功能较完整，也支持插件扩展

简单对比图：

| 维度 | Zipkin | CAT | SkyWalking |
| --- | --- | --- | --- |
| 接入复杂度 | 接入相对简单，常和 Spring Cloud 生态集成 | 需要代码埋点 | 通过 Java Agent 启动，代码无侵入 |
| 数据粒度 | 接口级 | 代码级，可细化到具体代码块 | 方法级，支持 RPC、HTTP |
| 支持语言 | Java、C#、Python、Node.js、Go、Ruby、Scala 等 | Java、C/C++、Python、Node.js、Go 等 | Java、Python、Node.js、PHP、Go、Ruby 等 |
| 调用链可视化 | 支持 | 支持 | 支持 |
| 聚合报表 | 较少 | 很丰富 | 较丰富 |
| 服务依赖图 | 简单 | 简单 | 较好 |
| 告警支持 | 不直接提供 | 支持 | 支持 |
| 存储机制 | 内存、Elasticsearch、MySQL 等 | MySQL、本地文件、HDFS 等 | Elasticsearch、MySQL、BanyanDB、PostgreSQL 等 |
| 社区情况 | 文档丰富，功能迭代相对慢 | 国内使用多，社区活跃度一般 | 社区活跃，更新较频繁 |
| 主要优势 | 部署简单，适合快速接入基础 Trace | 报表完整，适合综合监控 | 无侵入、Apache 项目、生态较完整 |
| 主要不足 | 报表能力弱，功能相对单一 | 代码侵入性高 | 插件开发门槛较高 |



## 选择 SkyWalking 的原因

综合来看，SkyWalking 的能力更完整，无代码侵入接入、多语言、报表、告警和插件体系都比较适合当前 Java Spring 微服务项目。


## 核心组件

 SkyWalking 架构分为三类组件：探针 Agent、后端 Backend 和前端 UI。<mark class="hltr-green-light">其中 OAP 是后端的核心部分，负责接收、处理和存储来自探针的数据，并生成聚合指标。</mark>

| 组件    | 作用                         | 比喻（交通网系统）        |
| ----- | -------------------------- | ---------------- |
| Agent | 探针，通过字节码增强收集应用运行时数据        | 摄像头              |
| OAP   | 接收、分析、聚合数据，并提供查询和告警能力      | 交通数据处理系统（后端）     |
| UI    | 展示拓扑、服务指标、Trace、日志、剖析任务和告警 | Dashboard 看板（前端） |

## 关键术语

| 术语                  | 含义                                                                     |
| ------------------- | ---------------------------------------------------------------------- |
| APM                 | Application Performance Monitor，应用性能监控，通过收集和分析响应时间、吞吐量、错误率等数据来帮助优化系统性能 |
| OAP                 | Observability Analysis Platform，观测分析平台，是 SkyWalking 后端的核心部分            |
| Agent               | 探针，通过字节码增强无侵入采集应用运行时数据                                                 |
| Metrics             | 指标，比如服务响应时间、服务成功率、分钟请求数                                                |
| Trace               | 一次完整的分布式请求，由多个 Span 或 Segment 组成                                       |
| Trace ID            | 一次 Trace 的全局标识，常用于 Trace 与日志互查                                         |
| Span                | 一次具体操作，比如接收 HTTP 请求、执行方法、调用数据库                                         |
| Endpoint            | 服务入口，比如 `GET:/orders/{id}` 或一个 RPC 方法                                  |
| Context Propagation | 把 Trace 上下文传到下游服务或异步任务                                                 |
| Sampling            | 只采集部分请求，控制 Agent、网络和存储开销                                               |
| Topology            | 根据调用数据得到的服务依赖关系                                                        |
| Apdex               | 根据响应时间评估用户体验的指标                                                        |
| OAL                 | OAP 中用于分析 Trace 和 Mesh 流量的规则语言                                         |
| MQE                 | Metrics Query Expression，当前告警规则使用的指标表达式                                |

## 安装部署

### 下载包说明

课件里是手动下载安装 SkyWalking，需下载两个包：

- **SkyWalking APM**：服务端包，里面包含 OAP、UI、配置文件和启动脚本，用来启动 SkyWalking 后端与可视化界面
- **Java Agent**：探针包，用来接入 Spring Boot 服务，通过 `-javaagent` 挂到业务 JVM 上采集 Trace、Metrics 等数据

从 `8.8.0` 之后，APM 和 Agent 分开安装，所以只下载 Java Agent 不够。需要先准备 SkyWalking APM 服务端，再给业务服务接入 Java Agent。

- APM 历史版本：`https://archive.apache.org/dist/skywalking`
- Java Agent 历史版本：`https://archive.apache.org/dist/skywalking/java-agent`

### Docker Compose 部署

目录可以这样放：

```text
skywalking/
├── compose.yaml
└── config/
    └── alarm-settings.yml
```

`compose.yaml`：

```yaml
services:
  banyandb:
    image: apache/skywalking-banyandb:0.10.2
    container_name: skywalking-banyandb
    command:
      - standalone
      - --stream-root-path
      - /data/stream-data
      - --measure-root-path
      - /data/measure-data
    ports:
      - "17912:17912"
      - "17913:17913"
    volumes:
      - banyandb-data:/data
    healthcheck:
      test: ["CMD", "sh", "-c", "nc -nz 127.0.0.1 17912"]
      interval: 5s
      timeout: 5s
      retries: 60

  oap:
    image: apache/skywalking-oap-server:10.4.0
    container_name: skywalking-oap
    depends_on:
      banyandb:
        condition: service_healthy
    environment:
      SW_STORAGE: banyandb
      SW_STORAGE_BANYANDB_TARGETS: banyandb:17912
      SW_HEALTH_CHECKER: default
      JAVA_OPTS: "-Xms1g -Xmx1g"
    ports:
      - "11800:11800"
      - "12800:12800"
    volumes:
      - ./config/alarm-settings.yml:/skywalking/ext-config/alarm-settings.yml:ro
    healthcheck:
      test: ["CMD-SHELL", "bash -c 'echo >/dev/tcp/127.0.0.1/12800'"]
      interval: 15s
      timeout: 5s
      retries: 20
      start_period: 60s

  ui:
    image: apache/skywalking-ui:10.4.0
    container_name: skywalking-ui
    depends_on:
      oap:
        condition: service_healthy
    environment:
      SW_OAP_ADDRESS: http://oap:12800
    ports:
      - "8080:8080"

volumes:
  banyandb-data:
```

先创建一个可用的 `config/alarm-settings.yml`：

```yaml
rules: {}
hooks: {}
```

启动：

```bash
cd skywalking
docker compose up -d
docker compose ps
```

检查 OAP：

```bash
curl http://localhost:12800/internal/l7check
```

如果 Spring Boot 运行在宿主机，Agent 连接 `127.0.0.1:11800`。如果应用也在 Compose 中，地址应写成 `oap:11800`，不能写 `localhost:11800`，因为容器内的 `localhost` 指向应用容器自己。

## Spring Boot 服务接入

接入 SkyWalking 前先确认两件事：

1. SkyWalking OAP 和 UI 已经启动，能访问 `http://127.0.0.1:8080`，如果改过 UI 端口就换成自己的端口
2. 微服务本身能正常跑通业务请求，否则后面 UI 上即使没有 Trace，也不好判断是业务问题还是 Agent 接入问题

### 下载与放置 Java Agent

从 SkyWalking 官方 Downloads 页面下载 Java Agent 后，一般先解压 SkyWalking APM 包，再在 APM 目录中新建 `agent` 文件夹，把 Java Agent 解压后的内容复制进去。这样后面启动业务服务时，就可以指向这个 `agent/skywalking-agent.jar`，也比较方便。

### 启动参数接入

以 `order-service` 为例，通过 `-javaagent` 参数配置 SkyWalking Agent 来追踪微服务。
最直接的启动方式是在 Java 启动参数里增加：
![[Pasted image 20260615104956.png|593]]
```bash
java \
  -javaagent:/opt/skywalking-agent/skywalking-agent.jar \   #本地的agent安装路径
  -Dskywalking.agent.service_name=order-service \
  -Dskywalking.collector.backend_service=127.0.0.1:11800 \
  -jar order-service.jar
```

`-javaagent` 必须放在 `-jar` 前面。

几个参数含义：

- `-javaagent`：后面换成自己本地的 `skywalking-agent.jar` 路径
- `-Dskywalking.agent.service_name`：服务名称，比如 `order-service`
- `-Dskywalking.collector.backend_service`：SkyWalking OAP 的 gRPC 地址，默认是 `127.0.0.1:11800`（需开放端口号）

启动完成后，打开 SkyWalking UI 左侧的服务页面，就能看到已经接入的服务。其他服务也按同样方式加 Agent 参数，只是 `service_name` 要改成各自的服务名。

也可以通过环境变量配置：

```bash
export SW_AGENT_NAME=order-service
export SW_AGENT_INSTANCE_NAME=order-service-local-01
export SW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800

java -javaagent:/opt/skywalking-agent/skywalking-agent.jar \
  -jar order-service.jar
```

常用配置：

| Agent 配置 | 环境变量 | 说明 |
| --- | --- | --- |
| `agent.service_name` | `SW_AGENT_NAME` | 服务名，同一业务服务的多个实例保持一致 |
| `agent.instance_name` | `SW_AGENT_INSTANCE_NAME` | 实例名，同一服务内保持唯一 |
| `agent.namespace` | `SW_AGENT_NAMESPACE` | 可用于区分环境或命名空间 |
| `agent.cluster` | `SW_AGENT_CLUSTER` | 标记数据中心或集群 |
| `collector.backend_service` | `SW_AGENT_COLLECTOR_BACKEND_SERVICES` | OAP gRPC 地址 |

服务名建议使用明确且固定的英文名，比如 `gateway-service`、`order-service`、`inventory-service`。不要把 Pod IP 或随机端口放到服务名里，这类信息应该放在实例名中。

### Docker 中接入

Spring Boot 镜像可以把 Agent 放进镜像：

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY agent /opt/skywalking-agent
COPY target/order-service.jar /app/app.jar

ENTRYPOINT ["java",
  "-javaagent:/opt/skywalking-agent/skywalking-agent.jar",
  "-jar",
  "/app/app.jar"]
```

应用服务配置：

```yaml
order-service:
  image: example/order-service:1.0.0
  environment:
    SW_AGENT_NAME: order-service
    SW_AGENT_INSTANCE_NAME: order-service-01
    SW_AGENT_COLLECTOR_BACKEND_SERVICES: oap:11800
  depends_on:
    oap:
      condition: service_healthy
```

如果同一个镜像启动多个副本，不要把 `SW_AGENT_INSTANCE_NAME` 写死。可以不配置，让 Agent 生成实例名，或者由 Kubernetes 使用 Pod 名注入。

## 自定义追踪

自动插件主要覆盖框架入口和常见中间件。纯业务方法、自研协议、特殊线程模型可能不会自动出现，这时可以使用 Java Agent Toolkit。

### Toolkit 依赖

```xml
<properties>
    <skywalking.version>9.6.0</skywalking.version>
</properties>

<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>${skywalking.version}</version>
</dependency>
```

Toolkit 版本建议和 Java Agent 版本保持一致。

### 相关注解

`@Trace` 会把方法加入链路追踪，自动生成 Span 并记录方法耗时和调用关系。`operationName` 可以自定义 Span 名称，方便在 Trace 页面里阅读。`@Tag` 顾名思义打上标签，可以把参数或返回值写入当前入口，方便辨认

`@Tag` 里的 `key` 是标签名，`value` 是取值表达式。
```java
import org.apache.skywalking.apm.toolkit.trace.Trace;
import org.apache.skywalking.apm.toolkit.trace.Tag;

@Trace(operationName = "OrderService/createOrder")
@Tag(key = "order.userId", value = "arg[0]")
@Tag(key = "order.id", value = "returnedObj.id")
public Order createOrder(Long userId, CreateOrderCommand command) {
    return orderRepository.save(userId, command);
}
```

如果要同时记录多个信息，可以使用 `@Tags`：

```java
import org.apache.skywalking.apm.toolkit.trace.Tag;
import org.apache.skywalking.apm.toolkit.trace.Tags;
import org.apache.skywalking.apm.toolkit.trace.Trace;

@Trace(operationName = "queryById")
@Tags({
    @Tag(key = "request", value = "arg[0]"),
    @Tag(key = "response", value = "returnedObj")
})
public OrderInfo queryById(Integer orderId) {
    return orderMapper.selectById(orderId);
}
```

`arg[n]` 表示第 `n` 个入参，从 `0` 开始；`returnedObj` 表示方法返回值，`returnedObj.id` 表示取返回对象里的 `id` 字段。

## 日志关联与上传

要想在 SkyWalking UI 里查看日志，并把日志和 Trace 关联起来，需要分成两件事：

1. 在本地日志中打印 Trace ID，方便从日志跳到 Trace
2. 通过 gRPC 把日志上传到 OAP，在 SkyWalking UI 中统一查询

### 添加依赖

Spring Boot 默认常用 Logback：

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>9.6.0</version>
</dependency>
```

### 编写 xml 文件

在 `src/main/resources` 下新建 `logback-spring.xml`。Spring Boot 会优先加载 `logback-spring.xml`，它比普通 `logback.xml` 更适合 Spring Boot 项目，也能使用 Spring 的扩展能力。

下面这个配置只负责把 Trace ID 打到本地控制台日志里：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
                <Pattern>
                    %d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} - %msg%n
                </Pattern>
            </layout>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

有 Agent 且当前线程处于 Trace 中时，`%X{tid}` 会输出 Trace ID；没有可用 Trace 时会显示 `TID: N/A`。

如果希望写出完整 SkyWalking 上下文，可以把 `%X{tid}` 换成 `%X{sw_ctx}`，内容包括：

```text
serviceName, instanceName, traceId, traceSegmentId, spanId
```

### 上传日志

只打印 Trace ID 还不等于上传日志。上面的配置只能让应用自己的控制台日志带上 Trace ID，SkyWalking UI 的 Log 页面还看不到这些日志。

如果要把日志上传到 OAP，需要在 `logback-spring.xml` 里添加 `GRPCLogClientAppender`：

```xml
<appender name="GRPC_LOG"
          class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
    <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
        <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
            <Pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} - %msg%n
            </Pattern>
        </layout>
    </encoder>
</appender>

<root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="GRPC_LOG"/>
</root>
```

这里要注意：`GRPCLogClientAppender` 上报日志时，会自动附带 Trace ID、Segment ID 和 Span ID。也就是说，layout 控制的是日志正文格式；日志与 Trace 的关联信息由 gRPC reporter 自动带上。
等重启服务并发起请求后，就可以在 SkyWalking UI 的 Log 页面查询日志，也可以根据 Trace ID 过滤日志并跳转到对应 Trace。


## 告警
SkyWalking OAP 的告警规则位于 apm 的 `config/alarm-settings.yml`。Docker 镜像可以把同名文件挂载到 `/skywalking/ext-config` 来覆盖默认配置。

可以把告警类比成烟雾报警器：当系统出现响应慢、成功率低这类异常时，就像检测到烟雾一样触发通知，提醒研发或运维尽快处理。

SkyWalking 默认规则主要覆盖这些场景：

1. 服务平均响应时间超过阈值
2. 服务成功率低于阈值
3. 服务响应时间百分位数超过阈值，比如 P50、P75、P90、P95、P99
4. 服务实例平均响应时间超过阈值
5. Endpoint 平均响应时间超过阈值
6. 数据库访问平均响应时间超过阈值
7. Endpoint 关系平均响应时间超过阈值

### 规则字段

简单罗列一些规则字段，用到的时候再查即可。

| 字段 | 说明 |
| --- | --- |
| 规则 ID | 必须以 `_rule` 结尾 |
| `expression` | MQE 表达式，结果必须是单值布尔结果 |
| `period` | 参与判断的分钟窗口 |
| `silence-period` | 告警触发后的静默时间 |
| `recovery-observation-period` | 条件恢复后连续观察多少个周期再发送恢复通知 |
| `include-names` | 只匹配指定实体 |
| `exclude-names` | 排除指定实体 |
| `include-names-regex` | 用正则匹配实体名 |
| `tags` | 附加标签，常用 `level` 标记告警级别 |
| `hooks` | 指定本规则调用哪些通知目标 |
| `message` | 告警文本，`{name}` 会替换成实体名 |

### Webhook

Webhook 是 OAP 把告警推送给外部系统的一种方式。告警触发后，OAP 会使用 `HTTP POST` 和 `application/json` 请求目标地址，请求体是告警消息数组，在编写代码时需要注意用 List<>~

接收方只要提供一个可访问的 HTTP 地址，就可以接收 SkyWalking 推送的告警数据。常见用途包括推送到钉钉、企业微信、飞书、邮箱，或者转给内部告警服务继续处理。

### 配置

完整的 `alarm-settings.yml` 可以这样写：

```yaml
rules:
  service_resp_time_rule:
    expression: sum(service_resp_time > 1000) >= 3
    period: 5
    silence-period: 10
    recovery-observation-period: 2
    message: "服务 {name} 最近 5 分钟内多次超过 1000ms"
    tags:
      level: WARNING
    hooks:
      - webhook.ops

hooks:
  webhook:
    ops:
      urls:
        - http://host.docker.internal:8088/webhooks/skywalking
      recovery-urls:
        - http://host.docker.internal:8088/webhooks/skywalking
      headers:
        Authorization: Bearer replace-with-your-token
```


## 排查顺序

实际遇到问题需要排查时可以按这个顺序：

1. 先看服务拓扑，确认调用方向和异常节点
2. 再看 Service、Instance、Endpoint 指标，缩小时间范围
3. 打开慢请求 Trace，查看哪个 Span 占用时间最多
4. 用 Trace ID 查询关联日志，检查参数、异常和业务状态
5. 接口内部仍看不出原因时，创建 Trace Profiling 任务
6. JVM 出现 CPU、分配或锁问题时，创建 Java App Profiling 任务
7. 确认指标规则后配置告警，再接 Webhook 通知
