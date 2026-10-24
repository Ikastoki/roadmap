# Test Smells

Los test smells son sintomas de que un test tiene problemas de diseno, mantenibilidad o fiabilidad. No son errores de logica, pero indican que el test se volvera problematico con el tiempo.

## Eager Test

Un test eager verifica demasiadas cosas a la vez. Cuando falla, no se sabe que parte del comportamiento se rompio. La regla es: un test debe verificar un unico comportamiento.

```java
// Mal: eager test
@Test
void testCompleto() {
    Usuario u = servicio.crear("user@mail.com");
    assertThat(u.getId()).isNotNull();
    assertThat(u.getEmail()).isEqualTo("user@mail.com");
    assertThat(u.isActivo()).isTrue();
    assertThat(u.getRol()).isEqualTo(Rol.USUARIO);
    assertThat(servicio.enviarBienvenida(u)).isTrue();
}

// Bien: tests separados
@Test void creaUsuarioConId() { ... }
@Test void creaUsuarioConEmail() { ... }
@Test void creaUsuarioActivo() { ... }
@Test void enviaBienvenida() { ... }
```

## Mystery Guest

Un mystery guest es un test que utiliza recursos externos (base de datos, archivos, servicios web) sin que quede explicito en el codigo del test. El lector no sabe que datos se estan usando ni de donde vienen.

```java
// Mal: mystery guest (usa datos de base de datos real)
@Test
void calculaTotal() {
    double total = facturaService.calcular("F-001"); // ? que factura es?
    assertThat(total).isEqualTo(250.0);
}

// Bien: datos explicitos en el test
@Test
void calculaTotal() {
    Factura factura = new Factura();
    factura.agregarLinea(new Linea("Producto", 100.0, 2));
    double total = facturaService.calcular(factura);
    assertThat(total).isEqualTo(200.0);
}
```

## Remote Control

Un test que depende de un servicio remoto (API externa, base de datos en otro servidor) es lento, fragil y no reproducible. Cualquier problema de red o cambio en el servicio externo hara fallar el test sin que el codigo propio tenga la culpa.

```java
// Mal: depende de API externa
@Test
void consultaClima() {
    Clima clima = servicioClima.obtener("Madrid");
    assertThat(clima.getTemperatura()).isNotNull();
}

// Bien: mock del servicio externo
@Test
void consultaClima() {
    when(servicioClima.obtener("Madrid"))
        .thenReturn(new Clima(25.0));
    // ...
}
```

## Slow Test

Un test lento desincentiva su ejecucion frecuente. Cuando la suite de tests tarda minutos, los desarrolladores dejan de ejecutarla y los bugs llegan a produccion.

## Fragile Test

Un test fragil falla por cambios que no deberian afectarlo. Por ejemplo, un test que depende del orden de los elementos en un `HashMap`, del formato exacto de un mensaje, o de la implementacion interna de un metodo.

## Excessive Setup

Un test que requiere decenas de lineas de configuracion antes de la asercion es dificil de leer y mantener. El exceso de setup suele indicar que el codigo bajo prueba tiene demasiadas dependencias o que el test no esta correctamente aislado.

```java
// Mal: excessive setup
@Test
void procesaPedido() {
    Usuario u = new Usuario("u1", "user@mail.com", "PASS", true, Rol.ADMIN);
    Producto p1 = new Producto("p1", "Laptop", 1200.0, Categoria.ELECTRONICA);
    Producto p2 = new Producto("p2", "Mouse", 25.0, Categoria.ELECTRONICA);
    Direccion d = new Direccion("Calle", "Ciudad", "28001", "Madrid", "Espana");
    Pedido pedido = new Pedido("P-001", u, List.of(p1, p2), d, MetodoPago.TARJETA);
    ConfigEnvio config = new ConfigEnvio(true, false, "Estandar");

    Resultado resultado = servicio.procesar(pedido, config, "ADMIN", true);

    assertThat(resultado.isExitoso()).isTrue();
}
```

Para simplificar el setup se pueden usar metodos factory, builders o la anotacion `@BeforeEach`.

```java
@Test
void procesaPedido() {
    Pedido pedido = new PedidoBuilder()
        .conUsuario("u1", "user@mail.com")
        .conProducto("Laptop", 1200.0)
        .conProducto("Mouse", 25.0)
        .build();

    assertThat(servicio.procesar(pedido).isExitoso()).isTrue();
}
```

Identificar test smells requiere practica y revision. Los code reviews de tests son el momento ideal para detectarlos y corregirlos antes de que se consoliden en la base de codigo.
