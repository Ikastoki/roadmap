# Load Balancing

El balanceo de carga en microservicios resuelve el problema de distribuir las peticiones entrantes entre multiples instancias del mismo servicio. Cuando un microservicio se escala horizontalmente para manejar mayor trafico, los clientes necesitan un mecanismo que seleccione automaticamente una instancia disponible sin configurar direcciones IP estaticas. Spring Cloud LoadBalancer proporciona esta funcionalidad como alternativa ligera a Netflix Ribbon, que ha sido deprecado en favor de esta implementacion.

El balanceo de carga opera a nivel de cliente: cada microservicio que consume otro servicio incorpora un balanceador de carga que resuelve el nombre del servicio a traves del service registry y selecciona una instancia segun la estrategia configurada. Este enfoque descentralizado elimina un punto unico de fallo y reduce la latencia al evitar un proxy centralizado.

## @LoadBalanced

La anotacion `@LoadBalanced` se aplica a un bean `RestTemplate` o `WebClient` para habilitar el balanceo de carga automatico. Cuando un cliente utiliza una URL con el esquema `lb://` (por ejemplo, `lb://order-service/api/orders`), el interceptor de load balancing resuelve el nombre del servicio a traves del DiscoveryClient y selecciona una instancia.

```java
@Configuration
public class LoadBalancedConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
                .setConnectTimeout(Duration.ofSeconds(3))
                .setReadTimeout(Duration.ofSeconds(5))
                .build();
    }

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder()
                .filter(ExchangeFilterFunctions
                        .basicAuthentication("user", "password"));
    }
}
```

```java
@Service
public class OrderClient {

    private final RestTemplate restTemplate;

    public OrderClient(@LoadBalanced RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public Order getOrder(String orderId) {
        // Spring Cloud LoadBalancer resuelve "order-service" a una instancia real
        return restTemplate.getForObject(
                "http://order-service/api/orders/" + orderId,
                Order.class);
    }
}
```

> Cuando se utiliza `@LoadBalanced`, el nombre del servicio en la URL se reemplaza por la direccion IP y puerto de la instancia seleccionada antes de enviar la peticion. Este proceso es transparente para el codigo de la aplicacion.

## ServiceInstanceListSupplier

El `ServiceInstanceListSupplier` es la interfaz que define como se obtiene la lista de instancias disponibles para un servicio determinado. Spring Cloud LoadBalancer ofrece multiples implementaciones que consultan el service registry y aplican filtros sobre la lista de instancias.

```java
@Configuration
public class CustomLoadBalancerConfig {

    @Bean
    public ServiceInstanceListSupplier discoveryServiceInstanceListSupplier(
            DiscoveryClient discoveryClient) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withHealthChecks()
                .build();
    }
}
```

Los suppliers disponibles incluyen `DiscoveryClientServiceInstanceListSupplier` que consulta el service registry, `CachingServiceInstanceListSupplier` que cachea los resultados para evitar consultas frecuentes, y `HealthCheckServiceInstanceListSupplier` que filtra instancias no saludables.

## Round Robin

La estrategia Round Robin es la predeterminada de Spring Cloud LoadBalancer. Distribuye las peticiones ciclicamente entre todas las instancias disponibles, garantizando que cada instancia reciba aproximadamente el mismo numero de peticiones.

```java
@Configuration
public class LoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> roundRobinLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

En entornos con multiples instancias de capacidades similares, Round Robin es la estrategia ideal ya que maximiza la distribucion uniforme del trafico. Su simplicidad lo hace predecible y facil de monitorear.

## Random

La estrategia Random selecciona una instancia de forma aleatoria entre las disponibles. A diferencia de Round Robin, no garantiza una distribucion uniforme, pero en conjuntos de instancias suficientemente grandes la distribucion tiende a ser equitativa.

```java
@Configuration
public class RandomLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

## Weight

Las estrategias de peso permiten distribuir el trafico de forma desproporcionada entre instancias de diferentes capacidades. Una instancia con el doble de capacidad recibe el doble de peticiones. Esto es util en escenarios donde las instancias se ejecutan en maquinas con recursos variables o donde se realizan despliegues graduales con versiones diferentes del servicio.

```yaml
spring:
  cloud:
    loadbalancer:
      configurations:
        weighted:
          enabled: true
```

```java
@Configuration
public class WeightedLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> weightedLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new WeightedLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

> Las instancias pueden etiquetarse con pesos a traves de metadatos en el service registry. En Eureka, esto se configura con la propiedad `eureka.instance.metadata-map.weight`, y en Consul con anotaciones de servicio.

## Caching y Health Checks

Spring Cloud LoadBalancer cachea la lista de instancias resueltas para evitar consultas frecuentes al service registry. La cache se invalida periodicamente segun la TTL configurada. Ademas, el integrador de health checks filtra automaticamente las instancias que no responden a los endpoints de salud, evitando que se envien peticiones a servicios caidos.

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 30s
        capacity: 256
      health-check:
        path: /actuator/health
        interval: 25s
        delayed:
          initial-delay: 5000
```

La combinacion de estas estrategias permite construir sistemas de balanceo de carga que se adaptan a las necesidades especificas de cada ecosistema de microservicios, desde simples configuraciones Round Robin hasta esquemas sofisticados con pesos, health checks y cacheo inteligente.

