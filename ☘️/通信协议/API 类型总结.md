# API 类型总结

> 相关笔记：[[通信协议|通信协议 知识总结]]、[[流式传输|流式传输]]

视频来源：[Every Type Of API Explained in 18 Minutes.](https://www.youtube.com/watch?v=VQyPVCT2Kl8)

这篇笔记主要将 API 类型到底是怎么联系起来，它们分别是在不同通信需求下长出来的方案：有的解决“怎么请求数据”，有的解决“服务之间怎么快一点”，有的解决“不要一直轮询”，还有的解决“事件太多以后怎么管理”。

先放一张关系图，后面再逐个展开。

![API 类型之间的关系|334](通信协议/assets/api-types/api-relationship-map.jpg)

## 整体脉络

- 最早我们只是想让客户端和服务端通信：页面要用户信息，服务端返回用户信息。这时 REST 就够用了，它把数据抽象成资源，用 `GET`、`POST`、`PUT`、`DELETE` 这些 HTTP 方法去操作。企业系统里，光“能通信”还不够。银行、保险、CRM 这类系统更在意契约、事务、安全和错误处理，所以 SOAP 这种更正式、更重的协议就有存在空间。

- 后面系统拆成微服务以后，问题又变了。服务之间要互相调用，而且调用次数很多、延迟敏感，这时 RPC / gRPC 这条线就出现了。它们不再把重点放在“资源 URL”，而是更像“调用远程函数”。

- 再往前端看，移动端和复杂页面经常只需要一小部分字段。REST 有时会返回太多数据，有时又要请求好几次，GraphQL 就是为这种“我只要这些字段”的需求出现的。

- 接着是实时场景。传统 HTTP 是客户端主动问服务端，适合普通请求，但不适合“有变化马上告诉我”。所以 Webhook、SSE、WebSocket 开始出现：Webhook 适合第三方系统回调你，SSE 适合服务端单向推送，WebSocket 适合双方都要实时说话。如果实时内容从文字消息变成音视频，就轮到 WebRTC。它解决的是浏览器和客户端之间的实时音视频、屏幕共享、点对点通信。

- 再往后，系统越来越多，很多业务不适合同步等待。订单创建后，库存、通知、风控、数据分析都要动起来，但订单服务不应该一个个等它们处理完。这就是 Event-Driven API、MQTT、AMQP、Kafka 这条线：用消息和事件把系统解耦。

- 最后还有两个偏“规范 / 连接层”的东西。AsyncAPI 不是一个运行时协议，它更像异步系统里的 OpenAPI，用来说明有哪些事件、谁生产、谁消费。MCP 则是 AI 时代出现的新接口规范，解决模型接外部工具和数据源时重复集成的问题。

## 请求响应：REST 和 SOAP

### REST API

![REST API|298](通信协议/assets/api-types/0000-rest-api.jpg)

REST 可以理解成最常见的 Web API 形态。客户端发一个 HTTP 请求，服务端返回一份数据。视频里用了“餐厅服务员”的类比：你告诉服务员要什么，服务员去厨房拿，再把结果带回来。对应到系统里，就是客户端请求服务端，服务端返回 JSON。

REST 的核心是资源。比如用户是资源，订单是资源，文章也是资源。这样做的好处是简单、通用、跨平台。Web 前端、移动端、小程序、服务端脚本，基本都能很容易调用 REST API。
```text
GET    /users/1
POST   /orders
PUT    /users/1
DELETE /items/9
```

实际公司场景里，[GitHub REST API](https://docs.github.com/en/rest) 就是非常典型的例子。你可以通过 REST API 获取仓库、Issue、Pull Request、用户信息，也可以做自动化脚本和第三方集成。

缺点：REST 适合普通请求响应，但它不是所有场景的答案。如果页面一次要聚合很多资源，一次调用可能不能返回全部所需要的，会请求很多次；如果服务之间要高频调用，JSON + HTTP 的开销也可能偏大；如果需要实时推送，REST 本身也不擅长。

### SOAP API

![SOAP API|296](通信协议/assets/api-types/0091-soap-api.jpg)

SOAP 是更早、更正式的一套通信协议。它用 XML 包一层固定结构，里面会有 envelope、header、body 这些部分。视频里把 REST 比成比较随意的电话沟通，把 SOAP 比成正式合同，这个类比挺准确。它更稳定、规范、强契约。它可以配合 WSDL 描述服务能力，也有一整套错误处理、安全、事务相关的规范。

所以 SOAP 现在看起来没那么“新”，但在企业系统里仍然很常见。比如 [Salesforce SOAP API](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_quickstart_intro.htm) 主要用于企业系统和 Salesforce 之间做数据集成，[Microsoft Exchange Web Services](https://learn.microsoft.com/en-us/exchange/client-developer/exchange-web-services/explore-the-ews-managed-api-ews-and-web-services-in-exchange) 这类历史系统也长期走 SOAP / XML 这一类接口风格。

这里可以这么记：REST 更适合互联网应用的通用接口，SOAP 更适合企业系统里需要严格契约的集成。

## 服务间调用：RPC 和 gRPC

### RPC

![RPC|289](通信协议/assets/api-types/0676-rpc.jpg)

RPC 的意思是 Remote Procedure Call，也就是远程过程调用。它想解决的问题很直接：服务拆开以后，我不想每次都手写 HTTP 请求、解析 JSON、处理各种细节，我希望像调用本地函数一样调用远程服务。

比如订单服务要校验用户是否登录，如果用 RPC 的思路，就可以像这样理解：

```text
authService.verifyUser(userId)
```

实际执行时这个函数不在本地，它会通过网络请求到认证服务。现实里，RPC 有很多形态。比如以太坊生态里很多节点都暴露 [JSON-RPC API](https://ethereum.org/en/developers/docs/apis/json-rpc/)，钱包、区块浏览器、脚本工具会通过它查询区块、交易、账户状态。Cloudflare Workers 也有 [Workers RPC](https://developers.cloudflare.com/workers/runtime-apis/rpc/) 这种更接近方法调用的接口模型。

RPC 的好处是调用表达简单，适合内部服务。需要注意的是，远程调用毕竟还是网络调用，不能真的当成本地函数一样随便调用。网络延迟、超时、重试、幂等，这些都要考虑。

### gRPC API

![gRPC API|325](通信协议/assets/api-types/0171-grpc-api.jpg)

gRPC 可以看成现代版本的 RPC。它用 Protocol Buffers 定义数据结构，用 HTTP/2 做传输，所以比传统 JSON + HTTP 更适合高频服务间通信。

它解决的需求主要有几个：
- 服务拆多以后，内部调用很多，希望序列化和传输更快
- 多语言服务之间需要统一接口定义
- 有些接口不只是请求响应，还需要服务端流、客户端流、双向流

这也是 gRPC 和普通 REST 很不一样的地方。REST 更像“我请求一个资源，你返回结果”，gRPC 更像“我调用你的某个能力，而且这个调用可以是流式的”。公开资料里，gRPC 在大型微服务系统里很常见。比如 [Uber 的实时 Push 平台](https://www.uber.com/blog/ubers-next-gen-push-platform-on-grpc/) 就把协议演进到基于 gRPC 的双向流，CNCF 也有 [Netflix 使用 gRPC 的案例](https://www.cncf.io/case-studies/netflix/)。

如果只是普通前后端接口，REST 更容易调试和接入。如果是内部微服务、高吞吐、低延迟、跨语言调用，gRPC 会更适合。

## 前端取数：GraphQL

### GraphQL API

![GraphQL API|299](通信协议/assets/api-types/0280-graphql-api.jpg)

GraphQL 解决的是 REST 里的一个常见问题：要么拿多了，要么拿少了。

比如一个用户页只想显示用户名和邮箱，但 REST 接口可能直接返回完整用户资料，这就是 over-fetching。反过来，如果页面还要订单、评论、收藏，又可能要连续请求好几个接口，这就是 under-fetching。GraphQL 的思路是让客户端声明自己需要什么字段：
🌰：
```graphql
query {
  user(id: "1") {
    name
    email
    orders {
      total
    }
  }
}
```

服务端按这个查询返回刚好需要的数据。这样前端灵活度会高很多，尤其适合移动端、多端应用、复杂后台页面。

## 实时通知：Webhook、SSE、WebSocket

### Webhook API

![Webhook API|348](通信协议/assets/api-types/0373-webhooks-api.jpg)

Webhook 的出现，是因为轮询太浪费。

传统 API 是一直问服务端发送类似心跳包的询问。Webhook 的思路是反过来：你先给对方一个 callback URL，等事件发生时，对方主动发 HTTP 请求通知你。所以 Webhook 有时候也被叫 reverse API。不需要你时刻监控数据，数据变化后会主动来找你。比如你的 github 上的 push、pull request、issue 等事件发生时都会通知到你的邮箱里～

Webhook 适合“事件发生后通知我”，但它不是持续连接。对方请求你一次，你处理一次。。

### SSE

![SSE|299](通信协议/assets/api-types/0743-sse.jpg)

SSE 是 Server-Sent Events，适合服务端持续往浏览器推消息。

它的模型很简单：浏览器发起一个 HTTP 请求，服务端不立刻关闭连接，而是有新数据就顺着这个连接继续往下发。它主要是单向的，也就是服务端到客户端。

所以 SSE 很适合这些场景：

- 任务进度
- 日志流
- 通知流
- 股票价格、比分、状态更新
- AI 回复的流式输出

实际应用里，OpenAI API 的 streaming 响应就使用了 SSE 这种形式，适合一边生成一边返回内容，这里可以和 [[流式传输|流式传输]] 放在一起看。它比 WebSocket 简单，因为底层仍然是普通 HTTP，浏览器里也有 `EventSource` 这样的原生能力。SSE 主要适合服务端单方面推送客户端。如果双方都要高频互发消息，就要考虑 WebSocket。

### WebSocket API

![WebSocket API|330](通信协议/assets/api-types/0451-websockets-api.jpg)

WebSocket 解决的是双向实时通信。它先通过 HTTP 握手，然后升级成 WebSocket 连接。连接建立后，客户端和服务端都可以主动发消息。

这和 SSE 的区别很关键：SSE 更像服务端一直往你这里播报，WebSocket 更像双方开了一条实时电话线。[Discord Gateway](https://discord.com/developers/docs/events/gateway) 就是 WebSocket API，客户端通过它接收实时事件、心跳、状态变化。

WebSocket 的优势就是适合聊天、协作编辑、在线游戏、行情系统。代价是连接状态要维护，心跳、断线重连、扩容、负载均衡都比普通 HTTP 更复杂。

## 实时音视频：WebRTC

### WebRTC API

![WebRTC API|328](通信协议/assets/api-types/0511-webrtc-api.jpg)

WebRTC 是传文件之外的，比如传音频、视频、屏幕共享，甚至点对点传文件。一套浏览器和客户端实时通信能力。它可以处理音视频编解码、网络穿透、带宽自适应等问题。实际连接前通常还需要信令服务帮两端交换连接信息。

这里可以把 WebRTC 和 WebSocket 区分开：WebSocket 更适合传结构化消息，WebRTC 更适合实时媒体流。很多系统会一起用它们，比如用 WebSocket 做信令，再用 WebRTC 传音视频。

## AI 接工具：MCP

### MCP Server

![MCP Server|309](通信协议/assets/api-types/0594-mcp-server.jpg)

MCP 是 Model Context Protocol，模型上下文协议是 AI 应用这两年才开始大量出现的一类接口规范。AI 助手想访问文件、数据库、代码仓库、日历、搜索、内部系统时，如果每个工具都单独写一套集成，成本会很高。MCP 的思路是给模型和外部工具之间加一层统一协议。只要工具提供 MCP Server，支持 MCP 的客户端就能用比较统一的方式接入。

你可以类比成 AI 工具世界里的通用连接头，比如一个 coding assistant 通过 MCP 读取项目文件、查 Git 历史、访问文档或数据库，就不需要每个能力都重新做一套私有集成。[Model Context Protocol 官方文档](https://modelcontextprotocol.io/introduction) 把 MCP 定位为连接 LLM 应用和外部数据源、工具的开放协议；OpenAI Apps SDK 也支持围绕 MCP server 描述工具能力。

MCP 和前面那些 API 不完全在同一层。REST、GraphQL、gRPC 更像服务接口；MCP 更像 AI 应用里的连接协议。它可以把 REST API、数据库、文件系统这些能力包装成模型可调用的工具。

## 消息与事件：MQTT、AMQP、Event-Driven API、Kafka

### MQTT

![MQTT|330](通信协议/assets/api-types/0811-mqtt.jpg)

MQTT 它是一个非常轻量的协议，很多 IoT 设备带宽小、电量有限、网络也不稳定，MQTT 协议就非常合适，在智能家居、传感器、工业设备、车联网中都很常见。

它采用发布订阅模型。设备把消息发到某个 topic，订阅这个 topic 的客户端就能收到消息，像一个大广播一样把消息给传出去。比如温度传感器发布数据，手机 App、仪表盘、自动化系统都可以收到它的信息。

### AMQP

![AMQP|311](通信协议/assets/api-types/0869-amqp.jpg)

AMQP 更偏可靠消息队列，注重消息不能丢、要能确认、要能路由、要能持久化，如果出现错误也会有错误的处理机制。如果一个支付消息、订单消息、医疗系统消息丢了，后果就比较严重。AMQP 这类协议会提供 acknowledgement、持久化、事务、路由等能力。RabbitMQ 是 AMQP 生态里很典型的实现。

### Event-Driven API

![Event-Driven API|398](通信协议/assets/api-types/0932-event-driven-api.jpg)

Event-Driven API 更像一种架构思路，不是某一个具体协议。

传统同步调用里，订单服务可能要依次调用库存服务、邮件服务、物流服务、分析服务。如果其中一个服务挂了，整个链路就容易被拖住。事件驱动的做法是：订单服务只发布一个 `OrderPlaced` 事件。库存服务监听这个事件去扣库存，邮件服务监听它去发确认邮件，分析服务监听它去记录数据。每个服务做自己的事，不需要订单服务挨个等结果。

这个模式的好处是解耦，后面新增一个短信通知服务，只要监听订单事件就行。

### Apache Kafka

![Apache Kafka|409](通信协议/assets/api-types/0999-apache-kafka.jpg)

Kafka 可以理解成事件驱动架构扩展到大规模之后的基础设施。普通消息队列通常是消息被消费后就删除，而 Kafka 更像一条可持久化、可回放的事件日志。最早就是 LinkedIn 为了处理大规模站内事件流做出来的，后来进入 Apache 生态。视频里举了 Netflix、Uber、LinkedIn 这类例子，本质都差不多：用户行为、播放、司机位置、消息、连接关系，这些事件量非常大，而且后续很多系统都要消费。

普通异步任务RabbitMQ / AMQP 可能就够了。如果是海量事件、日志、埋点、实时数据管道，Kafka 更合适。

### AsyncAPI

![AsyncAPI|433](通信协议/assets/api-types/1038-asyncapi.jpg)

当系统里只有几个 HTTP 接口时，用 OpenAPI / Swagger 就能把接口列清楚。但如果公司里有几十个服务通过 Kafka、MQTT、AMQP、WebSocket 通信，问题就变成：谁会发布什么事件？事件字段是什么？谁会消费？版本怎么演进？

**AsyncAPI 就是给这类异步接口写规范**，它会描述 channel、message、schema、producer、consumer 等信息。这样新同事接手时不用到处翻代码，也能知道系统里有哪些事件。

[AsyncAPI 官方文档](https://www.asyncapi.com/docs) 里把它定位为描述事件驱动和消息 API 的规范，[案例页](https://www.asyncapi.com/casestudies) 也能看到它主要服务的是异步系统文档化、工具生成和团队协作。

这里可以这么理解：Kafka / MQTT / AMQP 是通信方式，AsyncAPI 是说明书。

## 简单选择

| 需求 | 可以优先考虑 |
|---|---|
| 普通前后端请求、CRUD、公开接口 | REST |
| 企业系统强契约、XML、事务和安全规范 | SOAP |
| 内部服务像调用函数一样互相调用 | RPC |
| 微服务高性能、跨语言、流式调用 | gRPC |
| 前端只想拿需要的字段 | GraphQL |
| 第三方系统事件发生后通知你 | Webhook |
| 服务端持续向浏览器推送 | SSE |
| 客户端和服务端双向实时通信 | WebSocket |
| 音视频、屏幕共享、P2P 通信 | WebRTC |
| IoT 设备、低带宽、小消息 | MQTT |
| 可靠消息队列、企业业务消息 | AMQP |
| 服务之间通过事件解耦 | Event-Driven API |
| 大规模事件流、日志、实时数据管道 | Kafka |
| AI 模型接工具和数据源 | MCP |
| 异步 API 的文档和契约 | AsyncAPI |

## 资料来源

- [GitHub REST API](https://docs.github.com/en/rest)
- [GitHub GraphQL API](https://docs.github.com/en/graphql)
- [GitHub Webhooks](https://docs.github.com/en/webhooks)
- [Stripe Webhooks](https://docs.stripe.com/webhooks)
- [Slack Incoming Webhooks](https://api.slack.com/messaging/webhooks)
- [Discord Gateway](https://discord.com/developers/docs/events/gateway)
- [Discord Webhooks](https://discord.com/developers/docs/resources/webhook)
- [Binance WebSocket Streams](https://developers.binance.com/docs/binance-spot-api-docs/web-socket-streams)
- [Salesforce SOAP API](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_quickstart_intro.htm)
- [Shopify Admin GraphQL API](https://shopify.dev/docs/api/admin-graphql)
- [Ethereum JSON-RPC API](https://ethereum.org/en/developers/docs/apis/json-rpc/)
- [Cloudflare Workers RPC](https://developers.cloudflare.com/workers/runtime-apis/rpc/)
- [WebRTC](https://webrtc.org/)
- [OpenAI Streaming Responses](https://platform.openai.com/docs/guides/streaming-responses?api-mode=responses)
- [Model Context Protocol](https://modelcontextprotocol.io/introduction)
- [OpenAI Apps SDK](https://developers.openai.com/apps-sdk/)
- [Uber Next Gen Push Platform on gRPC](https://www.uber.com/blog/ubers-next-gen-push-platform-on-grpc/)
- [CNCF Netflix gRPC case study](https://www.cncf.io/case-studies/netflix/)
- [AWS IoT Core MQTT](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)
- [Azure Service Bus AMQP](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-overview)
- [Amazon EventBridge](https://aws.amazon.com/eventbridge/)
- [Apache Kafka Introduction](https://kafka.apache.org/intro)
- [AsyncAPI Docs](https://www.asyncapi.com/docs)
