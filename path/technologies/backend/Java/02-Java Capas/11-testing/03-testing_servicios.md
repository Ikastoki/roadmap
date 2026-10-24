# Testing de servicios

La capa de servicios contiene las reglas de negocio de la aplicacion. A diferencia de los repositorios --que principalmente persisten y recuperan datos-- los servicios toman decisiones: validan condiciones, calculan valores, orquestan repositorios y lanzan excepciones cuando las reglas se incumplen. Probar un servicio es, en esencia, probar que esas decisiones son correctas.

Un servicio bien disenado recibe sus dependencias por constructor y no crea objetos concretos por si mismo. Esta inyeccion de dependencias hace que cualquier repositorio, servicio externo o componente auxiliar sea sustituible por un mock, lo que permite aislar la logica de negocio de la infraestructura subyacente.

## Mock de repositories

El repositorio es la dependencia principal del servicio. Mockearlo permite controlar que datos devuelve y verificar que el servicio lo invoca correctamente. La verificacion de interacciones --comprobar que se llamo a un metodo con determinados argumentos-- es tan importante como la asercion del resultado.

```java
class OrderServiceTest {

    private OrderRepository orderRepository;
    private ProductRepository productRepository;
    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderRepository = mock(OrderRepository.class);
        productRepository = mock(ProductRepository.class);
        orderService = new OrderService(orderRepository, productRepository);
    }

    @Test
    void shouldCreateOrderWhenStockIsSufficient() {
        Long productId = 1L;
        Product product = new Product(productId, "Laptop", BigDecimal.valueOf(1000), 5);
        CreateOrderRequest request = new CreateOrderRequest(productId, 2);

        when(productRepository.findById(productId)).thenReturn(Optional.of(product));
        when(orderRepository.save(any(Order.class))).thenAnswer(invocation -> invocation.getArgument(0));

        OrderDTO result = orderService.createOrder(request);

        assertNotNull(result);
        assertEquals(BigDecimal.valueOf(2000), result.total());
        verify(productRepository).updateStock(productId, 3);
        verify(orderRepository).save(any(Order.class));
    }

    @Test
    void shouldThrowExceptionWhenProductNotFound() {
        when(productRepository.findById(99L)).thenReturn(Optional.empty());

        assertThrows(NotFoundException.class, () -> {
            orderService.createOrder(new CreateOrderRequest(99L, 1));
        });

        verify(orderRepository, never()).save(any());
    }

    @Test
    void shouldThrowExceptionWhenStockIsInsufficient() {
        Product product = new Product(1L, "Laptop", BigDecimal.valueOf(1000), 1);
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));

        assertThrows(BusinessException.class, () -> {
            orderService.createOrder(new CreateOrderRequest(1L, 5));
        });

        verify(orderRepository, never()).save(any());
    }
}
```

Cada test cubre un escenario: flujo feliz, recurso no encontrado y regla de negocio violada. Los mocks se configuran con `when().thenReturn()` para los repositorios y se verifica con `verify()` que el servicio llamo (o no llamo) a los metodos esperados.

> La regla "nunca mockear lo que no se posee" se matiza aqui: los repositorios son interfaces definidas en el proyecto, por lo que mockearlos es seguro y recomendado. Lo que no debe mockearse es la logica del propio servicio bajo prueba.

## Verificar logica de negocio

Las reglas de negocio son la razon de ser del servicio. Cada regla debe tener al menos un test que verifique su cumplimiento y otro que verifique su violacion. La estructura Given-When-Then ayuda a mantener los tests claros: datos preparados, accion ejecutada, resultado verificado.

```java
@Test
void shouldApplyDiscountForLoyalCustomers() {
    User user = new User(1L, "Ana", "ana@example.com", 15); // 15 pedidos previos
    when(userRepository.findById(1L)).thenReturn(Optional.of(user));
    when(orderRepository.countByUserId(1L)).thenReturn(15);

    BigDecimal discount = userService.calculateDiscount(1L);

    assertEquals(new BigDecimal("10.00"), discount); // 10 % de descuento
}

@Test
void shouldNotApplyDiscountForNewCustomers() {
    User user = new User(2L, "Luis", "luis@example.com", 2);
    when(userRepository.findById(2L)).thenReturn(Optional.of(user));
    when(orderRepository.countByUserId(2L)).thenReturn(2);

    BigDecimal discount = userService.calculateDiscount(2L);

    assertEquals(BigDecimal.ZERO, discount);
}
```

Las validaciones de entrada que pertenecen al servicio --no las validaciones sintacticas del controller, sino las reglas semanticas del dominio-- tambien deben probarse aqui.

```java
@Test
void shouldRejectNegativeOrderQuantities() {
    assertThrows(IllegalArgumentException.class, () -> {
        orderService.createOrder(new CreateOrderRequest(1L, -1));
    });
}

@Test
void shouldRejectOrderWhenEmailDomainIsNotAllowed() {
    CreateUserRequest request = new CreateUserRequest("Test", "test@spam.com");
    when(userRepository.existsByEmail("test@spam.com")).thenReturn(false);

    assertThrows(BusinessException.class, () -> {
        userService.createUser(request);
    });
}
```

## Verificar transacciones

Las transacciones garantizan que una secuencia de operaciones se ejecuta de forma atomica: o todas se completan o ninguna deja rastro. En Spring, `@Transactional` maneja esto automaticamente, pero el comportamiento transaccional debe verificarse explícitamente.

El escenario critico es la operacion que persiste varios objetos y falla a mitad del proceso. Sin transaccion, los objetos anteriores quedarian en la base de datos en un estado inconsistente.

```java
@Test
void shouldRollbackWhenPartialOperationFails() {
    when(productRepository.findById(1L)).thenReturn(Optional.of(new Product(1L, "Laptop", BigDecimal.valueOf(1000), 5)));
    when(productRepository.findById(2L)).thenReturn(Optional.of(new Product(2L, "Mouse", BigDecimal.valueOf(50), 10)));
    doThrow(new RuntimeException("Fallo en la actualizacion de stock"))
        .when(productRepository).updateStock(eq(2L), anyInt());

    assertThrows(RuntimeException.class, () -> {
        orderService.createBulkOrder(List.of(
            new CreateOrderRequest(1L, 1),
            new CreateOrderRequest(2L, 1)
        ));
    });

    verify(productRepository).updateStock(1L, 4);
    verify(orderRepository, never()).save(any());
}
```

En este ejemplo, el servicio falla al actualizar el stock del segundo producto. Si no hubiera transaccion, la actualizacion del stock del primer producto (ejecutada antes del fallo) se habria persistido. El test verifica que `save()` no se llamo nunca y que el stock del primer producto quedo intacto.

> Para verificar el comportamiento transaccional real es necesario un test de integracion con una base de datos real o en memoria, ya que los mocks no gestionan transacciones. Un test unitario solo puede verificar que se lanzaron las excepciones adecuadas.

La verificacion de transacciones tambien incluye comprobar que no se abren transacciones innecesarias para operaciones de solo lectura, y que la propagacion de transacciones entre servicios que se llaman entre si se comporta como se espera.
