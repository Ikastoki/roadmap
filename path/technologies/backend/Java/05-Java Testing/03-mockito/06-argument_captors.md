# Argument captors

Cuando un metodo bajo prueba invoca a una dependencia pasando objetos que se construyen dentro del propio metodo, el test no tiene acceso directo a esos objetos para verificarlos. Los capturadores de argumentos (`ArgumentCaptor`) resuelven este problema capturando el objeto que se paso a un metodo del mock, permitiendo inspeccionarlo despues de la ejecucion.

Un `ArgumentCaptor` se declara con la anotacion `@Captor` o se crea manualmente con `ArgumentCaptor.forClass()`:

```java
@Captor
private ArgumentCaptor<Usuario> usuarioCaptor;

// Equivalente sin anotacion
private ArgumentCaptor<Usuario> usuarioCaptor = ArgumentCaptor.forClass(Usuario.class);
```

La captura se realiza durante la verificacion, pasando el captor donde iria un argumento:

```java
verify(repositorio).guardar(usuarioCaptor.capture());
```

Despues de esta llamada, `usuarioCaptor.getValue()` devuelve el objeto que se paso al metodo `guardar`. Para metodos invocados multiples veces, `getAllValues()` devuelve la lista completa de argumentos capturados:

```java
verify(repositorio, times(3)).guardar(usuarioCaptor.capture());
List<Usuario> usuariosCapturados = usuarioCaptor.getAllValues();
```

El captor permite inspeccionar propiedades del objeto capturado con los assert de JUnit o AssertJ:

```java
verify(repositorio).guardar(usuarioCaptor.capture());
Usuario usuarioCapturado = usuarioCaptor.getValue();

assertEquals("email@example.com", usuarioCapturado.getEmail());
assertNotNull(usuarioCapturado.getFechaCreacion());
assertTrue(usuarioCapturado.isActivo());
```

Cuando el metodo captura tiene multiples parametros, se necesita un captor por parametro:

```java
@Captor
private ArgumentCaptor<Long> idCaptor;

@Captor
private ArgumentCaptor<Usuario> usuarioCaptor;

@Test
void testActualizacion() {
    servicio.actualizar(1L, usuario);

    verify(repositorio).actualizar(idCaptor.capture(), usuarioCaptor.capture());

    assertEquals(Long.valueOf(1L), idCaptor.getValue());
    assertEquals(usuario, usuarioCaptor.getValue());
}
```

> Un error comun es usar `ArgumentCaptor` cuando `verify()` con matchers es suficiente. Si solo se necesita comprobar que el argumento es igual a un valor conocido, `verify(repositorio).guardar(usuario)` es mas directo.

`ArgumentCaptor` acepta tipos genericos, pero debido al borrado de tipos, se debe proporcionar la clase del tipo concreto. Para clases anidadas como `List<Usuario>`, se puede usar `ArgumentCaptor.forClass(List.class)` y luego realizar casting con supresion de warnings.

La captura con `@Captor` exige que el captor este correctamente inicializado. Si se usa `@ExtendWith(MockitoExtension.class)`, la inicializacion es automatica. En caso contrario, debe invocarse `MockitoAnnotations.openMocks(this)`.

Los argument captors son especialmente utiles en metodos `void` que no devuelven nada pero reciben objetos que el SUT construye. Por ejemplo, cuando el servicio crea una entidad y la persiste:

```java
@Captor
private ArgumentCaptor<Factura> facturaCaptor;

@Test
void testEmisionFactura() {
    servicio.emitirFactura(pedido);

    verify(almacen).guardar(facturaCaptor.capture());
    Factura factura = facturaCaptor.getValue();

    assertEquals(pedido.getTotal(), factura.getImporte());
    assertEquals(EstadoFactura.EMITIDA, factura.getEstado());
}
```

Sin `ArgumentCaptor`, verificar el contenido de la factura requeria modificar la implementacion del servicio o hacer que el metodo devolviera la factura solo para testing. El captor rompe esa dependencia y mantiene la interfaz del servicio limpia de metodos superfluos.
