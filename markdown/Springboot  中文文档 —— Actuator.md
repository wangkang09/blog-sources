[TOC]

- 本文是 Springboot 2.1.1.RELEASE 的中文翻译

- Spring Boot 包含许多其他功能，可帮助你在将应用程序推送到生产环境时监控和管理应用程序
- 你可以选择使用 HTTP 端点或 JMX 来管理和监控应用程序
- 审计、健康和指标收集也可以自动应用于你的应用程序

## 1 **启用**

- [`spring-boot-actuator`](https://github.com/spring-projects/spring-boot/tree/v2.1.3.RELEASE/spring-boot-project/spring-boot-actuator) 模块提供了 Spring Boot 的所有生产就绪功能
- 启用这些功能的最简单方法是添加 `spring-boot-starter-actuator` starter 到依赖中。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- actuator 依赖 web-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```



## 2 **端点(endpoints)**

- 通过 Actuator 端点，你可以监控应用程序并与之交互
- Spring Boot 包含许多内置端点，也允许你添加自己的端点
- 例如，`health` 端点提供基本的应用程序健康信息



- 可以[启用或禁用](https://www.springcloud.cc/spring-boot.html#production-ready-endpoints-enabling-endpoints)每个单独的端点。它控制是否在应用程序上下文(bean 容器)中创建端点

- 要远程访问，还必须[通过JMX或HTTP公开端点](https://www.springcloud.cc/spring-boot.html#production-ready-endpoints-exposing-endpoints) 
- 大多数应用程序选择HTTP，其中端点的ID以及`/actuator`的前缀映射到URL
  - 例如，默认情况下，`health`端点映射到`/actuator/health`。

- 想了解有关 Actuator 端点及其请求和响应格式的更多信息，请参阅单独的API文档（[HTML](https://docs.spring.io/spring-boot/docs/2.1.1.RELEASE/actuator-api//html)或 [PDF](https://docs.spring.io/spring-boot/docs/2.1.1.RELEASE/actuator-api//pdf/spring-boot-actuator-web-api.pdf)）



### 2.1 启用端点

- 默认情况下，actuator 启用除 `shutdown` 之外的所有端点
- 要配置端点的启用，请使用其 `management.endpoint.<id>.enabled` 属性

```ini
management.endpoint.shutdown.enabled=true
```

- 可以改变默认值，即可以将所有端点都默认启用或默认关闭

```ini
# 只启用 info 端点
management.endpoints.enabled-by-default=false
management.endpoint.info.enabled=true
```

- **注：** 这里的禁用端点，是将其从上下文中完全去除
- **注：**如果执行暴露的技术（如：JMX、Web），请使用 2.2 中的 include 和 exclude 属性

### 2.2 公开端点

- 由于端点可能包含敏感信息，因此应仔细考虑何时暴露它们

| ID                 | JMX  | Web  |
| ------------------ | ---- | ---- |
| `auditevents`      | 是   | 否   |
| `beans`            | 是   | 否   |
| `caches`           | 是   | 否   |
| `conditions`       | 是   | 否   |
| `configprops`      | 是   | 否   |
| `env`              | 是   | 否   |
| `flyway`           | 是   | 否   |
| `health`           | 是   | 是   |
| `heapdump`         | N/A  | 否   |
| `httptrace`        | 是   | 否   |
| `info`             | 是   | 是   |
| `integrationgraph` | 是   | 否   |
| `jolokia`          | N/A  | 否   |
| `logfile`          | N/A  | 否   |
| `loggers`          | 是   | 否   |
| `liquibase`        | 是   | 否   |
| `metrics`          | 是   | 否   |
| `mappings`         | 是   | 否   |
| `prometheus`       | N/A  | 否   |
| `scheduledtasks`   | 是   | 否   |
| `sessions`         | 是   | 否   |
| `shutdown`         | 是   | 否   |
| `threaddump`       | 是   | 否   |

- 要更改暴露的端点，请使用以下特定的 `include` 和 `exclude` 属性：

| 属性                                        | 默认           |
| ------------------------------------------- | -------------- |
| `management.endpoints.jmx.exposure.exclude` |                |
| `management.endpoints.jmx.exposure.include` | `*`            |
| `management.endpoints.web.exposure.exclude` |                |
| `management.endpoints.web.exposure.include` | `info, health` |

- `exclude` 属性优先于 `include` 属性

```ini
# 修改 jmx.include 默认值，使得 jmx 仅暴露 health,info 端点
management.endpoints.jmx.exposure.include=health,info
# 修改 web.include 默认值，并添加 exclude，使得 web 暴露除 env,beans 以外的所有端点
management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude=env,beans
```

- **注：** 因为 * 在 yaml 文件中具有特殊含义，使用 * 号，必须加引号（'\*' 或 "\*"）

- 如果你的应用程序是公开的，我们强烈建议你[保护您的端点](https://www.springcloud.cc/spring-boot.html#production-ready-endpoints-security)

- 如果您想在暴露端点时实施自己的策略，可以注册`EndpointFilter` bean



### 2.3 保护HTTP端点

- 您应该像使用任何其他敏感URL一样注意保护HTTP端点
- 如果存在 Spring Security，则默认使用 Spring Security 的内容协商策略来保护端点
- 例如，如果你希望为 HTTP 端点配置自定义安全策略，只允许具有特定角色身份的用户访问它们，Spring Boot 提供了方便的 `RequestMatcher` 对象，可以与 Spring Security 结合使用

```java
@Configuration
public class ActuatorSecurity extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //EndpointRequest.toAnyEndpoint() 将请求与所有端点进行匹配，然后确保所有端点都具有 ENDPOINT_ADMIN 角色
        http.requestMatcher(EndpointRequest.toAnyEndpoint()).authorizeRequests()
                .anyRequest().hasRole("ENDPOINT_ADMIN")
                .and()
            	.httpBasic();
    }
}
```

- `EndpointRequest` 上还提供了其他几种匹配器方法。有关详细信息，请参阅API文档（[HTML](https://docs.spring.io/spring-boot/docs/2.1.1.RELEASE/actuator-api//html)或 [PDF](https://docs.spring.io/spring-boot/docs/2.1.1.RELEASE/actuator-api//pdf/spring-boot-actuator-web-api.pdf)）

- 如果应用程序部署在有防火墙的环境，可以暴露所有的 web 端点 通过 2.2 中的 web.include 方式

- 且，如果存在 Spring Security，还需要添加自定义安全配置，以使得所有请求都能通过 security

```java
@Configuration
public class ActuatorSecurity extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.requestMatcher(EndpointRequest.toAnyEndpoint()).authorizeRequests()
            .anyRequest().permitAll();
    }
}
```



### 2.4 配置端点缓存

- 端点对不带参数读取操作的响应自动缓存
- 要配置端点缓存响应的时间长度，请使用其 `cache.time-to-live` 属性

```ini
# 将beans端点缓存的生存时间设置为10秒
management.endpoint.beans.cache.time-to-live=10s
```

- **注：** 前缀 `management.endpoint.<name>` 用于唯一标识配置的端点
- **注：** 在进行一个身份验证 HTTP 请求时，`Principal` 被视为端点的输入，因此不会缓存响应



### 2.5 配置 /actuator 路径

- 使用 `host:ip/${base-path}` 可以访问 `发现页面` (返回所有暴露的端点信息) 
- 使用 `host:ip/${base-path}/xx` 可以访问特定的端点信息
- 默认情况下 `base-path=/actuator`

- **注：** 当  `base-path = /` 时，将禁用 `发现页面` 以防止与其他映射发生冲突

```ini
# 可通过此配置更改默认 path
management.endpoints.web.base-path = /aa
```



### 2.6 跨越支持(CORS)

- [跨源资源共享](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) （CORS）是一种[W3C规范](https://www.w3.org/TR/cors/)，允许您以灵活的方式指定授权的跨域请求类型
- 如果您使用Spring MVC或Spring WebFlux，可以配置Actuator的Web端点以支持此类方案

- 默认情况下禁用CORS支持，仅在设置了`management.endpoints.web.cors.allowed-origins`属性后才启用CORS支持

```ini
# 允许来自example.com域的GET和POST 请求
management.endpoints.web.cors.allowed-origins=http://example.com
management.endpoints.web.cors.allowed-methods=GET,POST
```

- **注：** 有关 选项的完整列表，请参阅 [CorsEndpointProperties](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator-autoconfigure/src/main/java/org/springframework/boot/actuate/autoconfigure/endpoint/web/CorsEndpointProperties.java)



### 2.7 自定义端点

- 如果您通过 @Bean 注解将 @Endpoint 注解的类配置到容器中
- 则该类中使用`@ReadOperation`，`@WriteOperation`或`@DeleteOperation`注释的任何方法都会通过JMX自动公开
- 并且在Web应用程序中也会通过HTTP自动公开
- 可以使用Jersey，Spring MVC或Spring WebFlux通过HTTP公开端点



- 您还可以使用 `@JmxEndpoint` 或 `@WebEndpoint` 编写特定技术的端点
- 您些端点仅限于各自的技术。例如，`@WebEndpoint` 仅通过 HTTP 暴露，而不是 JMX



- 您可以使用`@EndpointWebExtension`和`@EndpointJmxExtension`编写特定于技术的扩展
- 通过这些注释，您可以提供特定于技术的操作来扩充现有端点



- 如果您使用的是特定的 Web 框架，完全可以通过 @Controller、@RestController 代替 @Endpoint 注解
- 但是这样的话，这些端点，就无法通过 JMX 或使用其他 Web 框架（@Endpoint 具有统一性）

#### 2.7.1 接收输入

- 端点上的操作通过其参数接收输入
- 通过Web公开时，这些参数取自于 请求 parameters 、JSON 格式的 body
- 通过JMX公开时，参数将映射到MBean操作的参数
- 默认情况下需要参数，可以通过使用`@org.springframework.lang.Nullable`注释它们来使它们成为可选项

- **注：** 由于端点与技术无关，因此只能在方法签名中指定简单类型。不支持自定义类型声明单个参数

- To allow the input to be mapped to the operation method's parameters, Java code implementing an endpoint should be compiled with `-parameters`, and Kotlin code implementing an endpoint should be compiled with `-java-parameters`.
- This will happen automatically if you are using Spring Boot's Gradle plugin or if you are using Maven and `spring-boot-starter-parent`.

##### 2.7.1.1 输入类型转换

- 如有必要，传递给端点操作方法的参数将自动转换为所需类型
- 在调用操作方法之前，通过JMX或HTTP请求接收的输入将使用`ApplicationConversionService`的实例转换为所需类型

#### 2.7.2 自定义Web端点

- `@Endpoint`，`@WebEndpoint`或`@EndpointWebExtension`上的操作将使用Jersey，Spring MVC或Spring WebFlux通过HTTP自动公开

##### 2.7.2.1 Web Endpoint Request Predicates

- 为 Web 暴露端点上的每个操作自动生成 Request Predicate

##### 2.7.2.2 路径

- predicate   的路径由端点的ID和Web暴露的端点的基本路径确定
- 默认基本路径为`/actuator`。例如，ID为`sessions`的端点将使用`/actuator/sessions`作为 predicate   中的路径
- 可以通过使用`@Selector`注释操作方法的一个或多个参数来进一步定制路径

##### 2.7.2.3 HTTP方法

- predicate   的HTTP方法由操作类型决定，如下表所示：

| 操作               | HTTP 方法 |
| ------------------ | --------- |
| `@ReadOperation`   | `GET`     |
| `@WriteOperation`  | `POST`    |
| `@DeleteOperation` | `DELETE`  |

##### 2.7.2.4 Consume

- 对于使用请求体的 `@WriteOperation`（HTTP `POST`），predicate   的 consume 子句是 `application/vnd.spring-boot.actuator.v2+json, application/json`
- 对于所有其他操作，consume 子句为空

##### 2.7.2.5 Produce

- 谓词的produce子句可以由`@DeleteOperation`，`@ReadOperation`和`@WriteOperation`注释的`produces`属性确定
- 该属性是可选的。如果未使用，则自动确定produce子句

- 如果操作方法返回`void`或`Void`，则produce子句为空。如果操作方法返回`org.springframework.core.io.Resource`，则produce子句为`application/octet-stream`。对于所有其他操作，produce子句是`application/vnd.spring-boot.actuator.v2+json, application/json`

##### 2.7.2.6 Web 端点响应状态

- 端点操作的默认响应状态取决于操作类型（读取、写入或删除）以及操作返回的内容（如果有）
- `@ReadOperation` 返回一个值，响应状态为 200（OK）。如果它未返回值，则响应状态将为 404（未找到）
- 如果 `@WriteOperation` 或 `@DeleteOperation` 返回值，则响应状态将为 200（OK）。如果它没有返回值，则响应状态将为 204（无内容）
- 如果在没有必需参数的情况下调用操作，或者使用无法转换为所需类型的参数，则不会调用操作方法，并且响应状态将为 400（错误请求）

##### 2.7.2.7 Web 端点范围请求

- 可用 HTTP 范围请求请求部分 HTTP 资源
- 使用 Spring MVC 或 Spring Web Flux 时，返回 `org.springframework.core.io.Resource` 的操作会自动支持范围请求
- **注：** 使用 Jersey 时不支持范围请求

##### 2.7.2.8 Web 端点安全

- Web 端点或特定 Web 的端点扩展上的操作可以接收当前的 `java.security.Principal` 或 `org.springframework.boot.actuate.endpoint.SecurityContext` 作为方法参数
- 前者通常与 `@Nullable` 结合使用，为经过身份验证和未经身份验证的用户提供不同的行为
- 后者通常用于使用其 `isUserInRole(String)` 方法执行授权检查



#### 2.7.3 Servlet 端点

- 通过实现一个带有 `@ServletEndpoint` 注解的类，`Servlet` 可以作为端点暴露，该类也实现了 `Supplier<EndpointServlet>`
- Servlet 端点提供了与 Servlet 容器更深层次的集成，但代价是可移植性
- 它们旨在用于将现有 Servlet 作为端点暴露
- 对于新端点，应尽可能首选 `@Endpoint` 和 `@WebEndpoint` 注解



#### 2.7.4 Controller endpoints

- `@ControllerEndpoint`和`@RestControllerEndpoint`可用于实现仅由Spring MVC或Spring WebFlux公开的端点
- 使用Spring MVC和Spring WebFlux的标准注释（例如`@RequestMapping`和`@GetMapping`）映射方法，并将端点的ID用作路径的前缀
- Controller endpoints 提供与Spring Web框架的更深层次集成，但代价是可移植性。应尽可能优先考虑`@Endpoint`和`@WebEndpoint`注释



### 2.8 **健康信息（Health Information）**

- 您可以使用运行状况信息来检查正在运行的应用程序的 **健康状态**
- 监视软件经常使用它来在生产系统出现故障时向某人发出警报
- `health`端点公开的信息取决于`management.endpoint.health.show-details`属性，该属性可以使用以下值之一进行配置

| 名称              | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| `never`           | 细节永远不会显示。                                           |
| `when-authorized` | 详细信息仅向授权用户显示。可以使用`management.endpoint.health.roles`配置授权角色。 |
| `always`          | 详细信息显示给所有用户。                                     |

- 默认值为`never`。当用户处于一个或多个端点的角色时，将被视为已获得授权
- 如果端点没有配置角色（默认值），则认为所有经过身份验证的用户都已获得授权
- 可以使用`management.endpoint.health.roles`属性配置角色

- **注：** 如果你已保护应用程序并希望使用 `always`，则安全配置必须允许所有用户（不管有没有经过身份验证）对健康端点的访问，参考 2.3



- 健康信息是从 [`HealthIndicatorRegistry`](https://github.com/spring-projects/spring-boot/tree/v2.1.3.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/HealthIndicatorRegistry.java) 的内容中收集的（默认情况下，`ApplicationContext` 中定义的所有 [`HealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.3.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/HealthIndicator.java) 实例）
- Spring Boot 包含许多自动配置的 `HealthIndicators`，你也可以自己编写
- 默认情况下，最终系统状态由 `HealthAggregator` 根据状态的有序列表对每个 `HealthIndicator` 的状态进行排序
- 排序列表中的第一个状态作为整体健康状态。如果没有 `HealthIndicator` 返回一个 `HealthAggregator` 已知的状态，则使用 `UNKNOWN` 状态

- **注：** `HealthIndicatorRegistry` 可用于在运行时注册和注销健康指示器

#### 2.8.1 Auto-configured HealthIndicators

- Spring Boot 自动配置了以下 `HealthIndicator**`**
- **注：** 您可以通过设置`management.health.defaults.enabled`属性来禁用所有

| 名称                                                         | 描述                                   |
| ------------------------------------------------------------ | -------------------------------------- |
| [`CassandraHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/cassandra/CassandraHealthIndicator.java) | 检查Cassandra数据库是否已启动。        |
| [`CouchbaseHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/couchbase/CouchbaseHealthIndicator.java) | 检查Couchbase群集是否已启动。          |
| [`DiskSpaceHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/system/DiskSpaceHealthIndicator.java) | 检查磁盘空间不足。                     |
| [`DataSourceHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/jdbc/DataSourceHealthIndicator.java) | 检查是否可以获得与`DataSource`的连接。 |
| [`ElasticsearchHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/elasticsearch/ElasticsearchHealthIndicator.java) | 检查Elasticsearch集群是否已启动。      |
| [`InfluxDbHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/influx/InfluxDbHealthIndicator.java) | 检查InfluxDB服务器是否已启动。         |
| [`JmsHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/jms/JmsHealthIndicator.java) | 检查JMS代理是否已启动。                |
| [`MailHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/mail/MailHealthIndicator.java) | 检查邮件服务器是否已启动。             |
| [`MongoHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/mongo/MongoHealthIndicator.java) | 检查Mongo数据库是否已启动。            |
| [`Neo4jHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/neo4j/Neo4jHealthIndicator.java) | 检查Neo4j服务器是否已启动。            |
| [`RabbitHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/amqp/RabbitHealthIndicator.java) | 检查Rabbit服务器是否已启动。           |
| [`RedisHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/redis/RedisHealthIndicator.java) | 检查Redis服务器是否已启动。            |
| [`SolrHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/solr/SolrHealthIndicator.java) | 检查Solr服务器是否已启动。             |

#### 2.8.2 编写自定义 HealthIndicators

##### 2.8.2.1 实现  HealthIndicator 接口

- 实现`HealthIndicator`接口，根据自己的需要判断返回的状态是`UP`还是`DOWN`，功能简单
- 给定`HealthIndicator`的标识符是没有`HealthIndicator`后缀的bean的名称（如果存在）
- 以下示例中，健康信息在名为`my`的条目中可用

```java
@Component
public class MyHealthIndicator implements HealthIndicator {

    private static final String VERSION = "v1.0.0";
    @Override
    public Health health() {
        int code = check();
        if (code != 0) {
            Health.down().withDetail("code", code).withDetail("version", VERSION).build();
        }
        return Health.up().withDetail("code", code)
                .withDetail("version", VERSION).up().build();
    }
    private int check() {
        return 0;
    }
}
```

- 下表显示了内置状态的默认状态映射

| 状态           | 制图                                         |
| -------------- | -------------------------------------------- |
| DOWN           | SERVICE_UNAVAILABLE (503)                    |
| OUT_OF_SERVICE | SERVICE_UNAVAILABLE (503)                    |
| UP             | No mapping by default, so http status is 200 |
| UNKNOWN        | No mapping by default, so http status is 200 |

- **注：** 如果您需要更多控制权，可以定义自己的`HealthStatusHttpMapper` bean

##### 2.8.2.2 继承 AbstractHealthIndicator 抽象类

- 继承`AbstractHealthIndicator`抽象类，重写`doHealthCheck`方法，功能比第一种要强大一点点，默认的**DataSourceHealthIndicator 、 RedisHealthIndicator** 都是这种写法，内容回调中还做了异常的处理

```java
@Component("my2")
public class MyAbstractHealthIndicator extends AbstractHealthIndicator {

    private static final String VERSION = "v1.0.0";

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        int code = check();
        if (code != 0) {
            builder.down().withDetail("code", code).withDetail("version", VERSION).build();
        }
        builder.withDetail("code", code)
                .withDetail("version", VERSION).up().build();
    }

    private int check() {
        return 0;
    }
}
```



#### 2.8.3 Reactive Health Indicators

- 对于反应性应用程序，例如那些使用Spring WebFlux的应用程序，`ReactiveHealthIndicator`提供了一个非阻塞的接口来获取应用程序运行状况
- 与传统的 `HealthIndicator` 类似，健康信息从 [`ReactiveHealthIndicatorRegistry`](https://github.com/spring-projects/spring-boot/tree/v2.1.3.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/ReactiveHealthIndicatorRegistry.java) 的内容中收集
- 默认情况下，`ApplicationContext` 中定义的所有 [`HealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.3.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/HealthIndicator.java) 和 [`ReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.3.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/ReactiveHealthIndicator.java) 实例
- 不检查反应API 的 `Regular HealthIndicator `是在  elastic scheduler 上执行的

- **注：** 在响应式应用程序中，`ReactiveHealthIndicatorRegistry`可用于在运行时注册和取消注册运行状况指示器

##### 2.8.3.1 自定义 ReactiveHealthIndicator 

```java
@Component
public class MyReactiveHealthIndicator implements ReactiveHealthIndicator {

	@Override
	public Mono<Health> health() {
		return doHealthCheck() //perform some specific health check that returns a Mono<Health>
			.onErrorResume(ex -> Mono.just(new Health.Builder().down(ex).build())));
	}

}
```

- **注：** 要自动处理错误，请考虑从`AbstractReactiveHealthIndicator`扩展



##### 2.8.4 Auto-configured ReactiveHealthIndicators

- Spring Boot自动配置了以下`ReactiveHealthIndicators`

| 名称                                                         | 描述                            |
| ------------------------------------------------------------ | ------------------------------- |
| [`CassandraReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/cassandra/CassandraReactiveHealthIndicator.java) | 检查Cassandra数据库是否已启动。 |
| [`CouchbaseReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/couchbase/CouchbaseReactiveHealthIndicator.java) | 检查Couchbase群集是否已启动。   |
| [`MongoReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/mongo/MongoReactiveHealthIndicator.java) | 检查Mongo数据库是否已启动。     |
| [`RedisReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/redis/RedisReactiveHealthIndicator.java) | 检查Redis服务器是否已启动。     |

- If necessary, reactive indicators replace the regular ones
- Also, any `HealthIndicator` that is not handled explicitly is wrapped automatically



### 2.9 应用程序信息

- 应用程序信息公开了从`ApplicationContext`中定义的所有 beans 收集的各种信息

- Spring Boot包含一些自动配置的`InfoContributor` beans，您可以自己编写

#### 2.9.1  Auto-configured InfoContributors

- Spring Boot自动配置了以下`InfoContributor` beans：

| 名称                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`EnvironmentInfoContributor`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/EnvironmentInfoContributor.java) | 在`info`键下显示`Environment`中的任意键。                    |
| [`GitInfoContributor`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/GitInfoContributor.java) | 如果`git.properties`文件可用，则公开git信息。                |
| [`BuildInfoContributor`](https://github.com/spring-projects/spring-boot/tree/v2.1.1.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/BuildInfoContributor.java) | 如果`META-INF/build-info.properties`文件可用，则公开构建信息。 |

- 可以通过设置`management.info.defaults.enabled`属性来禁用所有

#### 2.9.2 自定义应用程序信息

- 您可以通过设置`info.*` Spring属性来自定义`info`端点公开的数据。`info`键下的所有`Environment`属性都会自动显示

```ini
info.app.encoding=UTF-8
info.app.java.source=1.8
info.app.java.target=1.8

# 您可以在构建时扩展信息属性，而不是对这些值进行硬编码 。
# 假设您使用Maven，您可以按如下方式重写前面的示例：
info.app.encoding=@project.build.sourceEncoding@
info.app.java.source=@java.version@
info.app.java.target=@java.version@
```

#### 2.9.3 Git提交信息

- `info`端点的另一个有用功能是它能够在构建项目时发布有关`git`源代码存储库状态的信息
- 如果`GitProperties` bean可用，则会公开`git.branch`，`git.commit.id`和`git.commit.time`属性
- 如果类路径的根目录中有`git.properties`文件，则自动配置`GitProperties` bean.有关更多详细[信息，](https://www.springcloud.cc/spring-boot.html#howto-git-info)请参阅“ [生成git信息](https://www.springcloud.cc/spring-boot.html#howto-git-info) ”

```ini
# 如果要显示完整的git信息（即git.properties的完整内容），请使用management.info.git.mode属性
management.info.git.mode=full
```

#### 2.9.4 构建信息

- 如果`BuildProperties` bean可用(如果类路径中有`META-INF/build-info.properties`文件)，`info`端点也可以发布有关您的构建的信息
- Maven和Gradle插件都可以生成该文件。有关更多详细[信息，](https://www.springcloud.cc/spring-boot.html#howto-build-info)请参阅“ [生成构建信息](https://www.springcloud.cc/spring-boot.html#howto-build-info) ”

#### 2.9.5 编写自定义InfoContributors

```java
import java.util.Collections;
import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;

@Component
public class ExampleInfoContributor implements InfoContributor {
	@Override
	public void contribute(Info.Builder builder) {
		builder.withDetail("example",
				Collections.singletonMap("key", "value"));
	}
}
```



## 3 **通过HTTP进行监控和管理**

- 如果您正在开发Web应用程序，Spring Boot Actuator会自动配置所有已启用的端点以通过HTTP公开
- 默认约定是使用端点`id`作为URL路径，前缀为`/actuator`。例如，`health`暴露为`/actuator/health`
- **注：**Actuator本身支持Spring MVC，Spring WebFlux和Jersey

### 3.1 自定义管理端点路径

#### 3.1.1 更改管理端点路径

```ini
# 这样 /actuator/{id} 被更改为 /manage/{id}
management.endpoints.web.base-path=/manage
```

- 除非已将管理端口配置为[使用其他HTTP端口公开端点](https://www.springcloud.cc/spring-boot.html#production-ready-customizing-management-server-port)，否则 `management.endpoints.web.base-path`相对于`server.servlet.context-path`
- 如果配置了`management.server.port`，则`management.endpoints.web.base-path`相对于`management.server.servlet.context-path`

- 如果要将端点映射到其他路径，可以使用`management.endpoints.web.path-mapping`属性

```ini
# 将 /actuator/health 映射到 /healthcheck,注，此时 发现页面不可用！
management.endpoints.web.base-path=/
management.endpoints.web.path-mapping.health=healthcheck
```



### 3.2  自定义Management Server端口

- 使用默认HTTP端口公开管理端点是基于云的部署的明智选择
- 但是，如果您的应用程序在您自己的数据中心内运行，您可能更喜欢使用不同的HTTP端口公开端点

```ini
management.server.port=8081
```



### 3.3 配置管理特定的SSL

- 配置为使用自定义端口时，还可以使用各种`management.server.ssl.*`属性为管理服务器配置自己的SSL

```ini
# 使用HTTPS时管理服务器通过HTTP可用
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=classpath:store.jks
server.ssl.key-password=secret
management.server.port=8080
management.server.ssl.enabled=false

# 主服务器和管理服务器都可以使用SSL但具有不同的密钥库
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=classpath:main.jks
server.ssl.key-password=secret
management.server.port=8080
management.server.ssl.enabled=true
management.server.ssl.key-store=classpath:management.jks
management.server.ssl.key-password=secret
```



### 3.4 自定义管理服务地址

- 您可以通过设置`management.server.address`属性来自定义管理端点可用的地址
- 如果您只想在内部或面向运行的网络上侦听或仅侦听来自`localhost`的连接，这样做非常有用

- **注：** 仅当端口与主服务器端口不同时，才能侦听不同的地址

```ini
# 不允许远程管理连接
management.server.port=8081
management.server.address=127.0.0.1
```



### 3.5 禁用HTTP端点

- 如果您不想通过HTTP公开端点，可以将管理端口设置为`-1`

```ini
management.server.port=-1
# 等同于
management.endpoints.web.exposure.exclude=*
```



## 4 **对JMX的监测和管理**

- Java Management Extensions（JMX）提供了一种监视和管理应用程序的标准机制
- 默认情况下，Spring Boot将管理端点公开为`org.springframework.boot`域下的JMX MBean

### 4.1 自定义MBean名称

- MBean的名称通常是从端点的`id`生成的。例如，`health`端点公开为`org.springframework.boot:type=Endpoint,name=Health`
- 如果您的应用程序包含多个Spring `ApplicationContext`，您可能会发现名称发生冲突。要解决此问题，可以将`spring.jmx.unique-names`属性设置为`true`，以便MBean名称始终是唯一的
- 您还可以自定义公开端点的JMX域

```ini
spring.jmx.unique-names=true
management.endpoints.jmx.domain=com.example.myapp
```

### 4.2 禁用JMX端点

```ini
# 如果您不想通过JMX公开端点
management.endpoints.jmx.exposure.exclude=*
```

### 4.3 通过 HTTP 使用 Jolokia 访问 JMX

- Jolokia 是一个 JMX-HTTP bridge，它提供了一种访问 JMX bean 的新方式

```xml
<dependency>
	<groupId>org.jolokia</groupId>
	<artifactId>jolokia-core</artifactId>
</dependency>
```

- 然后，可以通过向`management.endpoints.web.exposure.include`属性添加`jolokia`或`*`来公开Jolokia端点
- 然后，您可以在管理HTTP服务器上使用`/actuator/jolokia`来访问它

#### 4.3.1 自定义Jolokia

- Jolokia有许多设置，您可以通过设置servlet参数来进行传统配置

```ini
management.endpoint.jolokia.config.debug=true
```

#### 4.3.2 禁用Jolokia

```ini
# 如果您使用Jolokia但不希望Spring Boot配置它
management.endpoint.jolokia.enabled=false
```



## 5 **日志记录器**

- Spring Boot Actuator 有可在运行时 **查看和配置** 应用程序日志级别的功能
- 你可以查看全部或单个日志记录器的配置
- 该配置由显式配置的日志记录级别以及日志记录框架为其提供的有效日志记录级别组成

```java
TRACE、DEBUG、INFO、WARN、ERROR、FATAL、OFF、null(表示没有显式配置)
```

### 5.1 配置一个日志记录器

- 要配置给定的记录器，`POST`是资源URI的部分实体，如以下示例所示：

```json
{"configuredLevel": "DEBUG"}
```

- **注：** 要“重置”记录器的特定级别（并使用默认配置），您可以传递`null`的值`configuredLevel`



## 6 **Metrics**

- Spring Boot Actuator为[Micrometer](https://micrometer.io/)提供依赖关系管理和自动配置
- [Micrometer](https://micrometer.io/)是一个支持众多监控系统的应用程序指标外观，包括：
  - [AppOptics](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-appoptics)
  - [Atlas](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-atlas)
  - [Datadog](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-datadog)
  - [Dynatrace](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-dynatrace)
  - [Elastic](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-dynatrace)
  - [Ganglia](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-ganglia)
  - [Graphite](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-graphite)
  - [Humio](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-humio)
  - [Influx](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-influx)
  - [JMX](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-jmx)
  - [KairosDB](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-kairos)
  - [New Relic](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-newrelic)
  - [Prometheus](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-prometheus)
  - [SignalFx](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-signalfx)
  - [简单（内存中）](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-simple)
  - [StatsD](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-statsd)
  - [Wavefront](https://www.springcloud.cc/spring-boot.html#production-ready-metrics-export-wavefront)

- 要了解有关Micrometer功能的更多信息，请参阅其 [参考文档](https://micrometer.io/docs)，特别是 [概念部分](https://micrometer.io/docs/concepts)

### 6.1 入门

- Spring Boot 自动配置了一个 `Composite MeterRegistry`，并且将所有容器中的 `MeterRegistry` 添加到其中

- 在运行时，只需要 classpath 中有 `micrometer-registry-{system}` 依赖即可让 Spring Boot 配置该注册表



- 大部分注册表都有共同点，即使 Micrometer 注册实现位于 classpath 上，你也可以禁用特定的注册表。例如，要禁用 Datadog

```ini
management.metrics.export.datadog.enabled=false
```

- Spring Boot 还会将所有自动配置的注册表添加到 `Metrics` 类的全局静态复合注册表中，除非你明确禁止：

```ini
management.metrics.use-global-registry=false
```

- 在注册表中注册任何指标之前，你可以注册任意数量的 `MeterRegistryCustomizer` bean 以进一步配置注册表，例如通用标签：

```java
@Bean
MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
    return registry -> registry.config().commonTags("region", "us-east-1");
}
```

- 你可以通过指定泛型类型，自定义注册表实现：

```java
@Bean
MeterRegistryCustomizer<GraphiteMeterRegistry> graphiteMetricsNamingConvention() {
    return registry -> registry.config().namingConvention(MY_CUSTOM_CONVENTION);
}
```

- 你可以在组件中注入 `MeterRegistry` 并注册指标

```java
@Component
public class SampleBean {

    private final Counter counter;
	//SampleBean 相当于一个 Counter，通过此构造函数将其注册到 registry 中
    public SampleBean(MeterRegistry registry) {
        this.counter = registry.counter("received.messages");
    }

    public void handleMessage(String message) {
        this.counter.increment();
        // 处理消息实现
    }
}
```

- **注：** Spring Boot 还[配置内置的测量工具](https://docs.spring.io/spring-boot/docs/2.1.4.RELEASE/reference/htmlsingle/#production-ready-metrics-meter)（即 `MeterBinder` 实现），你可以通过配置或专用注解标记来控制。



### 6.2 支持的监控系统

#### 6.2.1 AppOptics

- 默认情况下，AppOptics注册表[会](https://api.appoptics.com/v1/measurements)定期将指标推送到 [api.appoptics.com/v1/measurements](https://api.appoptics.com/v1/measurements)
- 要将指标导出到SaaS [AppOptics](http://micrometer.io/docs/registry/appoptics)，必须提供您的API令牌：

```ini
management.metrics.export.appoptics.api-token=YOUR_TOKEN
```

#### 6.2.2 Atlas

- 默认情况下，度量标准导出到 本地计算机上运行的[Atlas](http://micrometer.io/docs/registry/atlas)

```ini
# 使用以下方式提供要使用的Atlas服务器的位置
management.metrics.export.atlas.uri=http://atlas.example.com:7101/api/v1/publish
```

#### 6.2.3 Datadog

- Datadog注册表定期将指标推送到[datadoghq](https://www.datadoghq.com/)

```ini
# 要将指标导出到Datadog，必须提供您的API密钥
management.metrics.export.datadog.api-key=YOUR_KEY
# 您还可以更改度量标准发送到Datadog的时间间隔
management.metrics.export.datadog.step=30s
```

#### 6.2.4 Dynatrace

- Dynatrace注册表定期将指标推送到配置的URI

```ini
# 要将指标导出到 Dynatrace，必须提供您的API令牌，设备ID和URI
management.metrics.export.dynatrace.api-token=YOUR_TOKEN
management.metrics.export.dynatrace.device-id=YOUR_DEVICE_ID
management.metrics.export.dynatrace.uri=YOUR_URI
# 您还可以更改指标发送到Dynatrace的时间间隔
management.metrics.export.dynatrace.step=30s
```

#### 6.2.5 Elastic

- 默认情况下，指标会导出到 本地计算机上运行的[Elastic](http://micrometer.io/docs/registry/elastic)

```ini
# 可以使用以下属性提供要使用的Elastic服务器的位置
management.metrics.export.elastic.host=http://elastic.example.com:8086
```

#### 6.2.6 Ganglia

- 默认情况下，度量标准将导出到 本地计算机上运行的[Ganglia](http://micrometer.io/docs/registry/ganglia)

```ini
# 可以使用以下命令提供要使用的Ganglia服务器主机和端口
management.metrics.export.ganglia.host=ganglia.example.com
management.metrics.export.ganglia.port=9649
```

#### 6.2.7 Graphite

- 默认情况下，度量标准将导出到 本地计算机上运行的[Graphite](http://micrometer.io/docs/registry/graphite)

```ini
# 可以使用以下命令提供要使用的Graphite服务器主机和端口
management.metrics.export.graphite.host=graphite.example.com
management.metrics.export.graphite.port=9004
```

- Micrometer 提供了一个默认的 `HierarchicalNameMapper`，它管理维度计数器 id 如何[映射到平面分层名称](https://micrometer.io/docs/registry/graphite#_hierarchical_name_mapping)
- **注：**要控制此行为，请定义 `GraphiteMeterRegistry` 并提供自己的 `HierarchicalNameMapper`。除非你自己定义，否则使用自动配置的 `GraphiteConfig` 和 `Clock` bean

```java
@Bean
public GraphiteMeterRegistry graphiteMeterRegistry(GraphiteConfig config, Clock clock) {
    return new GraphiteMeterRegistry(config, clock, MY_HIERARCHICAL_MAPPER);
}
```

#### 6.2.8 Humio

- 默认情况下，Humio 注册表会定期将指标推送到 [cloud.humio.com](https://cloud.humio.com/)

```ini
# 要将指标导出到 SaaS Humio，你必须提供 API 令牌
management.metrics.export.humio.api-token=YOUR_TOKEN
# 你还应配置一个或多个标记，以标识要推送指标的数据源
management.metrics.export.humio.tags.alpha=a
management.metrics.export.humio.tags.bravo=b
```

#### 6.2.9 Influx

- 默认情况下，度量将导出到本地的 [Influx](https://micrometer.io/docs/registry/influx)

```ini
# 要指定 Influx 服务器的位置，可以使用
management.metrics.export.influx.uri=https://influx.example.com:8086
```

#### 6.2.10 JMX

- Micrometer 提供了与 [JMX](https://micrometer.io/docs/registry/jmx) 的分层映射，主要为了方便在本地查看指标且可移植
- 默认情况下，度量将导出到 `metrics` JMX 域

```ini
# 可以使用以下方式提供要使用的域
management.metrics.export.jmx.domain=com.example.app.metrics
```

- Micrometer 提供了一个默认的 `HierarchicalNameMapper`，它管理维度计数器 id 如何[映射到平面分层名称](https://micrometer.io/docs/registry/jmx#_hierarchical_name_mapping)。
- 要控制此行为，请定义 `JmxMeterRegistry` 并提供自己的 `HierarchicalNameMapper`。除非你自己定义，否则使用自动配置的 `JmxConfig` 和 `Clock` bean

```java
@Bean
public JmxMeterRegistry jmxMeterRegistry(JmxConfig config, Clock clock) {
    return new JmxMeterRegistry(config, clock, MY_HIERARCHICAL_MAPPER);
}
```

#### 6.2.11 KairosDB

- 默认情况下，度量将导出到本地的 [KairosDB](https://micrometer.io/docs/registry/kairos)

```ini
# 可以使用以下方式提供 KairosDB 服务器的位置
management.metrics.export.kairos.uri=https://kairosdb.example.com:8080/api/v1/datapoints
```

#### 6.2.12 New Relic

- New Relic 注册表定期将指标推送到 [New Relic](https://micrometer.io/docs/registry/new-relic)

```ini
# 要将指标导出到 New Relic，你必须提供 API 密钥和帐户 ID
management.metrics.export.newrelic.api-key=YOUR_KEY
management.metrics.export.newrelic.account-id=YOUR_ACCOUNT_ID
# 你还可以更改将度量发送到 New Relic 的间隔时间
management.metrics.export.newrelic.step=30s
```

#### 6.2.13 **Prometheus**

- [Prometheus](https://micrometer.io/docs/registry/prometheus) 会抓取或轮询各个应用实例以获取指标数据

- Spring Boot 在 `/actuator/prometheus` 上提供 actuator 端点，以适当的格式呈现 [Prometheus scrape ](https://prometheus.io/)
- **注：** 默认情况下端点不可用，必须暴露，请参阅[暴露端点](https://docshome.gitbooks.io/springboot/content/pages/production-ready.html#production-ready-endpoints-exposing-endpoints)以获取更多详细信息
- 以下是要添加到 `prometheus.yml` 的示例 `scrape_config`

```yaml
scrape_configs:
  - job_name: 'spring'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['HOST:PORT']
```

#### 6.2.14 SignalFx

- SignalFx 注册表定期将指标推送到 [SignalFx](https://micrometer.io/docs/registry/signalfx)

```ini
# 要将指标导出到 SignalFx，你必须提供访问令牌
management.metrics.export.signalfx.access-token=YOUR_ACCESS_TOKEN
# 你还可以更改将指标发送到 SignalFx 的间隔时间
management.metrics.export.signalfx.step=30s
```

#### 6.2.15 Simple

- Micrometer 附带一个简单的内存后端，如果没有配置其他注册表，它将自动用作后备
- 这使你可以查看[指标端点](https://docshome.gitbooks.io/springboot/content/pages/production-ready.html#production-ready-metrics-endpoint)中收集的指标信息

```ini
# 只要你使用了任何其他可用的后端，内存后端就会自动禁用。你也可以显式禁用它
management.metrics.export.simple.enabled=false
```

#### 6.2.16 StatsD

- StatsD 注册表将 UDP 上的指标推送到 [StatsD](https://micrometer.io/docs/registry/statsd) 代理

- 默认情况下，metrics  将导出到本地的 StatsD 代理

```ini
# 可以使用以下方式提供要使用的StatsD代理主机和端口
management.metrics.export.statsd.host=statsd.example.com
management.metrics.export.statsd.port=9125
# 您还可以更改要使用的StatsD行协议（默认为Datadog）
management.metrics.export.statsd.flavor=etsy
```

#### 6.2.17 Wavefront

- Wavefront注册表会定期将指标推送到 [Wavefront](http://micrometer.io/docs/registry/wavefront)

```ini
# 如果您要将指标直接导出到Wavefront，则必须提供您的API令牌
management.metrics.export.wavefront.api-token=YOUR_API_TOKEN
# 你可以在环境中使用 Wavefront sidecar 或内部代理设置，将指标数据转发到 Wavefront API 主机
management.metrics.export.wavefront.uri=proxy://localhost:2878

# 你还可以更改将指标发送到 Wavefront 的间隔时间
management.metrics.export.wavefront.step=30s
```

- **注：** 如果将度量标准发布到Wavefront代理（如[文档](https://docs.wavefront.com/proxies_installing.html)中 [所述](https://docs.wavefront.com/proxies_installing.html)），则主机必须采用`proxy://HOST:PORT`格式



### 6.3 **支持的 metrics**

- Spring Boot 将自动配置 以下核心指标
  - JVM 指标，报告利用率：
    - 各种内存和缓冲池
    - 与垃圾回收有关的统计
    - 线程利用率
    - 加载/卸载 class 的数量
  - CPU 指标
  - 文件描述符指标
  - Kafka 消费者指标
  - Log4j2 指标：记录每个级别记录到 Log4j2 的事件数
  - Logback 指标：记录每个级别记录到 Logback 的事件数
  - 正常运行时间指标：报告正常运行时间和表示应用程序绝对启动时间的固定计量值
  - Tomcat 指标
  - [Spring Integration](https://docs.spring.io/spring-integration/docs/current/reference/html/system-management-chapter.html#micrometer-integration)指标

#### 6.3.1 Spring MVC Metrics

- 自动配置可以对Spring MVC处理的请求进行检测
- 当`management.metrics.web.server.auto-time-requests`为`true`时，将对所有请求进行此检测
- 或者，当设置为`false`时，您可以通过将`@Timed`添加到请求处理方法来启用检测

```java
@RestController
@Timed // 1
public class MyController {

	@GetMapping("/api/people")
	@Timed(extraTags = { "region", "us-east-1" })  // 2
	@Timed(value = "all.people", longTask = true)  // 3
	public List<Person> listPeople() { ... }
}
```

1. 一个 Controller，为其中的每个请求处理程序启用计时
2. 启用单个端点。如果你在类上使用了它，就不需要在方法上再次声明，但可以用它来进一步自定义该特定端点的计时器
3. 使用 `longTask = true` 的方法为该方法启用长任务计时器。长任务计时器需要单独的指标名称，并且可以使用短任务计时器进行堆叠

- 默认情况下，使用名称`http.server.requests`生成指标
- 可以通过设置`management.metrics.web.server.requests-metric-name`属性来自定义名称

- 默认情况下，Spring与MVC相关的指标标记有以下信息

| 标签        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `exception` | 处理请求时抛出的任何异常的简单类名。                         |
| `method`    | 请求的方法（例如，`GET`或`POST`）                            |
| `outcome`   | 根据响应的状态代码请求结果。1xx是`INFORMATIONAL`，2xx是`SUCCESS`，3xx是`REDIRECTION`，4xx `CLIENT_ERROR`，5xx是`SERVER_ERROR` |
| `status`    | 响应的HTTP状态代码（例如，`200`或`500`）                     |
| `uri`       | 如果可能，在变量替换之前请求URI模板（例如，`/api/person/{id}`） |

- 要自定义标记，请提供实现`WebMvcTagsProvider`的`@Bean`

#### 6.3.2 Spring WebFlux Metrics 

- 自动配置支持WebFlux控制器和功能处理程序处理的所有请求的检测
- 默认情况下，会生成名称为`http.server.requests`的指标
- 您可以通过设置`management.metrics.web.server.requests-metric-name`属性来自定义名称

- 默认情况下，与WebFlux相关的指标标记有以下信息

| 标签        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `exception` | 处理请求时抛出的任何异常的简单类名。                         |
| `method`    | 请求的方法（例如，`GET`或`POST`）                            |
| `outcome`   | 根据响应的状态代码请求结果。1xx是`INFORMATIONAL`，2xx是`SUCCESS`，3xx是`REDIRECTION`，4xx `CLIENT_ERROR`，5xx是`SERVER_ERROR` |
| `status`    | 响应的HTTP状态代码（例如，`200`或`500`）                     |
| `uri`       | 如果可能，在变量替换之前请求URI模板（例如，`/api/person/{id}`） |

- 要自定义标记，请提供实现`WebFluxTagsProvider`的`@Bean`

#### 6.3.3 Jersey Server Metrics

- 自动配置支持对Jersey JAX-RS实现处理的请求进行检测
- 当 `management.metrics.web.server.auto-time-requests` 为 `true` 时，将对所有请求进行该项检测
- 当设置为 `false` 时，你可以通过将 `@Timed` 添加到请求处理方法上来启用检测

```java
@Component
@Path("/api/people")
@Timed // <1>
public class Endpoint {
    @GET
    @Timed(extraTags = { "region", "us-east-1" }) // <2>
    @Timed(value = "all.people", longTask = true) // <3>
    public List<Person> listPeople() { ... }
}
```

1. 在资源类上，为资源中的每个请求处理程序启用计时。
2. 在方法上则启用单个端点。如果你在类上使用了它，则不需在方法上再次声明，但可以用它来进一步自定义该特定端点的计时器。
3. 在有 `longTask = true` 的方法上，为该方法启用长任务计时器。长任务计时器需要单独的度量名称，并且可以使用短任务计时器进行堆叠。

- 默认情况下，使用名为 `http.server.requests` 生成度量指标
- 可以通过设置 `management.metrics.web.server.requests-metric-name` 属性来自定义名称。

- 默认情况下，Jersey 服务器指标使用以下标签标记

| 标签        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `exception` | 处理请求时抛出的异常的简单类名。                             |
| `method`    | 请求的方法（例如，`GET` 或 `POST`）                          |
| `outcome`   | 根据响应状态码生成的请求结果。1xx 是 `INFORMATIONAL`，2xx 是 `SUCCESS`，3xx 是 `REDIRECTION`，4xx 是 `CLIENT_ERROR`，5xx 是 `SERVER_ERROR` |
| `status`    | 响应的 HTTP 状态码（例如，`200`或 `500`）                    |
| `uri`       | 如果可能，在变量替换之前请求 URI 模板（例如，`/api/person/{id}`） |

- 要自定义标签，请提供一个实现了 `JerseyTagsProvider` 的 `@Bean`

#### 6.3.4 HTTP Client Metrics

- Spring Boot Actuator 管理 RestTemplate 和 WebClient 的指标记录
- 为此，你必须注入一个自动配置的 builder 并使用它来创建实例
  - `RestTemplateBuilder` 用于 `RestTemplate`
  - `WebClient.Builder` 用于 `WebClient`
- 也可以手动指定负责此指标记录的自定义程序，即 `MetricsRestTemplateCustomizer` 和 `MetricsWebClientCustomizer`
- 默认情况下，使用名为 `http.client.requests` 生成度量指标

- 可以通过设置 `management.metrics.web.client.requests-metric-name` 属性来自定义名称



- 默认情况下，已指标记录客户端生成的度量指标使用以下标签标记
  - `method`，请求的方法（例如，`GET`或 `POST`）
  - `uri`，变量替换之前的请求 URI 模板（如果可能）（例如，`/api/person/{id}`）
  - `status`，响应的 HTTP 状态码（例如，`200` 或 `500`）
  - `clientName`，URI 的主机部分 
- 要根据你选择的客户端自定义标签，你可以提供一个实现了 `RestTemplateExchangeTagsProvider` 或 `WebClientExchangeTagsProvider` 的 `@Bean`
- `RestTemplateExchangeTags` 和 `WebClientExchangeTags` 中有便捷的静态函数

#### 6.3.5 Cache Metrics

- 自动配置允许在启动时使用前缀为`cache`的度量标准检测所有可用的`Cache`
- 缓存检测针对一组基本指标进行了标准化。此外，还提供了特定于缓存的指标。

- 支持以下缓存库
  - Caffeine
  - EhCache 2
  - Hazelcast
  - 任何兼容的JCache（JSR-107）实现
- 度量标准由缓存的名称和从bean名称派生的`CacheManager`的名称标记
- **注：** 只有启动时可用的缓存才会绑定到注册表。对于在启动阶段之后即时或以编程方式创建的缓存，需要显式注册。`CacheMetricsRegistrar` bean可用于简化此过程

#### 6.3.6 DataSource Metrics

- 自动配置使用名为`jdbc`的度量标准启用所有可用`DataSource`对象的检测
- 数据源检测会生成表示池中当前活动，最大允许和最小允许连接的计量器
- 这些仪表中的每一个都有一个以`jdbc`为前缀的名称
- 度量标准也由基于bean名称计算的`DataSource`的名称标记
- **注：** 默认情况下，Spring Boot为所有支持的数据源提供元数据; 如果您不喜欢自己喜欢的数据源，则可以添加额外的`DataSourcePoolMetadataProvider` beans。有关示例，请参阅`DataSourcePoolMetadataProvidersConfiguration`
- 此外，Hikari特定的指标以`hikaricp`前缀公开。每个度量标准都由池名称标记（可以使用`spring.datasource.name`控制）

#### 6.3.7 Hibernate Metrics

- 自动配置允许使用名为`hibernate`的度量标准启用统计信息的所有可用Hibernate `EntityManagerFactory`实例的检测
- 度量标准也由bean名称派生的`EntityManagerFactory`名称标记
- 要启用统计信息，标准JPA属性`hibernate.generate_statistics`必须设置为`true`。您可以在自动配置的`EntityManagerFactory`上启用它

```ini
spring.jpa.properties.hibernate.generate_statistics=true
```

#### 6.3.8 RabbitMQ Metrics

- 自动配置将使用名为`rabbitmq`的度量标准启用所有可用RabbitMQ连接工厂的检测

### 6.4 注册自定义 Metrics

- To register custom metrics, inject `MeterRegistry` into your component, as shown in the following example

```java
class Dictionary {

	private final List<String> words = new CopyOnWriteArrayList<>();
	//将 gaugeCollectionSize 注册的 registry 中
	Dictionary(MeterRegistry registry) {
		registry.gaugeCollectionSize("dictionary.size", Tags.empty(), this.words);
	}
	// …
}
```

- 如果您发现跨组件或应用程序重复检测一套度量标准，则可以将此套件封装在`MeterBinder`实现中
- 默认情况下，所有`MeterBinder` beans的指标都会自动绑定到所有 spring 管理的 MeterRegistry 上



### 6.5 自定义 MetricFilter

- 默认情况下，所有 `MeterFilter` bean 都将自动应用于 `MeterRegistry.Config`

```java
//将 mytag.region 标记重命名为 mytag.area 以获取以 com.example 开头的所有 meter ID
@Bean
public MeterFilter renameRegionTagMeterFilter() {
    return MeterFilter.renameTag("com.example", "mytag.region", "mytag.area");
}
```

#### 6.5.1 Common tags

- 通用标签通常用于操作环境中的维度，例如主机、实例、区域、堆栈等。通用标签应用于所有 meter，并且可以按照以下示例进行配置

```ini
# 将region和stack标记添加到所有 meter，其值分别为us-east-1和prod
management.metrics.tags.region=us-east-1
management.metrics.tags.stack=prod
```

- 如果您使用Graphite，则常用标记的顺序很重要。由于使用此方法无法保证常用标记的顺序，因此建议Graphite用户定义自定义`MeterFilter`

#### 6.5.2 Per-meter 属性

- 除了 `MeterFilter` bean 之外，还可以使用 properties 在 per-meter 基础上自定义。Per-meter 定义适用于以给定名称开头的所有 meter ID。例如，以下将禁用任何以 `example.remote` 开头的 ID 的 meter

```ini
management.metrics.enable.example.remote=false
```

- per-meter 自定义的配置

| Property                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `management.metrics.enable`                                  | 是否拒绝 meter 发布任何指标                                  |
| `management.metrics.distribution.percentiles-histogram`      | 是否发布一个适用于计算可聚合（跨维度）的百分比近似柱状图     |
| `management.metrics.distribution.minimum-expected-value`, `management.metrics.distribution.maximum-expected-value` | 通过限制预期值的范围来发布较少的柱状图桶                     |
| `management.metrics.distribution.percentiles`                | top99 范围数组，存储的是一个范围的百分百                     |
| `management.metrics.distribution.sla`                        | sla 范围数组，存储的是一个范围的上下界，可通过此数组计算 top99 |

- 有关`percentiles-histogram`，`percentiles`和`sla`背后概念的更多详细信息,请参阅[histogarm-distribution](https://micrometer.io/docs/concepts#_histograms_and_percentiles)



### 6.6 **Metrics endpoint**

- Spring Boot 提供了一个 `metrics` 端点，可以在诊断中用于检查应用程序收集的度量指标
- 默认情况下端点不可用，必须公开，请参阅[公开端点](https://www.springcloud.cc/spring-boot.html#production-ready-endpoints-exposing-endpoints)以获取更多详细信息
- 访问 `/actuator/metrics` 会显示可用的 meter 名称列表
- 你可以查看某一个 meter 的信息，方法是将其名称作为选择器，例如，`/actuator/metrics/jvm.memory.max`
- **注：** 你在此处使用的名称应与代码中使用的名称相匹配，而不是在命名约定规范化后的名称 —— 为了发送到监控系统。换句话说，如果 `jvm.memory.max` 由于 Prometheus 命名约定而显示为 `jvm_memory_max`，则在审查度量指标端点中的 meter 时，应仍使用 `jvm.memory.max` 作为选择器
- 你还可以在 URL 的末尾添加任意数量的 `tag=KEY:VALUE` 查询参数，以便多维度获取 meter，例如 `/actuator/metrics/jvm.memory.max?tag=area:nonheap`

- **注：** 报告的测量值是与 meter 名称和已应用的任何标签匹配的所有 meter 的统计数据的总和。因此，在上面的示例中，返回的 Value 统计信息是堆的 Code Cache，Compressed Class Space 和 Metaspace 区域的最大内存占用量的总和。如果你只想查看 Metaspace 的最大大小，可以添加一个额外的 `tag=id:Metaspace`，即 `/actuator/metrics/jvm.memory.max?tag=area:nonheap&tag=id:Metaspace`



## 7 审计

一旦 Spring Security 生效，Spring Boot Actuator 就拥有一个灵活的审计框架，它可以发布事件（默认情况下，`authentication success`、`failure` 和 `access denied` 例外）。此功能对事件报告和基于身份验证失败实现一个锁定策略非常有用。要自定义发布的安全事件，你可以提供自己的 `AbstractAuthenticationAuditListener` 和 `AbstractAuthorizationAuditListener` 实现。

你还可以将审计服务用于自己的业务事件。为此，请将现有的 `AuditEventRepository` 注入自己的组件并直接使用它或使用 Spring `ApplicationEventPublisher`（通过实现 `ApplicationEventPublisherAware`）发布 `AuditApplicationEvent`。



## 8 HTTP 追踪

- 所有 HTTP 请求将自动启用追踪功能。你可以通过查看 `httptrace` 端点来获取最近相关的 100 个请求响应信息

### 8.1 自定义 HTTP 追踪

- 要自定义每个追踪信息中包含的项，请使用 `management.trace.http.include` 属性配置。对于高级自定义，请考虑注册自己的 `HttpExchangeTracer` 实现
- 默认情况下，使用一个 `InMemoryHttpTraceRepository` 存储最新的 100 个请求响应信息。如果需要扩展容量，可定义自己的 `InMemoryHttpTraceRepository` bean 实例。你还可以创建自己的 `HttpTraceRepository` 实现来替代默认配置



## 9 Process Monitoring

- 在`spring-boot`模块中，您可以找到两个类来创建通常对进程监视有用的文件
  - `ApplicationPidFileWriter`创建一个包含应用程序PID的文件（默认情况下，在应用程序目录中，文件名为`application.pid`）
  - `WebServerPortFileWriter`创建一个包含正在运行的Web服务器端口的文件（默认情况下，在文件名为`application.port`的应用程序目录中）

- 默认情况下，这些编写器未激活，但您可以启用
  - 9.1
  - 9.2

### 9.1 扩展配置

- 在`META-INF/spring.factories`文件中，您可以激活写入PID文件的侦听器，如以下示例所示

```fa
org.springframework.context.ApplicationListener=\
org.springframework.boot.context.ApplicationPidFileWriter,\
org.springframework.boot.web.context.WebServerPortFileWriter
```

### 9.2 以编程方式

- 您还可以通过调用`SpringApplication.addListeners(…)`方法并传递相应的`Writer`对象来激活侦听器
- 此方法还允许您在`Writer`构造函数中自定义文件名和路径



## 10 Cloud Foundry支持

- Spring Boot的执行器模块包括在部署到兼容的Cloud Foundry实例时激活的其他支持
- `/cloudfoundryapplication`路径为所有`@Endpoint` beans提供了另一条安全路线

- 通过扩展支持，可以使用Spring Boot执行器信息扩充Cloud Foundry管理UI（例如可用于查看已部署应用程序的Web应用程序）。例如，应用程序状态页面可以包括完整的健康信息，而不是典型的“运行”或“停止”状态
- **注：** 常规用户无法直接访问`/cloudfoundryapplication`路径。为了使用端点，必须与请求一起传递有效的UAA令牌

### 10.1 禁用Extended Cloud Foundry Actuator支持

- 如果要完全禁用`/cloudfoundryapplication`端点，可以将以下设置添加到`application.properties`文件中

```ini
management.cloudfoundry.enabled=false
```

### 10.2 Cloud Foundry自签名证书

- 默认情况下，`/cloudfoundryapplication`端点的安全验证会对各种Cloud Foundry服务进行SSL调用
- 如果您的Cloud Foundry UAA或Cloud Controller服务使用自签名证书，则需要设置以下属性

```ini
management.cloudfoundry.skip-ssl-validation=true
```

### 10.3 自定义上下文路径

- 如果服务器的上下文路径已配置为`/`以外的任何其他内容，则Cloud Foundry端点将不会在应用程序的根目录中可用
- 例如，如果`server.servlet.context-path=/app`，Cloud Foundry端点将在`/app/cloudfoundryapplication/*`处可用

- 如果您希望Cloud Foundry端点始终在`/cloudfoundryapplication/*`处可用，则无论服务器的上下文路径如何，您都需要在应用程序中明确配置它
- 配置将根据使用的Web服务器而有所不同。对于Tomcat，可以添加以下配置

```java
@Bean
public TomcatServletWebServerFactory servletWebServerFactory() {
	return new TomcatServletWebServerFactory() {

		@Override
		protected void prepareContext(Host host,
				ServletContextInitializer[] initializers) {
			super.prepareContext(host, initializers);
			StandardContext child = new StandardContext();
			child.addLifecycleListener(new Tomcat.FixContextListener());
			child.setPath("/cloudfoundryapplication");
			ServletContainerInitializer initializer = getServletContextInitializer(
					getContextPath());
			child.addServletContainerInitializer(initializer, Collections.emptySet());
			child.setCrossContext(true);
			host.addChild(child);
		}

	};
}

private ServletContainerInitializer getServletContextInitializer(String contextPath) {
	return (c, context) -> {
		Servlet servlet = new GenericServlet() {

			@Override
			public void service(ServletRequest req, ServletResponse res)
					throws ServletException, IOException {
				ServletContext context = req.getServletContext()
						.getContext(contextPath);
				context.getRequestDispatcher("/cloudfoundryapplication").forward(req,
						res);
			}
		};
		context.addServlet("cloudfoundry", servlet).addMapping("/*");
	};
}
```



## 参考

[SpringBoot 2.1.1.RELEASE 官方文档](<https://docs.spring.io/spring-boot/docs/2.1.1.RELEASE/reference/html/>)

[spring-boot-actuator-api-2.1.1.RELEASE](<https://docs.spring.io/spring-boot/docs/2.1.1.RELEASE/actuator-api/html/#log-file>)

[SpringBoot 2.1.1.RELEASE 中文文档](<https://www.springcloud.cc/spring-boot.html#production-ready-customizing-management-server-port>)

[SpringBoot 2.x 中文文档](<https://docshome.gitbooks.io/springboot/content/pages/production-ready.html#production-ready>)

[自定义 HealthIndicators](<http://cmsblogs.com/?p=2955>)

[Spring Boot Actuator: Production-ready Features](<https://docs.spring.io/spring-boot/docs/2.2.1.RELEASE/reference/html/production-ready-features.html#production-ready> )