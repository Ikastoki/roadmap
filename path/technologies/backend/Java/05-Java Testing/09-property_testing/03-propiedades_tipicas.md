# Propiedades tipicas

El exito del property-based testing depende de la capacidad para identificar propiedades universales que el sistema debe cumplir. Existen patrones recurrentes que aparecen en casi cualquier dominio y que sirven como punto de partida para disenar propiedades.

## Orden

La propiedad de orden verifica que una operacion produce resultados consistentes respecto a una relacion de orden. Es util en algoritmos de ordenacion, colas de prioridad y estructuras ordenadas.

```java
@Property
boolean ordenacionEsEstable(@ForAll List<@IntRange(min = -100, max = 100) Integer> lista,
                            @ForAll @IntRange(min = 0, max = lista.size()) int indice) {
    List<Integer> ordenada = ordenar(lista);
    if (ordenada.size() < 2) return true;
    for (int i = 1; i < ordenada.size(); i++) {
        if (ordenada.get(i - 1) > ordenada.get(i)) return false;
    }
    return true;
}
```

## Conmutatividad

La conmutatividad afirma que el orden de los operandos no altera el resultado. Se aplica a operaciones binarias como suma, multiplicacion, union de conjuntos o concatenacion cuando el orden no importa.

```java
@Property
boolean sumaConmutativa(@ForAll int a, @ForAll int b) {
    return sumar(a, b) == sumar(b, a);
}
```

No toda operacion debe ser conmutativa. La resta, la division o la composicion de funciones no lo son. La propiedad solo tiene sentido cuando el dominio asi lo exige.

## Idempotencia

Una operacion es idempotente si aplicarla una vez o multiples veces produce el mismo resultado. Es una propiedad clave en sistemas distribuidos, APIs REST (PUT, DELETE), deduplicacion de datos y operaciones de saneamiento.

```java
@Property
boolean sanitizarEsIdempotente(@ForAll String texto) {
    String unaVez = sanitizar(texto);
    String dosVeces = sanitizar(sanitizar(texto));
    return unaVez.equals(dosVeces);
}
```

## Reversibilidad

La reversibilidad establece que una operacion tiene una inversa que restaura el estado original. Es tipica de operaciones de serializacion/deserializacion, compresion, cifrado y transformaciones bidireccionales.

```java
@Property
boolean serializacionCircular(@ForAll Usuario usuario) {
    String json = serializar(usuario);
    Usuario deserializado = deserializar(json);
    return usuario.equals(deserializado);
}
```

Esta propiedad es extraordinariamente potente porque verifica todo el ciclo de transformacion sin necesidad de conocer los valores intermedios.

## Invariante

Un invariante es una condicion que se mantiene verdadera antes y despues de cualquier operacion. Es la propiedad mas general y la mas util en sistemas con estado.

```java
@Property
boolean saldoNuncaNegativo(@ForAll @IntRange(min = 0, max = 10000) int saldoInicial,
                           @ForAll @IntRange(min = 0, max = saldoInicial) int retiro) {
    Cuenta cuenta = new Cuenta(saldoInicial);
    cuenta.retirar(retiro);
    return cuenta.getSaldo() >= 0;
}
```

Los invariantes son especialmente utiles para modelar reglas de negocio: un pedido nunca puede tener total negativo, un usuario no puede tener dos sesiones activas con el mismo token, una transaccion no puede tener fecha futura.

## Otras propiedades recurrentes

- **Monotonia**: si `a > b` entonces `f(a) > f(b)` (funciones estrictamente crecientes).
- **Asociatividad**: `f(a, f(b, c)) == f(f(a, b), c)`.
- **Distributividad**: `f(a, g(b, c)) == g(f(a, b), f(a, c))`.
- **Acotacion**: el resultado siempre esta dentro de un rango conocido.
- **Preservacion de estructura**: una transformacion conserva propiedades como longitud, cardinalidad o tipo.

Identificar propiedades requiere un cambio de mentalidad respecto al testing tradicional. No se piensa en "que valor deberia devolver" sino en "que debe ser siempre cierto". Una vez que el equipo desarrolla este enfoque, las propiedades revelan comportamientos inesperados que los tests tradicionales jamas detectarian.
