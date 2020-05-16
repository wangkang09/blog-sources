[TOC]

### 1 springboot 1.x 

#### 1.1 相关配置参数

```java
@Value("${server.context-path:/}")
private String contextPath1;
@Value("${server.servlet-path:/}")
private String servletPath1;
@Value("${server.port:/}")
private String serverPort;

@Value("${management.context-path:/}")
private String endpointContextPath1;
@Value("${management.port:#{null}}")
private Integer managementPort1;
```

#### 1.2 匹配 GetMapping 等注解方法

- 请求匹配 @GetMapping() 内部参数时**仅仅匹配**的是 HttpServletRequest.getPathInfo() 部分
- 只有 HttpServletRequest.getRequestURI() == contextPath1+servletPath1+注解参数时，请求才真正匹配此注解对应的方法
- 同理其它同类型注解

```java
//请求 uri 必须为 /contextPath1/serveletPath1/hello 才能匹配上此方法！！
//即 @GetMapping() 里的字符串匹配的只是 pathInfo，要想匹配上 @GetMapping 首先请求必须匹配 contextPath 和 serveltPath
@GetMapping("/hello")
public String hello() {
    return "hello";
}
```

#### 1.3 匹配 endpoint

- 要想匹配上 /info 端点，请求为：
  - 指定了 management.port：http://ip:serverPort/endpointContextPath1/info
  - 未指定 management.port：http://ip:managementPort1/contextPath1/servletPath1/endpointContextPath1/info
- 总结：如果指定了 management.port，则请求的参数和 contextPath 和 servletPath 无关

### 2 springboot 2.x 

#### 2.1 相关配置参数

```java
@Value("${server.servlet.context-path:/}")
private String contextPath2;
@Value("${spring.mvc.servlet.path:/}")
private String servletPath2;
@Value("${server.port:/}")
private String serverPort;

@Value("${management.endpoints.web.base-path:/actuator}")
private String basePath;
//改变指定端点的映射path
@Value("${management.endpoints.web.path-mapping.prometheus:/prometheus}")
private String pathMapping;

@Value("${management.server.port:#{null}}")
private Integer managementPort2;
//此参数只有在指定了 management.server.port 时才生效！
@Value("${management.server.servlet.context-path:/}")
private String endpointContextPath2;
```

#### 2.2 匹配 GetMapping 等注解方法

- 同理 1.2

#### 2.3 匹配 endpoint

- 匹配没有指定 **path-mapping** 的 endpoint /info
  - 指定了 management.server.port：http://ip:managementPort2/endpointContextPath2/basePath/info
  - 未指定 management.server.port：http://ip:serverPort/contextPath2/servletPath2/basePath/info
- 匹配指定了 **path-mapping** 的 endpoint 如 2.1所示的 prometheus 端口
  - 指定了 management.server.port：http://ip:managementPort2/endpointContextPath2/basePath/pathMapping
  - 未指定 management.server.port：http://ip:serverPort/contextPath2/servletPath2/basePath/pathMapping

- 总结：如果指定了 management.server.port，则请求的参数和 contextPath 和 servletPath 无关

### 

