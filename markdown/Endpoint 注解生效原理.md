[TOC]

- 在 [Springboot 中文文档 —— Actuator](https://blog.csdn.net/kangsa998/article/details/103021718) 一文中，介绍了端点以及其使用方法
- 本文将介绍 @Endpoint 注解的生效原理，以对 actuator 端点有一个更深入的了解



# 1 WebMvcEndpointManagementContextConfiguration

- 这里是向容器注入自定义的 HandlerMapping，供 DispatcherServlet 调用
- DispatcherServlet 根据各个 HandlerMapping 做实际的请求分发

```java
@ManagementContextConfiguration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
@ConditionalOnBean({ DispatcherServlet.class, WebEndpointsSupplier.class })
@EnableConfigurationProperties(CorsEndpointProperties.class)
public class WebMvcEndpointManagementContextConfiguration {

   //1.1 
   @Bean
   @ConditionalOnMissingBean
   public WebMvcEndpointHandlerMapping webEndpointServletHandlerMapping(...) {
       //向容器注入 WebMvcEndpointHandlerMapping，由 DispatcherServlet 调用，转发请求到真实的 endpoint 中的 特定 operation 中
   }

   //1.2 
   @Bean
   @ConditionalOnMissingBean
   public ControllerEndpointHandlerMapping controllerEndpointHandlerMapping(...) {
       //向容器注入 ControllerEndpointHandlerMapping，由 DispatcherServlet 调用，转发请求到真实的 endpoint
   }
}
```

## 1.1 webEndpointServletHandlerMapping

```java
@Bean
@ConditionalOnMissingBean
public WebMvcEndpointHandlerMapping webEndpointServletHandlerMapping(...) {    
    List<ExposableEndpoint<?>> allEndpoints = new ArrayList<>();
    //获取所有的 web 类型 endpoints（@Endpoint、@WebEndpoint 注解）
    //这里可能会触发 endpoints 的初始化，但是应该是被 5 给抢先了
    Collection<ExposableWebEndpoint> webEndpoints = webEndpointsSupplier.getEndpoints();
    allEndpoints.addAll(webEndpoints);
    //获取所有 servlet 类型 endpoints（@ServletEndpoint 注解）
    allEndpoints.addAll(servletEndpointsSupplier.getEndpoints());
    //获取所有 controller 类型 endpoints（@ControllerEndpoint 注解）
    allEndpoints.addAll(controllerEndpointsSupplier.getEndpoints());
    //web endpoint 的 base path
    String basePath = webEndpointProperties.getBasePath();
    EndpointMapping endpointMapping = new EndpointMapping(basePath);
    //当 bathPath 为空，且 endpoint 的端口和server 端口一样，才不暴露
    boolean shouldRegisterLinksMapping = 是否暴露 /actuator 发现页面;
    return new WebMvcEndpointHandlerMapping(...);
}
```

## 1.2 ControllerEndpointHandlerMapping

- 同 1.1

```java
@Bean
@ConditionalOnMissingBean
public ControllerEndpointHandlerMapping controllerEndpointHandlerMapping(...) {
    EndpointMapping endpointMapping = new EndpointMapping(webEndpointProprties.getBasePath());
    return new ControllerEndpointHandlerMapping(...);
}
```
# 2 EndpointAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
public class EndpointAutoConfiguration {
   @Bean
   @ConditionalOnMissingBean 
   public ParameterValueMapper endpointOperationParameterMapper(...) {
       //方法参数：@EndpointConverter ObjectProvider<Converter<?, ?>> converters
       //方法参数：@EndpointConverter ObjectProvider<GenericConverter> genericConverters
		//获取容器中的 @EndpointConverter（Converter，GenericConverter），用于 @Endpoint 输入参数的类型转换
        //如果没有，则使用默认的 ApplicationConversionService
        //如果有，则使用 它们，来设置 ApplicationConversionService
   }
   @Bean
   @ConditionalOnMissingBean
   //返回可缓存的 endpoint 的缓存时间 
   //management.endpoint.endpointName.cache.time-to-live=xx 来配置 endpointName 的缓存时间
   public CachingOperationInvokerAdvisor endpointCachingOperationInvokerAdvisor(Environment environment) {
      return new CachingOperationInvokerAdvisor(new EndpointIdTimeToLivePropertyFunction(environment));
   }
}
```

# 3 WebEndpointAutoConfiguration

- 生成 endpoint 相关信息

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication
@AutoConfigureAfter(EndpointAutoConfiguration.class)
@EnableConfigurationProperties(WebEndpointProperties.class)
public class WebEndpointAutoConfiguration {

   @Bean
   //health,info 等路径的重定义，如：
   //management.endpoints.path-mapping.health=healthcheck
   //management.endpoints.path-mapping.info=myInfo
   public PathMapper webEndpointPathMapper() {
      return new MappingWebEndpointPathMapper(this.properties.getPathMapping());
   }
   @Bean
   @ConditionalOnMissingBean
   public EndpointMediaTypes endpointMediaTypes() {
      return EndpointMediaTypes.DEFAULT;
   }
   @Bean
   @ConditionalOnMissingBean(WebEndpointsSupplier.class)
   public WebEndpointDiscoverer webEndpointDiscoverer(...) {
      //方法参数如下： 
      //1. ParameterValueMapper，2.EndpointMediaTypes，3.PathMapper：info->Myinfo
      //4. OperationInvokerAdvisor,5.EndpointFilter<ExposableWebEndpoint>
      //通过如上参数创建 discoverer 类
      return new WebEndpointDiscoverer(...);
   }
   @Bean
   @ConditionalOnMissingBean(ControllerEndpointsSupplier.class)
   public ControllerEndpointDiscoverer controllerEndpointDiscoverer(...) {
      //同上 
      return new ControllerEndpointDiscoverer(...);
   }
    
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
static class WebEndpointServletConfiguration {
    @Bean
    @ConditionalOnMissingBean(ServletEndpointsSupplier.class)
    ServletEndpointDiscoverer servletEndpointDiscoverer(...) {
        //同 WebEndpointDiscoverer
        return new ServletEndpointDiscoverer(...);
    }
}

   @Bean//5
   @ConditionalOnMissingBean
   public PathMappedEndpoints pathMappedEndpoints(Collection<EndpointsSupplier<?>> endpointSuppliers) {
      //通过 basepath 和 所有的  endpointSuppliers，得到 所有 endpoint 对象 和其对应的 operation
      return new PathMappedEndpoints(this.properties.getBasePath(), endpointSuppliers);
   }

   @Bean
   public ExposeExcludePropertyEndpointFilter<ExposableWebEndpoint> webExposeExcludePropertyEndpointFilter() {
      //从  WebEndpointProperties 中得到 WebEndpointProperties.Exposure
      WebEndpointProperties.Exposure exposure = this.properties.getExposure();
      //使用参数，如果没有 include 则使用默认的 info/health，如果有 include 则默认的失效 
      return new ExposeExcludePropertyEndpointFilter<>(ExposableWebEndpoint.class, exposure.getInclude(),
            exposure.getExclude(), "info", "health");
   }

   @Bean
   public ExposeExcludePropertyEndpointFilter<ExposableControllerEndpoint> controllerExposeExcludePropertyEndpointFilter() {
      WebEndpointProperties.Exposure exposure = this.properties.getExposure();
      //同上，这里没有默认配置
      return new ExposeExcludePropertyEndpointFilter<>(...);
   }
}
```
# 4 请求生效流程

## 4.1 doDispatch(request, response)

```java
//DispatcherServlet
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    // Determine handler for the current request.
    mappedHandler = getHandler(processedRequest);
    // Determine handler adapter for the current request.
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
    // Process last-modified header, if supported by the handler.
    String method = request.getMethod();

    //可以自定义配置预处理方法，如鉴权等
    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
    }
    // Actually invoke the handler.
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

    applyDefaultViewName(processedRequest, mv);
    //调用后置处理器，如加密等
    mappedHandler.applyPostHandle(processedRequest, response, mv);
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}
```

### 4.1.1 getHandler(processedRequest)

```java
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   //其中关键的mapping： WebMvcEndpointHandlerMapping、ControllerEndpointHandlerMapping、RequestMappingHandlerMapping
   if (this.handlerMappings != null) {
      for (HandlerMapping mapping : this.handlerMappings) {
         //获取request对应的handler，并处理为 chain 
         HandlerExecutionChain handler = mapping.getHandler(request);
         if (handler != null) {
            return handler;
         }
      }
   }
   return null;
}
```

### 4.1.2 getHandlerAdapter(Object handler)

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
   // 默认提供多个 handlerAdapter
   if (this.handlerAdapters != null) {
      for (HandlerAdapter adapter : this.handlerAdapters) { 
         if (adapter.supports(handler)) {
            //返回第一个可以匹配的 adapter 
            //这里返回的是：RequestMappingHandlerAdapter
            return adapter;
         }
      }
   }
   throw new ServletException("No adapter for handler [" + handler +
         "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

### 4.1.3 ha.handle()

```java
//AbstractWebMvcEndpointHandlerMapping 内部类
private final class OperationHandler {
   @ResponseBody
   Object handle(HttpServletRequest request, @RequestBody(required = false) Map<String, String> body) {
      //最终走到这 
      return this.operation.handle(request, body);
   }
}
@Override
public Object handle(HttpServletRequest request, @RequestBody(required = false) Map<String, String> body) {
	//通过 this.operation.invoke 最终调用实际方法
	return handleResult(this.operation.invoke(invocationContext), HttpMethod.resolve(request.getMethod()));
}
```


# 5 PathMappedEndpoints

```java
@Bean
@ConditionalOnMissingBean
public PathMappedEndpoints pathMappedEndpoints(Collection<EndpointsSupplier<?>> endpointSuppliers) {
    //通过 basepath 和 所有的  endpointSuppliers，得到 对象
    return new PathMappedEndpoints(this.properties.getBasePath(), endpointSuppliers);
}

public PathMappedEndpoints(String basePath, Collection<EndpointsSupplier<?>> suppliers) {
    Assert.notNull(suppliers, "Suppliers must not be null");
    this.basePath = (basePath != null) ? basePath : "";
    //3.1.1
    this.endpoints = getEndpoints(suppliers);
}
```

## 5.1 getEndpoints

- 获取所有类型的 endpoint

```java
private Map<EndpointId, PathMappedEndpoint> getEndpoints(Collection<EndpointsSupplier<?>> suppliers) {
    Map<EndpointId, PathMappedEndpoint> endpoints = new LinkedHashMap<>();
    suppliers.forEach((supplier) -> {
        supplier.getEndpoints().forEach((endpoint) -> {
            if (endpoint instanceof PathMappedEndpoint) {
                endpoints.put(endpoint.getEndpointId(), (PathMappedEndpoint) endpoint);
            }
        });
    });
    return Collections.unmodifiableMap(endpoints);
}
```

## 5.2 supplier.getEndpoints()

- 得到一个 supplier 所有的 endpoints

```java
//EndpointDiscoverer
@Override
public final Collection<E> getEndpoints() {
    if (this.endpoints == null) {
        this.endpoints = discoverEndpoints();
    }
    return this.endpoints;
}

private Collection<E> discoverEndpoints() {
    //5.2.1 得到所有的 endpoint
    Collection<EndpointBean> endpointBeans = createEndpointBeans();
    //5.2.2 将 extension 加入所有 extension id 对应的 endpoint id 的 endpointBean（filter 必须匹配，否则不填充）
    //目前有：healthEndpointWebExtension、environmentEndpointWebExtension
    addExtensionBeans(endpointBeans);
    //5.2.3 最终处理 endpoint
    return convertToEndpoints(endpointBeans);
}
```

### 5.2.1 createEndpointBeans

```java
//EndpointDiscoverer
private Collection<EndpointBean> createEndpointBeans() {
    Map<EndpointId, EndpointBean> byId = new LinkedHashMap<>();
    //得到所有 @Endpoint 及其子注解的类名
    String[] beanNames = BeanFactoryUtils.beanNamesForAnnotationIncludingAncestors(this.applicationContext,
                                                                                 Endpoint.class);
    
    for (String beanName : beanNames) {
        //不是被代理的类(!beanName.startsWith(scopedTarget.))，才能进
        if (!ScopedProxyUtils.isScopedTarget(beanName)) {
            //5.2.1.1
            EndpointBean endpointBean = createEndpointBean(beanName);
            EndpointBean previous = byId.putIfAbsent(endpointBean.getId(), endpointBean);
        }
    }
    return byId.values();
}
```

#### 5.2.1.1 createEndpointBean(beanName)

```java
private EndpointBean createEndpointBean(String beanName) {
    Object bean = this.applicationContext.getBean(beanName);
    return new EndpointBean(this.applicationContext.getEnvironment(), beanName, bean);
}

EndpointBean(Environment environment, String beanName, Object bean) {
    MergedAnnotation<Endpoint> annotation = MergedAnnotations
        .from(bean.getClass(), SearchStrategy.TYPE_HIERARCHY).get(Endpoint.class);
    String id = annotation.getString("id");

    this.beanName = beanName;
    this.bean = bean;
    this.id = EndpointId.of(environment, id);
    this.enabledByDefault = annotation.getBoolean("enableByDefault");
    //获取对应的 filter,controller 对应的是 ControllerEndpointFilter
    this.filter = getFilter(this.bean.getClass());
}
```

### 5.2.2 addExtensionBeans

- 将所有 endpointBean 对应的 extensionBean 填充到其中（必须匹配）

```java
//EndpointDiscoverer
private void addExtensionBeans(Collection<EndpointBean> endpointBeans) {
    Map<EndpointId, EndpointBean> byId = endpointBeans.stream()
        .collect(Collectors.toMap(EndpointBean::getId, Function.identity()));
    //得到所有 @EndpointExtension 及其子注解的类名
    //注意 @EndpointExtension 上的 id 对应 @Endpoint 上的 id
    //说明，此 EndpointExtension 是对 对应的 Endpoint 类的扩展
    String[] beanNames = ...;
    //
    for (String beanName : beanNames) {
        ExtensionBean extensionBean = createExtensionBean(beanName);
        //得到 ExtensionBean 对应的 EndpointBean
        EndpointBean endpointBean = byId.get(extensionBean.getEndpointId());
        //5.2.2.1
        addExtensionBean(endpointBean, extensionBean);
    }
}
```

#### 5.2.2.1 addExtensionBean(endpointBean, extensionBean)

- 为 endpointBean 添加 extensionBean

```java
//EndpointDiscoverer
private void addExtensionBean(EndpointBean endpointBean, ExtensionBean extensionBean) {
    //只有 endpointBean、extensionBean、对应的 supllier 匹配，才能将 extensionBean 加入 endpointBean
    if (isExtensionExposed(endpointBean, extensionBean)) {
        endpointBean.addExtension(extensionBean);
    }
}
```

##### 5.2.2.1.1 isExtensionExposed(endpointBean, extensionBean)

- 

```java
//EndpointDiscoverer
private boolean isExtensionExposed(EndpointBean endpointBean, ExtensionBean extensionBean) {
   return isFilterMatch(extensionBean.getFilter(), endpointBean) && isExtensionExposed(extensionBean.getBean());
}
//isFilterMatch：判断 extensionBean 对应的 filter 和 endpointBean 对应的是否一样
//isExtensionExposed:一般为 true
```

##### 5.2.2.1.2 isEndpointExposed(endpointBean)

- 判断 endpointbean 是否匹配对应的 filter
- 判断此 bean 是否被 exclude 了！(配置文件配置)
- 判断 bean 上的注解是否匹配当前 supplier

```java
//isFilterMatch(endpointBean.getFilter(), endpointBean):判断 bean 是否匹配对应的 filter，具体可查看：isExposed(ExposableEndpoint<?> endpoint)、isExcluded(ExposableEndpoint<?> endpoint) 方法
//isEndpointFiltered(endpointBean):判断此 bean 是否被 exclude 了！(配置文件配置)
//isEndpointExposed(endpointBean.getBean()):判断 bean 上的注解是否匹配当前 supplier
//其中@Endpoint、@WebEndpoint 都只适用于 WebDiscoverer！原因可跟踪一下方法源码
private boolean isEndpointExposed(EndpointBean endpointBean) {
    return isFilterMatch(endpointBean.getFilter(), endpointBean) && !isEndpointFiltered(endpointBean)
        && isEndpointExposed(endpointBean.getBean());
}
```



### 5.2.3 convertToEndpoints

```java
//EndpointDiscoverer
private Collection<E> convertToEndpoints(Collection<EndpointBean> endpointBeans) {
    Set<E> endpoints = new LinkedHashSet<>();
    for (EndpointBean endpointBean : endpointBeans) {
        //这里就过滤掉不属于当前 supplier 的 endpoint 的了
        if (isEndpointExposed(endpointBean)) {
            endpoints.add(convertToEndpoint(endpointBean));
        }
    }
    //得到属于自己的包装了 对应的 extendendpoint 的 endpoint
    return Collections.unmodifiableSet(endpoints);
}
```

#### 5.2.3.1 convertToEndpoint(endpointBean)

```java
//EndpointDiscoverer
private E convertToEndpoint(EndpointBean endpointBean) {
    MultiValueMap<OperationKey, O> indexed = new LinkedMultiValueMap<>();
    EndpointId id = endpointBean.getId();
    //为 endpoint 创建 operations
    addOperations(indexed, id, endpointBean.getBean(), false);
    //一个 endpoint 只能有一个 extension
    if (endpointBean.getExtensions().size() > 1) {
		//报错
    }
    for (ExtensionBean extensionBean : endpointBean.getExtensions()) {
        //为 endpoint 对应的 extension 添加 operation
        addOperations(indexed, id, extensionBean.getBean(), true);
    }
    //判断 operation 有没有重复
    assertNoDuplicateOperations(endpointBean, indexed);
    List<O> operations = indexed.values().stream().map(this::getLast).filter(Objects::nonNull)
        .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
    //创建
    return createEndpoint(endpointBean.getBean(), id, endpointBean.isEnabledByDefault(), operations);
}
```

##### 5.2.3.2.1 addOperations()

```java
//EndpointDiscoverer
private void addOperations(MultiValueMap<OperationKey, O> indexed, EndpointId id, Object target,
      boolean replaceLast) {
   Set<OperationKey> replacedLast = new HashSet<>();
   // 创建此 endpoint 对应的所有 Operations
   Collection<O> operations = this.operationsFactory.createOperations(id, target);
   for (O operation : operations) {
      OperationKey key = createOperationKey(operation);
      O last = getLast(indexed.get(key));
      if (replaceLast && replacedLast.add(key) && last != null) {
         indexed.get(key).remove(last);
      }
      indexed.add(key, operation);
   }
}
```

##### 5.2.3.2.2 createOperations(...)

- 创建此 endpoint 对应的所有 Operations

```java
Collection<O> createOperations(EndpointId id, Object target) {
   return MethodIntrospector
         .selectMethods(target.getClass(), (MetadataLookup<O>) (method) -> createOperation(id, target, method))
         .values();
}
//最终会走到不同 Discoverer 的各自实现中
@Override
protected WebOperation createOperation(...) {
    //根据 pathMapping 中的值替换 endpoint id 对应的值（如果有的话）
    String rootPath = PathMapper.getRootPath(this.endpointPathMappers, endpointId);
    WebOperationRequestPredicate requestPredicate = this.requestPredicateFactory.getRequestPredicate(rootPath,
                                                                                                     operationMethod);
    return new DiscoveredWebOperation(endpointId, operationMethod, invoker, requestPredicate);
}
```

##### 5.2.3.2.3 createEndpoint(...)

```java
//不同的 Discoverer 有不同的实现，以下是 WebEndpointDiscoverer
@Override
protected ExposableWebEndpoint createEndpoint(Object endpointBean, EndpointId id, boolean enabledByDefault,
      Collection<WebOperation> operations) {
    //默认为 /actuator
   String rootPath = PathMapper.getRootPath(this.endpointPathMappers, id);
   return new DiscoveredWebEndpoint(this, endpointBean, id, rootPath, enabledByDefault, operations);
}
```



# 参考

springboot 2.2.1.RELEASE