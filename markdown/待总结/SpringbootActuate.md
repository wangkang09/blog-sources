[TOC]



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







<https://www.cnblogs.com/yanfei1819/p/10900613.html>

<http://cmsblogs.com/?p=2955>

<https://www.cnblogs.com/yanfei1819/p/11226397.html>

<https://docshome.gitbooks.io/springboot/content/pages/production-ready.html#production-ready>



<https://www.springcloud.cc/spring-boot.html>



<https://docs.spring.io/spring-boot/docs/2.2.1.RELEASE/reference/html/production-ready-features.html#production-ready> 官方