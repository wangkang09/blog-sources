[TOC]

## 1 PrometheusMetricsExportAutoConfiguration

```java
public class PrometheusMetricsExportAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CollectorRegistry collectorRegistry() {
        return new CollectorRegistry(true);
    }
  
    @Bean
    @ConditionalOnMissingBean
    //注册一个 prometheusMeterRegistry，此 registry 会被注册到 compositeRegistry 中
    public PrometheusMeterRegistry prometheusMeterRegistry(...) {
        return new PrometheusMeterRegistry(prometheusConfig, collectorRegistry, clock);
    }
   //正真 /prometheus 端口生效的类  
   @Configuration(proxyBeanMethods = false)
   @ConditionalOnAvailableEndpoint(endpoint = PrometheusScrapeEndpoint.class)
   public static class PrometheusScrapeEndpointConfiguration {
      @Bean
      @ConditionalOnMissingBean
      public PrometheusScrapeEndpoint prometheusEndpoint(CollectorRegistry collectorRegistry) {
         return new PrometheusScrapeEndpoint(collectorRegistry);
      }
   }
    @Bean
    @ConditionalOnMissingBean//适配配置类
    public PrometheusConfig prometheusConfig(PrometheusProperties P) {
        return new PrometheusPropertiesConfigAdapter(P);
    } 
}
```