# Service Discovery

El descubrimiento de servicios es el mecanismo que permite a los microservicios encontrarse dinamicamente en un entorno distribuido. Sin este patron, cada servicio necesitaria configurar explicitamente las direcciones y puertos de todos los servicios con los que se comunica, lo que generaria una friccion operativa inaceptable en sistemas con multiples instancias que se escalan y despliegan de forma independiente.

El patron fundamental es un **service registry** centralizado donde cada微servicio registra su ubicacion (host, puerto, metadatos) al iniciar y se desregistra al apagarse. Los clientes consultan este registro para resolver nombres de servicio a direcciones concretas, delegando al registry la responsabilidad de mantener la informacion actualizada.

## Eureka Server

Netflix Eureka es la implementacion de service registry mas utilizada en el ecosistema de Spring Cloud. Un Eureka Server es un servicio independiente que mantiene una tabla en memoria de todas las instancias registradas, ejecutando periodicamente tareas de renovacion y limpieza de registros expirados.

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: true
    eviction-interval-timer-in-ms: 60000
```

> La auto-preservacion de Eureka es un mecanismo de seguridad que evita que el servidor elimine registros de servicios sanos cuando la red presenta particiones. Si la tasa de renuncias cae por debajo de un umbral, Eureka asume que existe un problema de red y deja de expirar registros.

## Eureka Client

Cada microservicio que necesite descubrir o ser descubierto incorpora el cliente de Eureka. Al iniciar, el cliente registra la informacion del servicio en el Eureka Server y ejecuta periodicamente una tarea de renovacion (heartbeat) para confirmar que la instancia sigue activa. Si una instancia deja de enviar heartbeats, Eureka la marca como candidata a eliminacion despues de un periodo de expiracion configurado.

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```yaml
spring:
  application:
    name: order-service

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
    registry-fetch-interval-seconds: 5
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

La resolucion de servicios funciona a traves de `DiscoveryClient`, que consulta el registro de Eureka y retorna una lista de instancias disponibles para un nombre de servicio determinado. Spring Cloud Ribbon integra esta resolucion con el balanceo de carga, permitiendo que los clientes HTTP distribuyan las peticiones entre las instancias descubiertas.

```java
@Service
public class InventoryServiceClient {

    private final DiscoveryClient discoveryClient;
    private final RestTemplate restTemplate;

    public InventoryServiceClient(DiscoveryClient discoveryClient,
                                  RestTemplate restTemplate) {
        this.discoveryClient = discoveryClient;
        this.restTemplate = restTemplate;
    }

    public List<ServiceInstance> getInventoryInstances() {
        return discoveryClient.getInstances("inventory-service");
    }

    public Inventory checkStock(String productId) {
        String url = "http://inventory-service/api/stock/" + productId;
        return restTemplate.getForObject(url, Inventory.class);
    }
}
```

## @EnableDiscoveryClient vs @EnableEurekaClient

La anotacion `@EnableDiscoveryClient` es una abstraccion generica del ecosistema Spring Cloud que funciona con cualquier implementacion de descubrimiento (Eureka, Consul, Zookeeper). La anotacion `@EnableEurekaClient` es especifica de Eureka. En la practica, `@EnableDiscoveryClient` es la opcion recomendada ya que permite cambiar de implementacion sin modificar el codigo de la aplicacion.

## Consul as Alternative

HashiCorp Consul ofrece una alternativa a Eureka con funcionalidades adicionales como service health checks nativos, soporte para KV store y un modelo de servicios con multiples checks. Spring Cloud Consul integra Consul como service registry con una experiencia similar a la de Eureka.

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConsulClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsulClientApplication.class, args);
    }
}
```

```yaml
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: ${spring.application.name}
        health-check-path: /actuator/health
        health-check-interval: 15s
        instance-id: ${spring.application.name}:${random.value}
```

## Service Registration

El proceso de registro sigue un ciclo de vida bien definido: el servicio inicia y se registra, envia heartbeats periodicos para confirmar su disponibilidad, y se desregistra al recibir una senal de apagado (SIGTERM). Si el servicio no puede desregistrarse limpiamente (por ejemplo, ante un fallo del proceso), el registry expira el registro despues de un periodo configurado.

```
Servicio inicia -> Registro en Eureka
       |
       v
  Heartbeat cada 10s ----> Eureka actualiza timestamp
       |
       v
  Servicio se detiene -> Desregistro limpio
       |
       v
  Eureka elimina registro -> Clientes dejan de recibir la instancia
```

La resolucion de nombres en el lado del cliente se basa en la cache local del registro. Cada cliente mantiene una copia actualizada del registro que refresca periodicamente (por defecto cada 30 segundos). Esto introduce un delay eventual en la propagacion de cambios, que es aceptable en la mayoria de los escenarios pero debe considerarse en situaciones de escalado rapido.

> En entornos con multiples zonas de disponibilidad, Eureka ofrece la configuracion de zonas que permite priorizar instancias en la misma zona antes de recurrir a instancias en zonas remotas, reduciendo la latencia de red.

