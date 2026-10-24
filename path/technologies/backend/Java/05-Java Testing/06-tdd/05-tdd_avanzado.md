# TDD avanzado

A medida que los proyectos crecen en complejidad, el TDD basico necesita complementarse con enfoques que aborden multiples niveles de abstraccion y distintos estilos de diseno. Las escuelas de TDD ofrecen perspectivas diferentes sobre como comenzar y hacia donde evolucionar.

## Outside-in (London style)

El enfoque outside-in, tambien conocido como London school o mockist TDD, comienza desde la capa mas externa de la aplicacion (controladores, servicios de alto nivel) y desciende hacia las dependencias. Los tests se escriben para la unidad mas externa, y las dependencias se simulan con mocks o fakes.

Caracteristicas principales:
- Se disena la interaccion entre objetos antes que la implementacion interna
- Los tests definen el contrato entre componentes
- Se usa extensivamente mocks para aislar la unidad bajo test
- El diseno emerge de las necesidades de comunicacion entre objetos

```java
@Test
void shouldCompleteOrderWhenPaymentSucceeds() {
    PaymentGateway gateway = mock(PaymentGateway.class);
    InventoryService inventory = mock(InventoryService.class);
    when(gateway.charge(anyDouble())).thenReturn(true);

    OrderService service = new OrderService(gateway, inventory);
    Order order = service.placeOrder(new Cart(List.of("item1")), "user123");

    assertNotNull(order.id());
    verify(inventory).reserve("item1", order.id());
}
```

> London school es ideal cuando se trabaja con sistemas con multiples capas y dependencias claramente definidas. Favorece el diseno por contrato y la separation of concerns.

## Inside-out (Chicago style)

El enfoque inside-out, o Chicago school (tambien llamado classicist TDD), comienza desde las unidades de dominio mas pequenas y va creciendo hacia afuera. Se prefieren fakes y estados reales sobre mocks, y los tests tienden a ser mas integradores incluso a nivel unitario.

Caracteristicas principales:
- Se comienza por las entidades de domino y reglas de negocio
- Se usan fakes en lugar de mocks cuando es posible
- Los tests verifican estado, no interaccion
- La integracion entre componentes se prueba con tests de integracion separados

```java
@Test
void shouldReserveItemsFromInventory() {
    Inventory inventory = new Inventory();
    inventory.add("item1", 10);

    inventory.reserve("item1", 5);

    assertEquals(5, inventory.availableQuantity("item1"));
}
```

> Chicago school es preferible cuando el dominio de negocio es complejo y se quiere mantener los tests mas cercanos al comportamiento real, evitando el acoplamiento a las interacciones internas.

## Acceptance TDD

El Acceptance TDD (ATDD) opera a nivel de requisitos funcionales. Los tests de aceptacion se escriben antes que el codigo de produccion y describen el comportamiento del sistema desde la perspectiva del usuario. Sirven como criterio de aceptacion para las historias de usuario.

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class OrderAcceptanceTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("Usuario completa una compra exitosa")
    void userCompletesSuccessfulPurchase() {
        String response = restTemplate.postForObject(
            "/orders",
            new OrderRequest(List.of("SKU-001"), "user@example.com"),
            String.class
        );
        assertNotNull(response);
        assertTrue(response.contains("orderId"));
    }
}
```

El flujo de ATDD es:
1. Se escribe un test de aceptacion que falla
2. Se escribe el test unitario para la primera unidad involucrada (TDD clasico)
3. Se implementa el codigo hasta que el test de aceptacion pase

## Property-based testing

El property-based testing (PBT) complementa al example-based testing verificando propiedades invariantes del sistema con miles de entradas generadas automaticamente. Herramientas como jqwik o quickcheck permiten expresar propiedades en lugar de ejemplos concretos.

```java
import net.jqwik.api.*;

class MathProperties {

    @Property
    void absoluteValueIsAlwaysNonNegative(@ForAll int value) {
        assertTrue(Math.abs(value) >= 0);
    }

    @Property
    void doublingIsEquivalentToAddingItself(@ForAll int value) {
        assertEquals(value * 2, value + value);
    }
}
```

PBT descubre casos de esquina que el desarrollador no habria considerado y es particularmente util para validar algoritmos, transformaciones de datos y logica matematica.

## Example-based vs Property-based

| Aspecto | Example-based | Property-based |
|---------|---------------|----------------|
| Entradas | Fijas, elegidas manualmente | Generadas automaticamente |
| Cobertura | Limitada a casos conocidos | Amplia, cubre casos no previstos |
| Especificacion | Comportamiento concreto | Invariante o propiedad general |
| Coste de escritura | Bajo, casos explicitos | Alto, requiere abstraer la propiedad |
| Mantenimiento | Fragil si cambia la logica | Robusto porque valida propiedades |

Ambos enfoques no son excluyentes. La practica recomendada es usar example-based para el desarrollo iterativo (TDD clasico) y property-based para reforzar las propiedades criticas del dominio.
