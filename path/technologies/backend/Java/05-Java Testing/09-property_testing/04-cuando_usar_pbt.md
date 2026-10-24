# Cuando usar PBT

El property-based testing no es una alternativa al testing tradicional, sino un complemento que brilla en escenarios concretos. Aplicarlo donde no corresponde anade complejidad sin beneficio. Identificar las situaciones adecuadas es clave para obtener valor real.

## Algoritmos complejos

Los algoritmos con multiples caminos logicos, condiciones anidadas o rangos de entrada amplios se benefician enormemente del PBT. Un algoritmo de ordenacion, una busqueda en arbol o una validacion de formato (email, DNI, IBAN) tienen propiedades claras que pueden verificarse con cientos de entradas aleatorias.

```java
@Property
boolean validacionEmail(@ForAll String email) {
    boolean valido = EmailValidator.esValido(email);
    if (valido) {
        return email.contains("@") && email.contains(".");
    }
    return true; // emails invalidos pueden no cumplir nada
}
```

## Funciones puras

Las funciones puras son el escenario ideal para PBT: sin efectos secundarios, sin estado mutable, sin dependencias externas. La propiedad se puede verificar de forma aislada y los generadores pueden producir entradas arbitrarias sin preocuparse por el contexto.

Cualquier funcion que reciba parametros y devuelva un resultado sin alterar el estado global es candidata. Transformaciones de cadenas, calculos matematicos, conversion de tipos, mapeo de datos.

## Transformaciones de datos

Cuando los datos fluyen por un pipeline de transformacion (lectura, parsing, validacion, enriquecimiento, persistencia), las propiedades de idempotencia, reversibilidad e invariante son naturales.

```java
@Property
boolean transformacionConservaValor(@ForAll Pedido pedido) {
    PedidoDTO dto = transformar(pedido);
    Pedido reconstruido = revertir(dto);
    return pedido.equals(reconstruido);
}
```

## Serializacion

La serializacion es quizas el caso de uso mas comun del PBT. La propiedad de round-trip (serializar y deserializar) verifica automaticamente que todas las instancias de un tipo se pueden convertir a su representacion externa y volver sin perdida.

Cubre automaticamente campos nulos, listas vacias, valores extremos, tipos anidados y cualquier combinacion que al desarrollador se le habria escapado.

## Complemento a unit testing

El PBT no reemplaza los tests unitarios tradicionales. Los tests con ejemplos siguen siendo necesarios para:

- Verificar comportamientos esperados conocidos.
- Documentar el uso esperado de la API.
- Cubrir casos que las propiedades no capturan.
- Validar reglas de negocio especificas sin una propiedad general clara.

La estrategia recomendada es escribir tests tradicionales para los casos nominales y los requisitos de negocio, y anadir propiedades para las invariantes, las operaciones sobre rangos y las transformaciones.

```java
// Test tradicional para casos conocidos
@Test
void descuentoSegunCategoria() {
    assertThat(calcDescuento(100, "VIP")).isEqualTo(20);
}

// Propiedad para invariante general
@Property
boolean descuentoNuncaSuperaTotal(@ForAll @IntRange(min=0, max=10000) int total,
                                  @ForAll @StringLength(min=1, max=10) String cat) {
    return calcDescuento(total, cat) <= total;
}
```

El PBT es especialmente valioso en codigo de infraestructura, APIs publicas, librerias y cualquier componente donde los errores en casos extremos tienen consecuencias graves. En aplicaciones de gestion con dominios simples y flujos lineales, el testing tradicional es suficiente y mas rapido de escribir.
