[TOC]

- springboot 通过 endpoint 将容器中的 metrics 暴露出去
- 本文分析了 metrics 生效、暴露流程

# 1 MetricsAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(Timed.class)
//得到 MetricsProperties 对应的配置属性 @ConfigurationProperties("management.metrics")
@EnableConfigurationProperties(MetricsProperties.class)
@AutoConfigureBefore(CompositeMeterRegistryAutoConfiguration.class)
public class MetricsAutoConfiguration {
   @Bean
   @ConditionalOnMissingBean
   public Clock micrometerClock() {
      return Clock.SYSTEM;
   }
   //向 spring 容器中注入 postProcessor 用于处理容器中的 MeterRegistry 类
    //得到一个 BeanPostProcessor，用来处理 MeterRegistry 类	
   //将所有的 MeterRegistry，添加到 Metrics.globalRegistry 中（如果 this.metricsProperties.getObject().isUseGlobalRegistry() 为 true）
   //将所有的 meterFilters，meterBinders 绑定到所有的 MeterRegistry 中
   //所以用户可以自定义  MeterRegistry，meterFilters，meterBinders
   @Bean
   public static MeterRegistryPostProcessor meterRegistryPostProcessor(ObjectProvider<MeterBinder> meterBinders,
         ObjectProvider<MeterFilter> meterFilters,
         ObjectProvider<MeterRegistryCustomizer<?>> meterRegistryCustomizers,
         ObjectProvider<MetricsProperties> metricsProperties, ApplicationContext applicationContext) {
      return new MeterRegistryPostProcessor(meterBinders, meterFilters, meterRegistryCustomizers, metricsProperties,
            applicationContext);
   }

   @Bean
   @Order(0)
    //向容器中注入 meter 过滤器
   public PropertiesMeterFilter propertiesMeterFilter(MetricsProperties properties) {
      return new PropertiesMeterFilter(properties);
   }
}
```

# 2 CompositeMeterRegistryAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@Import({ NoOpMeterRegistryConfiguration.class, CompositeMeterRegistryConfiguration.class })
@ConditionalOnClass(CompositeMeterRegistry.class)
public class CompositeMeterRegistryAutoConfiguration {

}
```

## 2.1 NoOpMeterRegistryConfiguration

- 如果容器中没有其他 MeterRegistry 类时，会注入这个 无操作 MeterRegistry 类

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(Clock.class)
@ConditionalOnMissingBean(MeterRegistry.class)
class NoOpMeterRegistryConfiguration {
   @Bean
   CompositeMeterRegistry noOpMeterRegistry(Clock clock) {
      return new CompositeMeterRegistry(clock);
   }
}
```

## 2.2 CompositeMeterRegistryConfiguration

- 当容器中有 MeterRegistry 并且没有一个 primary MeterRegistry 时，才注入默认的组合 MeterRegistry 

```java
@Configuration(proxyBeanMethods = false)
@Conditional(MultipleNonPrimaryMeterRegistriesCondition.class)
class CompositeMeterRegistryConfiguration {
   @Bean
   @Primary
   CompositeMeterRegistry compositeMeterRegistry(Clock clock, List<MeterRegistry> registries) {
      return new CompositeMeterRegistry(clock, registries);
   }
   static class MultipleNonPrimaryMeterRegistriesCondition extends NoneNestedConditions {
      MultipleNonPrimaryMeterRegistriesCondition() {
         super(ConfigurationPhase.REGISTER_BEAN);
      }
      //没有 REGISTER_BEAN.class	
      @ConditionalOnMissingBean(MeterRegistry.class)
      static class NoMeterRegistryCondition {}
      //只有一个 primary MeterRegistry 
      @ConditionalOnSingleCandidate(MeterRegistry.class)
      static class SingleInjectableMeterRegistry {}
   }
}
```

# 3 MeterRegistryPostProcessor

```java
class MeterRegistryPostProcessor implements BeanPostProcessor {

   MeterRegistryPostProcessor(ObjectProvider<MeterBinder> meterBinders,
         ObjectProvider<MeterFilter> meterFilters,
         ObjectProvider<MeterRegistryCustomizer<?>> meterRegistryCustomizers,
         ObjectProvider<MetricsProperties> metricsProperties) {
      //查找得到容器中所有的  meterBinders/meterFilters/meterRegistryCustomizers
      this.meterBinders = meterBinders;
      this.meterFilters = meterFilters;
      this.meterRegistryCustomizers = meterRegistryCustomizers;
      //得到 metricsProperties 配置
      this.metricsProperties = metricsProperties;
   }
   //生成 bean 时，会调用  postProcessAfterInitialization 方法
   //具体时机请参考： 
   @Override
   public Object postProcessAfterInitialization(Object bean, String beanName)
         throws BeansException { 
      if (bean instanceof MeterRegistry) {
         //使用  MeterRegistryConfigurer 对 MeterRegistry 类做自定义操作
         getConfigurer().configure((MeterRegistry) bean);
      }
      return bean;
   }
   private MeterRegistryConfigurer getConfigurer() {
      if (this.configurer == null) {
         //通过几个参数创建  MeterRegistryConfigurer
         this.configurer = new MeterRegistryConfigurer(this.meterRegistryCustomizers,
               this.meterFilters, this.meterBinders,
               this.metricsProperties.getObject().isUseGlobalRegistry());
      }
      return this.configurer;
   }
}
```

## 3.1 MeterRegistryConfigurer

```java
class MeterRegistryConfigurer {
    void configure(MeterRegistry registry) {
		// Customizers must be applied before binders, as they may add custom
		// tags or alter timer or summary configuration.
		customize(registry);
		addFilters(registry);//为所有 MeterRegistry 添加 Fitler
		addBinders(registry);//为 CompositeMeterRegistry 添加 Binder，如果没有 CompositeMeterRegistry，则为所有 registry 添加 binder
		if (this.addToGlobalRegistry && registry != Metrics.globalRegistry) {
            //如果需要加入 GlobalRegistry 且自身又不是 globalRegistry 则加入其中
            //globRegistry 是一个 CompositeMeterRegistry
            //这个 Metrics.globalRegistry 很关键
			Metrics.addRegistry(registry);
		}
	}
}
```

# 4 MetricsEndpointAutoConfiguration

- 这个配置类向容器中注入了 metrics 的 endpoint，将所有 metrics 信息通过 web 端口暴露出去
- 使得所有的 metrics 生效！

```java
@Configuration
@ConditionalOnClass(Timed.class)
@ConditionalOnEnabledEndpoint(endpoint = MetricsEndpoint.class)
@AutoConfigureAfter({ MetricsAutoConfiguration.class,
      CompositeMeterRegistryAutoConfiguration.class })
public class MetricsEndpointAutoConfiguration {
   @Bean
   @ConditionalOnBean(MeterRegistry.class)
   //如果开发人员没有自定义 MetricsEndpoint 则创建一个 
   @ConditionalOnMissingBean
   //必须有一个 primary MeterRegistry，以这个为主 
   public MetricsEndpoint metricsEndpoint(MeterRegistry registry) {
      return new MetricsEndpoint(registry);
   }
}
```

## 4.1 MetricsEndpoint

- 这是 metrics 起作用的核心类，通过这个 endpoint 将 metrics 数据暴露出去
- [endpoint 生效原理](<https://blog.csdn.net/kangsa998/article/details/103166953>)

```java
@Endpoint(id = "metrics")
public class MetricsEndpoint {
   //通过 /metrics 路径，得到所有 registry 的名字 —— 4.1.1 
   @ReadOperation
   public ListNamesResponse listNames() {
      Set<String> names = new LinkedHashSet<>();
      collectNames(names, this.registry);
      return new ListNamesResponse(names);
   }
    //通过 metricName 和 tag 获取对应的 metric 的 信息 —— 4.1.2
   @ReadOperation
   public MetricResponse metric(@Selector String requiredMetricName, @Nullable List<String> tag) {...}
}
```

### 4.1.1 获取所有 registry 名字

- 重要变量：meterMap

```java
private void collectNames(Set<String> names, MeterRegistry registry) {
   if (registry instanceof CompositeMeterRegistry) {
       //默认是组合的，搜集每个子 registry 下注册的 registry
      ((CompositeMeterRegistry) registry).getRegistries().forEach((member) -> collectNames(names, member));
   } else {
      registry.getMeters().stream().map(this::getName).forEach(names::add);
   }
}
//得到每个 registry 中注册的 meter
public List<Meter> getMeters() {
    //每个 registry 中的 meterMap 通过 getOrCreateMeter()、remove(Meter.Id id) 修改的
    //key 为每个 meter 的 id，我们取的是 value
    return Collections.unmodifiableList(new ArrayList<>(meterMap.values()));
}
```

### 4.1.2 获取特定 meter 的信息

- 重要变量：meterMap、meter.getId().getTags()->获取meter 对应的所有 tags
- 重要方法：
  - meter.measure()：返回 meter 对于的 List\<measurement\>
  - Map.merge(key,value,function)：按照 function 合并相同 key 的value 值

```java
@ReadOperation
public MetricResponse metric(@Selector String requiredMetricName, @Nullable List<String> tag) {
   // 将 tag 列表对，转换为 Tag 
   List<Tag> tags = parseTags(tag);
   // 通过 metric name 和 tags 找出对应的 Meters，4.1.2.1
   Collection<Meter> meters = findFirstMatchingMeters(this.registry, requiredMetricName, tags);
   if (meters.isEmpty()) {
      return null;
   }
   //4.1.2.2
   Map<Statistic, Double> samples = getSamples(meters);
   //获取所有的 tags 
   Map<String, Set<String>> availableTags = getAvailableTags(meters);
   //去除以参数传进来的 tags 
   tags.forEach((t) -> availableTags.remove(t.getKey()));
   Meter.Id meterId = meters.iterator().next().getId();
   //包装 metricName,meter description,baseUnit，samples，tags 
   return new MetricResponse(requiredMetricName, meterId.getDescription(), meterId.getBaseUnit(),
         asList(samples, Sample::new), asList(availableTags, AvailableTag::new));
}
```

#### 4.1.2.1 获取所有 meters

```java
private Stream<Meter> meterStream() {
    //通过 meter name 过滤得到对应的 meters
    Stream<Meter> meterStream = registry.getMeters().stream().filter(m -> nameMatches.test(m.getId().getName()));
	//通过 tags 过滤一些 meters
    if (!tags.isEmpty() || !requiredTagKeys.isEmpty()) {
        meterStream = meterStream.filter(m -> {
            boolean requiredKeysPresent = true;
            if (!requiredTagKeys.isEmpty()) {
                final List<String> tagKeys = new ArrayList<>();
                m.getId().getTags().forEach(t -> tagKeys.add(t.getKey()));
                requiredKeysPresent = tagKeys.containsAll(requiredTagKeys);
            }

            return requiredKeysPresent && m.getId().getTags().containsAll(tags);
        });
    }
	//最终返回 name、tags 对应的 meters
    return meterStream;
}
```

#### 4.1.2.2 将 meters 转换为 Map<Statistic, Double> 形式

- 从这里可以看出 一个 meter 的 所有类型必须匹配 Statistic 枚举类中的类型

```java
private Map<Statistic, Double> getSamples(Collection<Meter> meters) {
   Map<Statistic, Double> samples = new LinkedHashMap<>();
   //遍历 meters，设置 samples，这些 meters 是经过 name 和 tags 过滤后的
   meters.forEach((meter) -> mergeMeasurements(samples, meter));
   return samples;
}

private void mergeMeasurements(Map<Statistic, Double> samples, Meter meter) {
    //调用 meter 的 measure 方法，遍历meter 返回的 List<Measurement>（每个 meter 实现类，都实现了这个方法）
    meter.measure().forEach((measurement) -> samples.merge(measurement.getStatistic(), measurement.getValue(),
                                                           mergeFunction(measurement.getStatistic())));
}
//对于一个meter 返回了 多个 Measurement,要遍历它们，按照 statistic 的类型进行 合并，合并规则如下
//即，如果有多个 count 类型，则合并这些 count 类型的值，作为总的 count 类型的值
private BiFunction<Double, Double, Double> mergeFunction(Statistic statistic) {
    return Statistic.MAX.equals(statistic) ? Double::max : Double::sum;
}
```

# 5 meter 示例

## 5.1 JvmGcMetrics

- JvmGcMetrics 是在 JvmMetricsAutoConfiguration 中注入到容器中的，并且是以 MeterBinder 的形式加入
- 在 3.1 节中，会将此 MeterBinder 注册到 CompositeMeterRegistry 中

```java
@NonNullApi
@NonNullFields
public class JvmGcMetrics implements MeterBinder, AutoCloseable {
	//通过监听 GarbageCollectorMXBean 更新 maxDataSize、allocatedBytes 等 metrix 参数！！
    @Override
    public void bindTo(MeterRegistry registry) {
		//为 registry 注册一个 gauge
        //最终这个 gauge 会以 meter 的形式加入到 此 registry 的 meterMap 中！！
        AtomicLong maxDataSize = new AtomicLong(0L);
        Gauge.builder("jvm.gc.max.data.size", maxDataSize, AtomicLong::get)
            .tags(tags)
            .description("Max size of old generation memory pool")
            .baseUnit(BaseUnits.BYTES)
            .register(registry);
		//为 registry 注册一个 counter
        Counter allocatedBytes = Counter.builder("jvm.gc.memory.allocated").tags(tags)
            .baseUnit(BaseUnits.BYTES)
            .description("Incremented for an increase in the size of the young generation memory pool after one GC to before the next")
            .register(registry);
    }
}
```

## 5.2 LogbackMetrics

```java
public class LogbackMetrics implements MeterBinder, AutoCloseable {
    @Override
    public void bindTo(MeterRegistry registry) {
        MetricsTurboFilter filter = new MetricsTurboFilter(registry, tags);
        metricsTurboFilters.put(registry, filter);
        //通过将此 filter 加入 loggerContext，logback 记录时会调用 filter 的 decide 方法来更改 logback metrics 参数
        loggerContext.addTurboFilter(filter);
    }
class MetricsTurboFilter extends TurboFilter {
    private final Counter errorCounter;
    MetricsTurboFilter(MeterRegistry registry, Iterable<Tag> tags) {
        errorCounter = Counter.builder("logback.events")
                .tags(tags).tags("level", "error")
                .description("Number of error level events that made it to the logs")
                .baseUnit("events")
                .register(registry);
    }
    public FilterReply decide(...) {
        Boolean ignored = LogbackMetrics.ignoreMetrics.get();
        if (ignored != null && ignored) {
            return FilterReply.NEUTRAL;
        }
        if (level.isGreaterOrEqual(logger.getEffectiveLevel()) && format != null) {
            switch (level.toInt()) {
                case Level.ERROR_INT:
                    errorCounter.increment();
                    break;
            }
        }
        return FilterReply.NEUTRAL;
    }
}
```

## 5.3 DataSourcePoolMetrics

```java
public class DataSourcePoolMetrics implements MeterBinder {
   @Override
   public void bindTo(MeterRegistry registry) {
      if (this.metadataProvider.getDataSourcePoolMetadata(this.dataSource) != null) {
         bindPoolMetadata(registry, "active", DataSourcePoolMetadata::getActive);
         bindPoolMetadata(registry, "idle", DataSourcePoolMetadata::getIdle);
         bindPoolMetadata(registry, "max", DataSourcePoolMetadata::getMax);
         bindPoolMetadata(registry, "min", DataSourcePoolMetadata::getMin);
      }
   }
   //bindPoolMetadata 最终会调用此方法 
   private <N extends Number> void bindDataSource(...) {
      //判断此 datasource 是否有对应的 DataSourcePoolMetadata
      if (function.apply(this.dataSource) != null) {
         //如果有就调用对应的 getActive/getIdle 等方法，进行 metrics 统计 
         registry.gauge("jdbc.connections." + metricName, this.tags, this.dataSource,
               (m) -> function.apply(m).doubleValue());
      }
   }
}
```

# 参考

spring-boot-2.2.1.RELEASE