[TOC]

- 在 [Springboot 中文文档 —— Actuator](https://blog.csdn.net/kangsa998/article/details/103021718) 一文中，介绍了端点以及其使用方法
- 本文将介绍 @Endpoint 注解的生效原理，以对 actuator 端点有一个更深入的了解



# 1 WebMvcEndpointManagementContextConfiguration

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
       //向容器注入 WebMvcEndpointHandlerMapping
   }

   //1.2 
   @Bean
   @ConditionalOnMissingBean
   public ControllerEndpointHandlerMapping controllerEndpointHandlerMapping(...) {
       //向容器注入 ControllerEndpointHandlerMapping
   }

}
```

## 1.1 webEndpointServletHandlerMapping

- @Bean 方法参数介绍

```txt
WebEndpointsSupplier：
ServletEndpointsSupplier：
ControllerEndpointsSupplier：
EndpointMediaTypes：
CorsEndpointProperties：
WebEndpointProperties：
Environment：
```

- @Bean 方法介绍

```java
@Bean
@ConditionalOnMissingBean
public WebMvcEndpointHandlerMapping webEndpointServletHandlerMapping(...) {    
    List<ExposableEndpoint<?>> allEndpoints = new ArrayList<>();
    //获取所有的 web 类型 endpoints
    Collection<ExposableWebEndpoint> webEndpoints = webEndpointsSupplier.getEndpoints();
    allEndpoints.addAll(webEndpoints);
    //获取所有 servlet 类型 endpoints
    allEndpoints.addAll(servletEndpointsSupplier.getEndpoints());
    //获取所有 controller 类型 endpoints
    allEndpoints.addAll(controllerEndpointsSupplier.getEndpoints());
    //web endpoint 的 base path
    String basePath = webEndpointProperties.getBasePath();
    EndpointMapping endpointMapping = new EndpointMapping(basePath);
    boolean shouldRegisterLinksMapping = StringUtils.hasText(basePath)
        || ManagementPortType.get(environment).equals(ManagementPortType.DIFFERENT);
    return new WebMvcEndpointHandlerMapping(endpointMapping, webEndpoints, endpointMediaTypes,
                                            corsProperties.toCorsConfiguration(), new EndpointLinksResolver(allEndpoints, basePath),
                                            shouldRegisterLinksMapping);
}
```

## 1.2 ControllerEndpointHandlerMapping

- @Bean 方法参数介绍

```txt
ControllerEndpointsSupplier：
CorsEndpointProperties：
WebEndpointProperties：
```
- @Bean 方法介绍

```java

@Bean
@ConditionalOnMissingBean
public ControllerEndpointHandlerMapping controllerEndpointHandlerMapping(...) {
    EndpointMapping endpointMapping = new EndpointMapping(webEndpointProperties.getBasePath());
    return new ControllerEndpointHandlerMapping(endpointMapping, controllerEndpointsSupplier.getEndpoints(),
                                                corsProperties.toCorsConfiguration());
}
```



# 2 WebEndpointAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication
@AutoConfigureAfter(EndpointAutoConfiguration.class)
@EnableConfigurationProperties(WebEndpointProperties.class)
public class WebEndpointAutoConfiguration {

   private final ApplicationContext applicationContext;

   private final WebEndpointProperties properties;

   public WebEndpointAutoConfiguration(ApplicationContext applicationContext, WebEndpointProperties properties) {
      this.applicationContext = applicationContext;
      this.properties = properties;
   }

   @Bean
   //health,info 等路劲的重定义，如：health:healthcheck
   public PathMapper webEndpointPathMapper() {
      return new MappingWebEndpointPathMapper(this.properties.getPathMapping());
   }

   @Bean
   @ConditionalOnMissingBean
   public EndpointMediaTypes endpointMediaTypes() {
      return EndpointMediaTypes.DEFAULT;
   }

   //web endpoint 的暴露类，关键的类
   @Bean
   @ConditionalOnMissingBean(WebEndpointsSupplier.class)
   public WebEndpointDiscoverer webEndpointDiscoverer(...) {
      //方法参数如下： 
      //1. ParameterValueMapper，2.EndpointMediaTypes，3.PathMapper
      //4. OperationInvokerAdvisor,5.EndpointFilter<ExposableWebEndpoint>
      //通过如上参数创建 discoverer 类
      return new WebEndpointDiscoverer(...);
   }

   @Bean//，关键的类
   @ConditionalOnMissingBean(ControllerEndpointsSupplier.class)
   public ControllerEndpointDiscoverer controllerEndpointDiscoverer(...) {
      //同上 
      return new ControllerEndpointDiscoverer(...);
   }

   @Bean//3.1 最最重要的方法！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
   @ConditionalOnMissingBean
   public PathMappedEndpoints pathMappedEndpoints(Collection<EndpointsSupplier<?>> endpointSuppliers) {
      //通过 basepath 和 所有的  endpointSuppliers，得到 对象
      return new PathMappedEndpoints(this.properties.getBasePath(), endpointSuppliers);
   }

   @Bean
   public ExposeExcludePropertyEndpointFilter<ExposableWebEndpoint> webExposeExcludePropertyEndpointFilter() {
      //从  WebEndpointProperties 中得到 WebEndpointProperties.Exposure
      WebEndpointProperties.Exposure exposure = this.properties.getExposure();
      //使用参数 
      return new ExposeExcludePropertyEndpointFilter<>(ExposableWebEndpoint.class, exposure.getInclude(),
            exposure.getExclude(), "info", "health");
   }

   @Bean
   public ExposeExcludePropertyEndpointFilter<ExposableControllerEndpoint> controllerExposeExcludePropertyEndpointFilter() {
      WebEndpointProperties.Exposure exposure = this.properties.getExposure();
      //同上 
      return new ExposeExcludePropertyEndpointFilter<>(...);
   }

   @Configuration(proxyBeanMethods = false)
   @ConditionalOnWebApplication(type = Type.SERVLET)
   static class WebEndpointServletConfiguration {

      @Bean
      @ConditionalOnMissingBean(ServletEndpointsSupplier.class)
      ServletEndpointDiscoverer servletEndpointDiscoverer(ApplicationContext applicationContext,
            ObjectProvider<PathMapper> endpointPathMappers,
            ObjectProvider<EndpointFilter<ExposableServletEndpoint>> filters) {
         //方法参数
         //1. PathMapper,2.EndpointFilter<ExposableServletEndpoint>
         //同 webEndpointDiscoverer
         return new ServletEndpointDiscoverer(applicationContext,
               endpointPathMappers.orderedStream().collect(Collectors.toList()),
               filters.orderedStream().collect(Collectors.toList()));
      }

   }

}
```

## 3.1 PathMappedEndpoints

```java
@Bean//3.1 最最重要的方法！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
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

### 3.1.1 getEndpoints

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

### 3.1.2 supplier.getEndpoints()

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
    //3.1.2.1 得到所有的 endpoint
    Collection<EndpointBean> endpointBeans = createEndpointBeans();
    
    //3.1.2.2 将 extension 加入所有 extension id 对应的 endpoint id 的 endpointBean（filter 必须匹配，否则不填充）
    //目前有：healthEndpointWebExtension、environmentEndpointWebExtension
    addExtensionBeans(endpointBeans);
    //3.1.2.3
    return convertToEndpoints(endpointBeans);
}
```

#### 3.1.2.1 createEndpointBeans

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
            //3.1.2.1.1
            EndpointBean endpointBean = createEndpointBean(beanName);
            EndpointBean previous = byId.putIfAbsent(endpointBean.getId(), endpointBean);
            Assert.state(previous == null, () -> "Found two endpoints with the id '" + endpointBean.getId() + "': '"
                         + endpointBean.getBeanName() + "' and '" + previous.getBeanName() + "'");
        }
    }
    return byId.values();
}
```

##### 3.1.2.1.1 createEndpointBean(beanName)

```java
private EndpointBean createEndpointBean(String beanName) {
    Object bean = this.applicationContext.getBean(beanName);
    return new EndpointBean(this.applicationContext.getEnvironment(), beanName, bean);
}

EndpointBean(Environment environment, String beanName, Object bean) {
    MergedAnnotation<Endpoint> annotation = MergedAnnotations
        .from(bean.getClass(), SearchStrategy.TYPE_HIERARCHY).get(Endpoint.class);
    String id = annotation.getString("id");
    Assert.state(StringUtils.hasText(id),
                 () -> "No @Endpoint id attribute specified for " + bean.getClass().getName());
    this.beanName = beanName;
    this.bean = bean;
    this.id = EndpointId.of(environment, id);
    this.enabledByDefault = annotation.getBoolean("enableByDefault");
    //获取对应的 filter,controller 对应的是 ControllerEndpointFilter
    this.filter = getFilter(this.bean.getClass());
}
```

###### 3.1.2.1.1.1 getFilter(this.bean.getClass())

- 获取当前bean 对应的 FilteredEndpoint

```java
private Class<?> getFilter(Class<?> type) {
    return MergedAnnotations.from(type, SearchStrategy.TYPE_HIERARCHY).get(FilteredEndpoint.class)
        .getValue(MergedAnnotation.VALUE, Class.class).orElse(null);
}
```



#### 3.1.2.2 addExtensionBeans

- 将所有 endpointBean 对应的 extensionBean 填充到其中（必须匹配）

```java
//EndpointDiscoverer
private void addExtensionBeans(Collection<EndpointBean> endpointBeans) {
    Map<EndpointId, EndpointBean> byId = endpointBeans.stream()
        .collect(Collectors.toMap(EndpointBean::getId, Function.identity()));
    //得到所有 @EndpointExtension 及其子注解的类名
    //注意 @EndpointExtension 上的 id 对应 @Endpoint 上的 id
    //说明，此 EndpointExtension 是对 对应的 Endpoint 类的扩展
    String[] beanNames = BeanFactoryUtils.beanNamesForAnnotationIncludingAncestors(this.applicationContext,
                                                                                   EndpointExtension.class);
    //
    for (String beanName : beanNames) {
        ExtensionBean extensionBean = createExtensionBean(beanName);
        //得到 ExtensionBean 对应的 EndpointBean
        EndpointBean endpointBean = byId.get(extensionBean.getEndpointId());
        Assert.state(endpointBean != null, () -> ("Invalid extension '" + extensionBean.getBeanName()
                                                  + "': no endpoint found with id '" + extensionBean.getEndpointId() + "'"));
        //3.1.2.2.1
        addExtensionBean(endpointBean, extensionBean);
    }
}
```

##### 3.1.2.2.1 addExtensionBean(endpointBean, extensionBean)

```java
//EndpointDiscoverer
private void addExtensionBean(EndpointBean endpointBean, ExtensionBean extensionBean) {
    //只有 endpointBean、extensionBean、对应的 supllier 匹配，才能将 extensionBean 加入 endpointBean
    if (isExtensionExposed(endpointBean, extensionBean)) {
        Assert.state(isEndpointExposed(endpointBean) || isEndpointFiltered(endpointBean),
                     () -> "Endpoint bean '" + endpointBean.getBeanName() + "' cannot support the extension bean '"
                     + extensionBean.getBeanName() + "'");
        endpointBean.addExtension(extensionBean);
    }
}
```

###### 3.1.2.2.1.1 isExtensionExposed(endpointBean, extensionBean)

- 匹配 supplier 和 extensionBean

```java
//EndpointDiscoverer
private boolean isExtensionExposed(EndpointBean endpointBean, ExtensionBean extensionBean) {
   return isFilterMatch(extensionBean.getFilter(), endpointBean) && isExtensionExposed(extensionBean.getBean());
}
```

###### 3.1.2.2.1.2 isEndpointExposed(endpointBean)

- 匹配 supplier 和 endpointBean

```java
//调用对应 supplier 的重写方法
//ControllerEndpointDiscoverer：只有 endpoint 为 Controller 返回 true
//EndpointDiscoverer：直接返回 true
//ServletEndpointDiscoverer：只有 endpoint 为 ServeltEndpoint 返回 true
```



#### 3.1.2.3 convertToEndpoints

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

##### 3.1.2.3.1 convertToEndpoint(endpointBean)

```java
//EndpointDiscoverer
private E convertToEndpoint(EndpointBean endpointBean) {
    MultiValueMap<OperationKey, O> indexed = new LinkedMultiValueMap<>();
    EndpointId id = endpointBean.getId();
    addOperations(indexed, id, endpointBean.getBean(), false);
    if (endpointBean.getExtensions().size() > 1) {
        String extensionBeans = endpointBean.getExtensions().stream().map(ExtensionBean::getBeanName)
            .collect(Collectors.joining(", "));
        throw new IllegalStateException("Found multiple extensions for the endpoint bean "
                                        + endpointBean.getBeanName() + " (" + extensionBeans + ")");
    }
    for (ExtensionBean extensionBean : endpointBean.getExtensions()) {
        addOperations(indexed, id, extensionBean.getBean(), true);
    }
    assertNoDuplicateOperations(endpointBean, indexed);
    List<O> operations = indexed.values().stream().map(this::getLast).filter(Objects::nonNull)
        .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
    return createEndpoint(endpointBean.getBean(), id, endpointBean.isEnabledByDefault(), operations);
}
```

###### 3.1.2.3.2.1 addOperations()

```java
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

###### 3.1.2.3.2.1 createOperations()

- 创建此 endpoint 对应的所有 Operations

```java
Collection<O> createOperations(EndpointId id, Object target) {
   return MethodIntrospector
         .selectMethods(target.getClass(), (MetadataLookup<O>) (method) -> createOperation(id, target, method))
         .values();
}
```

# 3 EndpointAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
public class EndpointAutoConfiguration {

   //3.1 
   @Bean
   @ConditionalOnMissingBean 
   public ParameterValueMapper endpointOperationParameterMapper(...) {
		//获取容器中的 @EndpointConverter（Converter，GenericConverter）
        //如果没有，则使用默认的 ApplicationConversionService
        //如果有，则使用 它们，来设置 ApplicationConversionService
   }
   @Bean
   @ConditionalOnMissingBean
   //可缓存的 endpoint 的缓存时间 
   public CachingOperationInvokerAdvisor endpointCachingOperationInvokerAdvisor(Environment environment) {
      return new CachingOperationInvokerAdvisor(new EndpointIdTimeToLivePropertyFunction(environment));
   }
}
```



