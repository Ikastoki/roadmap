# Maintenance de tests

Los tests son codigo de produccion. Merecen el mismo nivel de atencion, revision y refactoring que el codigo que verifican. Descuidar el mantenimiento de los tests convierte la suite en una carga en lugar de un activo.

## Tests como codigo de produccion

Un test mal escrito tiene consecuencias similares a las de un bug en produccion: consume tiempo de otros desarrolladores, genera desconfianza y ralentiza el desarrollo.

- Los tests deben seguir las mismas convenciones de estilo que el codigo de produccion.
- Las variables, metodos y clases de test deben nombrarse con el mismo cuidado.
- La complejidad ciclomatica de un test debe ser minima.
- El codigo repetido en tests debe extraerse a metodos auxiliares o fixtures.

```java
// Codigo repetido en varios tests
private Pedido crearPedidoValido() {
    Usuario u = new Usuario("user@mail.com");
    Producto p = new Producto("Laptop", 1200.0);
    return new Pedido(u, List.of(p));
}

@Test
void procesaPedidoValido() {
    Pedido pedido = crearPedidoValido();
    assertThat(servicio.procesar(pedido).isExitoso()).isTrue();
}
```

## Refactoring de tests

Cuando el codigo de produccion cambia, los tests pueden necesitar actualizarse. El refactoring de tests sigue los mismos principios que el refactoring de codigo:

- Cambios pequenos y graduales.
- Un unico test se modifica a la vez.
- Despues de cada cambio, la suite debe seguir pasando.

La diferencia es que al refactorizar tests no hay otra red de seguridad: si el test deja de compilar o falla, el error debe corregirse inmediatamente.

## Eliminar tests obsoletos

Un test obsoleto es peor que no tener test: da una falsa sensacion de cobertura y consume tiempo de ejecucion. Un test se vuelve obsoleto cuando:

- La funcionalidad que verificaba ya no existe.
- La funcionalidad cambio radicalmente y el test ya no es relevante.
- El test siempre pasa porque las aserciones son debiles o no existen.
- El test esta comentado o ignorado permanentemente.

```java
// Test obsoleto: la funcionalidad ya no existe
@Test @Disabled("Funcionalidad eliminada en v2.0")
void metodoEliminado() { ... }
```

Los tests obsoletos deben eliminarse, no conservarse "por si acaso". El control de versiones los conserva si se necesitan en el futuro.

## Tests como deuda tecnica

Los tests mal escritos, fragiles o lentos son deuda tecnica. Acumular esta deuda tiene costes:

- Tiempo de ejecucion creciente que ralentiza el feedback.
- Tests que fallan sin motivo que generan desconfianza.
- Cobertura que no protege porque los tests son debiles.
- Desarrolladores que evitan ejecutar la suite completa.

La deuda tecnica en tests se gestiona igual que en el codigo de produccion: se identifica, se prioriza y se paga en sprints dedicados o como parte del trabajo regular.

> Un equipo que mantiene sus tests como codigo de primera clase tiene una suite que inspira confianza. Un equipo que descuida sus tests termina con una suite que nadie entiende, nadie ejecuta y nadie se atreve a tocar.
