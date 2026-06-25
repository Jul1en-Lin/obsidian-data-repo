# Spring Cloud Config
> 相关笔记：[[Spring Cloud|微服务 知识总结]]


Spring Cloud Config 解决的是微服务项目中“配置分散、环境复杂、修改成本高”的问题。单体项目里，配置通常写在本地 `application.yml` 中，应用启动时直接读取即可。但微服务项目会拆出多个服务，每个服务又会区分开发、测试、生产等环境。如果每个服务都自己维护一份配置，时间久了就会出现配置重复、版本混乱、修改困难的问题。

配置中心的思路就是把配置从服务代码里抽出来，交给一个统一的服务管理。Spring Cloud Config 中，承担统一管理角色的是 Config Server，真正使用配置的业务服务是 Config Client。Config Server 从 Git、本地文件系统、Vault、JDBC 等后端读取配置，再通过 HTTP 接口提供给各个客户端。Config Client 启动时访问 Config Server，拉取属于自己的配置并加载到 Spring Environment 中。

![test](image-20260515153553-efbbqze.png)

## 为什么需要 Spring Cloud Config

微服务拆分之后，配置数量会快速增加。比如一个电商系统可能有用户服务、订单服务、商品服务、网关服务，每个服务又可能有 `dev`​、`test`​、`prod` 三套环境。如果所有配置都放在各自服务内部，修改数据库地址、缓存地址、开关配置时，就需要分别进入多个服务修改，甚至重新打包发布。

Spring Cloud Config 把配置统一放到远程配置仓库中，服务本身只保留“我是谁、我要去哪里拉配置”这类最基础的信息。这样一来，配置的维护位置就从“每个微服务内部”变成了“统一配置仓库”，配置读取入口也从“服务本地文件”变成了“Config Server”。

这种变化带来两个直接好处。第一，配置可以集中管理，避免同一个配置在多个服务中重复维护。第二，配置可以通过 Git 追踪历史，谁改了什么、什么时候改的，都有记录。对于团队协作和多环境管理来说，这比散落在各个服务里的配置更可靠。

## Config Server 与 Config Client 的关系

Config Server 是配置中心服务端，它不一定直接保存配置文件，而是连接一个配置后端。入门阶段最常见的是 Git 仓库。Git 仓库中可以存放类似 `user-service-dev.yml`​、`order-service-prod.yml` 这样的配置文件。Config Server 启动后，会读取这些配置，并通过 HTTP 接口提供给客户端。

Config Client 是具体业务服务，例如 `user-service`​、`order-service`​、`gateway-service`。这些服务启动时会主动访问 Config Server。访问时，客户端会告诉服务端三个关键信息：应用名、环境和配置版本。Config Server 根据这三个信息找到对应配置，再返回给客户端。

这里的三个信息非常重要。`application`​ 表示应用名，通常来自 `spring.application.name`​。`profile`​ 表示环境，例如 `dev`​、`test`​、`prod`​。`label`​ 通常表示 Git 分支或标签，例如 `main`​、`master`​、`v1.0`。所以 Config Server 本质上是在回答一个问题：某个应用在某个环境下，需要读取哪个版本的配置。

如果 `user-service`​ 启动时指定了应用名为 `user-service`​，环境为 `dev`​，配置仓库分支为 `main`​，那么它访问 Config Server 时，Config Server 就会尝试查找和 `user-service-dev` 相关的配置，并把结果组合成 Spring 可以识别的 PropertySource 返回给客户端。

## Config Server 简单部署

搭建 Config Server 的第一步是创建一个普通 Spring Boot 服务，并添加 Config Server 依赖。这个依赖让当前服务具备从配置仓库读取配置并对外暴露配置接口的能力。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

然后在启动类上添加 `@EnableConfigServer`。这个注解的作用是开启 Config Server 功能，让这个 Spring Boot 应用不再只是普通服务，而是可以作为配置中心服务端工作。

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

接下来需要告诉 Config Server 配置仓库在哪里。最常见的是配置 Git 仓库地址。

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://example.com/config-repo.git
          default-label: main
```

​`server.port`​ 决定 Config Server 的访问端口，常见示例端口是 `8888`​。`spring.cloud.config.server.git.uri`​ 指向远程配置仓库地址。`default-label` 表示默认读取哪个分支或标签。

启动 Config Server 后，如果配置仓库中存在 `user-service-dev.yml`​，客户端或浏览器可以通过类似 `http://localhost:8888/user-service/dev/main`​ 的路径访问配置。这个路径中的 `user-service`​ 对应应用名，`dev`​ 对应环境，`main` 对应 Git 分支。

本地学习时也可以使用 native 模式，把配置仓库放在本地目录中。它的好处是简单，不需要先准备远程 Git 仓库，但真实项目中更常用 Git，因为 Git 更适合团队协作、版本回退和变更审计。

```yaml
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: file:///D:/config-repo
```

## Config Client 简单部署

Config Client 是业务服务。它的目标不是管理配置，而是在启动时从 Config Server 拿到自己的配置。客户端需要添加 `spring-cloud-starter-config` 依赖。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

新版本更推荐使用 `spring.config.import` 的方式导入远程配置。这样写的含义很明确：当前应用启动时，除了读取本地配置，也会从 Config Server 导入配置。

```yaml
spring:
  application:
    name: user-service
  profiles:
    active: dev
  config:
    import: optional:configserver:http://localhost:8888
```

这里的因果关系要理解清楚。`spring.application.name`​ 决定应用名，所以它影响 Config Server 查找哪个服务的配置。`spring.profiles.active`​ 决定当前环境，所以它影响读取 `dev`​、`test`​ 还是 `prod`​ 配置。`spring.config.import` 决定配置来源，所以它告诉 Spring Boot：请到这个 Config Server 地址拉取远程配置。

如果配置仓库中有 `user-service-dev.yml`，内容如下：

```yaml
user:
  level: vip
```

那么 `user-service`​ 启动后，就可以像读取本地配置一样读取到 `user.level`。从业务代码角度看，它不需要关心这个配置原本来自 Git 还是本地文件，因为配置最终都会进入 Environment。

```java
@RestController
public class UserController {

    @Value("${user.level:normal}")
    private String userLevel;

    @GetMapping("/level")
    public String level() {
        return userLevel;
    }
}
```

到这里，Spring Cloud Config 的第一条主线就完整了：配置放在 Git 中，Config Server 读取 Git，Config Client 启动时从 Config Server 拉取配置。

## 刷新机制

Config Client 启动时会拉取配置，但服务启动完成后，它不会自动每秒去检查配置仓库有没有变化。也就是说，如果你修改了 Git 仓库中的配置，已经运行中的服务通常不会立即感知。

这就产生了一个新问题：配置中心解决了“配置放在哪里、启动时怎么读取”的问题，但还没有完全解决“运行时配置变化后怎么生效”的问题。

最基础的方式是手动刷新。比如通过 Actuator 的 `/actuator/refresh` 端点让某个服务重新加载配置。这个方式在单个服务、单个实例时还能接受，但在真实微服务场景中很快会变得麻烦。

假设系统有 5 个服务，每个服务部署 3 个实例，那么一次配置变化可能需要处理 15 个实例。如果人工逐个调用刷新接口，不仅低效，还容易漏掉某个实例。为了让配置变更自动传播，就需要 Webhook 和 Spring Cloud Bus 配合。

## Webhook 的概念

Webhook 是一种事件回调机制。它和轮询正好相反。轮询是服务不断去问配置仓库有没有变化；Webhook 是配置仓库发生变化后，主动通知指定地址。

以 Git 仓库为例，当开发者修改配置并 push 到 GitHub、GitLab 或 Gitee 后，仓库平台可以向 Config Server 的某个接口发送 HTTP POST 请求。这个请求的意义不是传输完整配置，而是告诉 Config Server：配置仓库已经发生变化，你需要处理刷新。

在 Spring Cloud Config 中，Config Server 引入 `spring-cloud-config-monitor`​ 后，可以开启 `/monitor`​ 端点。Webhook 通常就配置到这个 `/monitor`​ 端点上。于是配置变化后的链路变成：开发者修改配置，push 到仓库，仓库触发 Webhook，Webhook 请求 Config Server 的 `/monitor`，Config Server 获知配置变化。

![image](image-20260515153616-soy4td7.png)

Webhook 只解决了“谁来告诉 Config Server 仓库变了”的问题。但 Config Server 知道配置变了以后，还要通知所有相关微服务实例刷新。这个广播能力就是 Spring Cloud Bus 要解决的问题。

## Spring Cloud Bus 自动刷新机制

Spring Cloud Bus 可以理解为分布式系统中的消息总线。它把多个微服务实例通过消息代理连接起来，例如 RabbitMQ 或 Kafka。Config Server 收到配置变化通知后，可以把刷新事件发送到 Bus 上，再由 Bus 广播给相关的 Config Client 实例。

有了 Spring Cloud Bus，配置刷新链路就从“人工逐个刷新实例”变成了“触发一次事件，消息总线自动广播”。这个变化的关键价值是降低运维成本，并减少遗漏实例的风险。

完整流程可以按因果顺序理解。首先，配置仓库发生变化。然后，Webhook 把变化通知给 Config Server 的 `/monitor`​。Config Server 收到通知后，不直接逐个调用服务，而是发布一个 `RefreshRemoteApplicationEvent`​。接着，Spring Cloud Bus 通过 RabbitMQ 或 Kafka 把这个事件广播出去。最后，Config Client 收到事件后刷新本地 Environment，并重新创建或刷新带有 `@RefreshScope` 的 Bean。

​`@RefreshScope`​ 很关键。配置刷新不是让所有对象都自动变成新值。只有那些被纳入刷新范围的 Bean，才会在刷新事件到来后重新读取配置。因此，运行时可能变化的配置类，通常会配合 `@RefreshScope` 使用。

## Webhook + Bus 简单配置

Config Server 侧需要具备两个能力：接收仓库变更通知，以及把刷新事件发布到消息总线。因此需要添加 monitor 和 bus 相关依赖。以 RabbitMQ 为例：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-monitor</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

Config Client 侧也需要接入 Bus，因为客户端要能从消息总线接收刷新事件。同时还需要 Actuator 参与刷新相关能力。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Config Server 和 Config Client 要连接同一个消息代理。如果使用 RabbitMQ，可以配置连接信息。

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

如果某个配置类需要运行时刷新，可以使用 `@RefreshScope`。

```java
@Component
@RefreshScope
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

配置仓库的 Webhook 地址通常指向 Config Server 的 `/monitor`​。本地测试时，也可以手动向 `/monitor` 发送请求，模拟仓库触发通知。

```bash
curl -X POST http://localhost:8888/monitor \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "path=user-service"
```

这里的 `path=user-service`​ 表示和 `user-service` 相关的配置发生变化，Config Server 会尝试将刷新事件发送给匹配的应用。

## 加密环境

配置中心集中管理配置后，新的问题出现了：敏感配置也会集中起来。数据库密码、Redis 密码、第三方接口密钥如果直接以明文形式存入 Git 仓库，一旦仓库权限配置不当、人员误操作、日志暴露或历史提交泄露，就可能造成安全问题。

所以，配置中心不仅要解决“配置统一管理”，还要解决“敏感配置如何安全存储”。Spring Cloud Config 的加密机制就是为这个问题服务的。

它的基本思路是：Git 仓库里不直接保存明文密码，而是保存以 `{cipher}`​ 开头的密文。Config Server 读取配置时识别到 `{cipher}` 前缀，就在返回给 Config Client 前完成解密。这样，仓库里是密文，业务服务拿到的是正常可用的明文配置。

![image](image-20260515153630-elxti8w.png)

## 加密与解密端点

Config Server 提供 `/encrypt`​ 和 `/decrypt`​ 两个端点。`/encrypt`​ 用来把明文变成密文，`/decrypt` 用来验证密文是否可以正确还原。

例如，把一个明文密码加密：

```bash
curl localhost:8888/encrypt -s -d mysecret
```

返回的密文需要加上 `{cipher}` 前缀后写入配置文件。

```yaml
spring:
  datasource:
    username: appuser
    password: '{cipher}682bc583f4641835fa2db009355293665d2647dade3375c0ee201de2a49f7bda'
```

如果使用 `.properties` 文件，加密值不要再额外加引号，否则可能影响解密。

```properties
spring.datasource.password={cipher}682bc583f4641835fa2db009355293665d2647dade3375c0ee201de2a49f7bda
```

​`/decrypt` 常用于本地验证密文是否正确。

```bash
curl localhost:8888/decrypt -s -d 682bc583f4641835fa2db009355293665d2647dade3375c0ee201de2a49f7bda
```

如果 Config Server 解密失败，它不会简单地把错误密文当成密码继续使用，而是可能生成带 `invalid` 前缀的属性。这种设计是为了避免密文被误当成真实密码传给客户端。

## 对称加密

对称加密的特点是加密和解密使用同一个密钥。它的优点是配置简单，非常适合学习和本地 demo。Config Server 只需要知道这个共享密钥，就能完成 `/encrypt`​ 和 `/decrypt`。

```yaml
encrypt:
  key: mysecretkey
```

也可以用环境变量保存密钥，避免把密钥写死在配置文件中。

```bash
ENCRYPT_KEY=mysecretkey
```

对称加密的风险也来自这个“同一个密钥”。因为加密和解密都依赖它，一旦密钥泄露，别人就可以解密所有使用这个密钥生成的配置密文。所以对称加密虽然方便，但在生产环境中要非常重视密钥的存放和权限控制。

## 非对称加密

非对称加密使用一对密钥：公钥和私钥。通常可以用公钥加密，用私钥解密。它比对称加密更适合正式环境，因为加密能力和解密能力可以分离，密钥管理也更规范。

在 Spring Cloud Config 中，非对称加密通常通过 keystore 配置。可以用 JDK 自带的 `keytool` 创建测试密钥库。

```bash
keytool -genkeypair -alias mytestkey -keyalg RSA \
  -keystore server.jks
```

更完整的测试命令可以包含密码和证书信息。

```bash
keytool -genkeypair -alias mytestkey -keyalg RSA \
  -dname "CN=Web Server,OU=Unit,O=Organization,L=City,S=State,C=US" \
  -keypass changeme -keystore server.jks -storepass letmein
```

把生成的 `server.jks` 放到 Config Server 的 classpath 下，然后配置 keystore 信息。

```yaml
encrypt:
  keyStore:
    location: classpath:/server.jks
    password: letmein
    alias: mytestkey
    secret: changeme
```

这里的 `location`​ 指向密钥库位置，`password`​ 用于打开密钥库，`alias`​ 指定使用哪个密钥，`secret` 是密钥自身的密码。相比对称加密，非对称加密配置更复杂，但安全边界更清晰。

![image](image-20260515153648-fraj3t8.png)

## 复习总结

Spring Cloud Config 的知识链路可以从“配置管理”一路推到“自动刷新”和“安全加密”。

最开始的问题是微服务配置太分散，所以引入 Config Server 和 Config Client。Config Server 统一读取配置，Config Client 启动时拉取配置。配置集中之后，又会遇到运行时变更不生效的问题，所以引入刷新机制。手动刷新在多实例场景下成本高，于是使用 Webhook 发现配置仓库变化，再通过 Spring Cloud Bus 把刷新事件广播给所有相关客户端。配置集中管理后，敏感信息也被集中存储，因此还需要加密机制。Config Server 通过 `{cipher}`​、`/encrypt`​、`/decrypt`、对称密钥或非对称密钥库来保护敏感配置。

简化记忆如下：

- Config Server 解决“配置统一放在哪里、怎么对外提供”。
- Config Client 解决“业务服务启动时怎么拿配置”。
- Webhook 解决“配置仓库变化后怎么通知 Config Server”。
- Spring Cloud Bus 解决“Config Server 怎么把刷新事件广播给所有实例”。
- ​`@RefreshScope` 解决“哪些 Bean 能在运行时重新加载配置”。
- ​`{cipher}` 解决“敏感配置如何以密文形式保存在 Git 中”。
- 对称加密适合快速配置，非对称加密适合更正式的安全场景。

‍
