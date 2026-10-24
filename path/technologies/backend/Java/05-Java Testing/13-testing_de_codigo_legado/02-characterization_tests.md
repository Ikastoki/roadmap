# Characterization tests

Los characterization tests son la herramienta principal para abordar codigo legacy del que no se tiene documentacion ni tests. Su objetivo no es verificar la correccion, sino capturar el comportamiento actual del sistema para poder refactorizar con una red de seguridad.

## Grabar comportamiento actual

Cuando un metodo no tiene tests, cualquier cambio puede alterar su comportamiento sin que nadie lo note. El primer paso es ejecutar el metodo con un conjunto representativo de entradas y registrar las salidas.

```java
@Test
void caracterizacionProcesarPedido() {
    Pedido pedido = new Pedido();
    pedido.setCliente("C-001");
    pedido.setTotal(250.0);

    String resultado = servicio.procesarPedido(pedido);

    // El test captura lo que SÍ hace, no lo que DEBERIA hacer
    assertThat(resultado).isEqualTo("PEDIDO_ACEPTADO");
    assertThat(pedido.getEstado()).isEqualTo(Estado.CONFIRMADO);
    assertThat(pedido.getImpuesto()).isEqualByComparingTo(52.5);
}
```

El test se ejecuta y se observa el resultado. Si el test pasa, queda registrado el comportamiento actual. A partir de ese momento, cualquier cambio que modifique el comportamiento hara fallar el test.

## Testear lo que es, no lo que deberia ser

La tentacion con los characterization tests es corregir el codigo mientras se escriben los tests: "este metodo deberia devolver X, no Y". Eso es un error. El objetivo es capturar el comportamiento actual, aunque sea incorrecto.

```java
// El metodo actual devuelve null para clientes VIP, aunque deberia
// devolver un descuento del 20%. El test captura el bug existente.
@Test
void caracterizacionDescuentoVip() {
    Cliente vip = new Cliente("VIP", 100.0);
    Double descuento = servicio.calcularDescuento(vip);
    assertThat(descuento).isNull(); // comportamiento actual, aunque erroneo
}
```

Una vez que el comportamiento actual esta registrado, se puede anadir un ticket para corregir el bug y modificar el test cuando se implemente la correccion. Sin el characterization test, la correccion podria introducir cambios laterales no detectados.

## Safety net

El conjunto de characterization tests forma una safety net: una red de seguridad que detecta cambios involuntarios. Con esa red, se puede refactorizar con confianza, sabiendo que si algo se rompe, los tests lo indicaran.

El proceso tipico es:

1. Identificar un metodo que se va a modificar.
2. Escribir characterization tests que cubran sus entradas y salidas.
3. Ejecutar los tests para verificar que pasan con el codigo actual.
4. Modificar el codigo.
5. Ejecutar los tests: si fallan, el comportamiento ha cambiado y se debe revisar.

## Cuando eliminar characterization tests

Los characterization tests no son permanentes. Su vida util termina cuando:

- El codigo original se ha refactorizado completamente y se dispone de tests de especificacion adecuados.
- Se ha corregido un bug y el test se ha actualizado para reflejar el comportamiento correcto.
- El modulo se ha reescrito y los characterization tests originales ya no tienen sentido.

En ese momento, los characterization tests se sustituyen por tests de especificacion que verifican el comportamiento deseado, no el actual.

> Los characterization tests son una herramienta transitoria. Mantenerlos indefinidamente congela el comportamiento actual e impide la mejora. El objetivo es refactorizar hasta que los characterization tests puedan eliminarse.
