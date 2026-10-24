# Contract Testing

El contract testing verifica que los productores y consumidores de una API mantienen un contrato compatible sobre la estructura de los datos que intercambian. En arquitecturas de microservicios, donde un servicio depende de las respuestas de multiples servicios upstream, los cambios no compatibles en una API pueden provocar fallos en cascada que solo se detectan en produccion. El contract testing detecta estas incompatibilidades antes del despliegue, proporcionando una red de seguridad entre los equipos que desarrollan servicios interdependientes.

Spring Extension Contract implementa este patron utilizando contratos definidos como codigo que se ejecutan tanto en el lado del productor como en el del consumidor. Los contratos especifican la estructura de las peticiones y respuestas esperadas, y se generan automaticamente stubs (simuladores) que permiten al consumidor probar su codigo contra una implementacion predecible del productor.

## Producer/Consumer Contracts

Un contrato describe la interaccion entre un productor y un consumidor: la peticion que el consumidor envia y la respuesta que el productor debe retornar. Los contratos se definen en un lenguaje declarativo Groovy o YAML que especifica las partes de la interaccion.

```groovy
// Contrato definido en Groovy
Contract.make {
    description "Deberia retornar una orden por su ID"
    request {
        method GET()
        url('/api/orders/123')
        headers {
            contentType(applicationJson())
        }
    }
    response {
        status OK()
        body(
            id: 123,
            status: 'CREATED',
            total: 99.99
        )
        headers {
            contentType(applicationJson())
        }
    }
}
```

```yaml
# Contrato definido en YAML
description: Deberia retornar una orden por su ID
request:
  method: GET
  url: /api/orders/123
  headers:
    Content-Type: application/json
response:
  status: 200
  body:
    id: 123
    status: CREATED
    total: 99.99
  headers:
    Content-Type: application/json
```

> Los contratos deben ser lo suficientemente especificos para detectar cambios incompatibles, pero lo suficientemente flexibles para permitir evoluciones compatibles. Definir cada campo exacto genera contratos fragiles; enfocarse en los campos criticos del contrato proporciona la proteccion necesaria sin crear friccion innecesaria.

## Stub Runner

El Stub Runner genera y ejecuta stubs automaticos a partir de los contratos definidos. Durante las pruebas del consumidor, el Stub Runner levanta un servidor HTTP que simula las respuestas del productor segun los contratos, permitiendo que el consumidor valide su logica de integracion sin depender del servicio real.

```java
@SpringBootTest
@AutoConfigureStubRunner(
        ids = "com.example:order-service:+:stubs:8080",
        stubsMode = StubRunnerProperties.StubsMode.LOCAL)
class OrderConsumerContractTest {

    @Autowired
    private OrderClient orderClient;

    @Test
    void shouldParseOrderResponse() {
        Order order = orderClient.getOrder("123");

        assertThat(order.getId()).isEqualTo(123L);
        assertThat(order.getStatus()).isEqualTo(Status.CREATED);
        assertThat(order.getTotal()).isEqualByComparingTo(BigDecimal.valueOf(99.99));
    }
}
```

El parametro `ids` del `@AutoConfigureStubRunner` sigue el formato `{groupId}:{artifactId}:{version}:{classifier}:{port}`. El classifier `stubs` indica que se buscan los artefactos de stubs generados por el productor.

```xml
<!-- Dependencia del Stub Runner en el proyecto del consumidor -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

> El Stub Runner puede ejecutarse en modo LOCAL (descarga los stubs de un repositorio Maven) o en modo REMOTE (descarga los stubs de un repositorio Git o un servidor de stubs). El modo REMOTE es preferido en pipelines de CI/CD para mantener los stubs sincronizados con la version mas reciente del productor.

## WireMock

WireMock es una libreria de simulacion HTTP que Spring Extension Contract utiliza como motor subyacente para los stubs. Cada contrato se traduce a un mapeo WireMock que define la peticion esperada y la respuesta a retornar. Esto permite utilizar funcionalidades avanzadas de WireMock como request matching, response templating y simulators de latencia.

```java
@SpringBootTest
@AutoConfigureWireMock(port = 9090)
class OrderIntegrationTest {

    @WireMockTest
    WireMockRuntimeInfo wmRuntimeInfo;

    @Test
    void shouldStubOrderEndpoint() {
        stubFor(get(urlEqualTo("/api/orders/123"))
                .withHeader("Accept", containing("application/json"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("{\"id\":123,\"status\":\"CREATED\"}")));

        // La prueba utiliza el stub configurado
    }
}
```

## Flujo de Trabajo

El flujo tipico de contract testing en un ecosistema de microservicios sigue un ciclo donde los contratos se definen, generan y ejecutan en ambos lados:

1. El productor define los contratos de su API en su repositorio.
2. Al ejecutar las pruebas del productor, los contratos se verifican contra la implementacion real.
3. Los contratos verificados se publican como artefactos Maven con classifier `stubs`.
4. El consumidor descarga los stubs y los utiliza en sus propias pruebas de integracion.
5. Si el consumidor falla contra los stubs, el contrato se ha roto y debe resolverse antes del despliegue.

```yaml
# Configuracion del plugin de Spring Cloud Contract en el productor
spring-cloud-contract:
  verifier:
    contractsMode: EXACT
    basePackage: com.example.orders
    testMode: MOCKMVC
  contractDependencies:
    groupId: com.example
    artifactId: order-service-contracts
    version: 1.0.0
```

```java
// Prueba del productor que verifica los contratos
@SpringBootTest
@AutoConfigureMockMvc
class OrderControllerContractTest extends ContractVerifierTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    void validateOrderApiContract() {
        mockMvc.perform(get("/api/orders/123")
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(123))
                .andExpect(jsonPath("$.status").value("CREATED"));
    }
}
```

> Los contratos deben versionarse de forma independiente de los servicios. Un cambio incompatible en la API debe incrementar la version del contrato y generar una nueva version de los stubs, permitiendo que los consumidores migren gradualmente.

El contract testing reduce significativamente el riesgo de despliegues en sistemas de microservicios al proporcionar una verificacion automatica de la compatibilidad entre servicios. La inversion en contratos claros y bien mantenidos se traduce en menos incidentes en produccion y mayor confianza en los ciclos de despliegue continuo.

