# Excepciones checked

Java es uno de los pocos lenguajes modernos que distingue entre excepciones checked y unchecked. Las excepciones checked (`Exception` y sus subclases, excepto `RuntimeException`) deben declararse en la clausula `throws` del metodo o capturarse explicitamente. Esta particularidad afecta a la forma de escribir tests.

## Checked exceptions en tests

Un metodo de test puede declarar `throws Exception` sin tener que capturar cada excepcion checked individualmente. JUnit 5 trata cualquier excepcion lanzada desde un test como un fallo, por lo que declarar `throws Exception` es una practica aceptada y comun.

```java
@Test
void procesoCorrectamente() throws Exception {
    byte[] datos = Files.readAllBytes(Paths.get("archivo.txt"));
    Resultado resultado = servicio.procesar(datos);
    assertThat(resultado.esValido()).isTrue();
}
```

Declarar `throws Exception` en lugar de la excepcion concreta simplifica el codigo cuando el metodo bajo prueba lanza multiples excepciones checked o cuando se usan APIs de I/O que declaran `IOException`. No obstante, si el test solo interactua con una excepcion concreta, es mejor declararla explicitamente para documentar el comportamiento esperado.

## throws Exception como regla

Es frecuente ver tests que declaran `throws Exception` aunque el metodo bajo prueba no lance ninguna excepcion checked. Esto es seguro pero enganoso: oculta informacion sobre que puede fallar realmente. La recomendacion es declarar `throws Exception` solo cuando sea necesario y, cuando no, dejar la declaracion vacia.

```java
// Mejor: explicito
@Test
void calculoCorrecto() {
    assertThat(calculador.sumar(2, 2)).isEqualTo(4);
}

// Aceptable pero menos informativo
@Test
void calculoCorrecto() throws Exception {
    assertThat(calculador.sumar(2, 2)).isEqualTo(4);
}
```

## Throwable assertions

Cuando se usa AssertJ, las aserciones sobre excepciones checked utilizan el mismo mecanismo que las unchecked. `assertThatThrownBy` acepta cualquier `Throwable`.

```java
@Test
void ioExceptionEnLectura() {
    Path ruta = Paths.get("/no/existe");

    assertThatThrownBy(() -> Files.readAllBytes(ruta))
        .isInstanceOf(IOException.class)
        .hasMessageContaining("/no/existe");
}
```

Si el metodo bajo prueba lanza una excepcion checked, `assertThrows` de JUnit 5 tambien funciona sin distincion.

```java
@Test
void lanzaCheckedException() {
    assertThrows(IOException.class, () -> {
        Files.readAllBytes(Paths.get("/no/existe"));
    });
}
```

## Metodos que lanzan checked exceptions en mocks

Al usar Mockito con metodos que declaran checked exceptions, se debe especificar la excepcion en `thenThrow`:

```java
@Test
void servicioLanzaIOException() throws IOException {
    ServicioExterno mock = mock(ServicioExterno.class);
    when(mock.llamar()).thenThrow(new IOException("Error de red"));

    assertThatThrownBy(() -> mock.llamar())
        .isInstanceOf(IOException.class);
}
```

Mockito permite lanzar checked exceptions incluso si el metodo no las declara en su firma (usando `willThrow` en lugar de `thenThrow`), pero esto rompe el contrato del metodo y debe evitarse.

> En tests no es necesario capturar checked exceptions con try-catch a menos que se quiera verificar un comportamiento posterior a la excepcion. `assertThrows` y `assertThatThrownBy` manejan la captura de forma transparente.
