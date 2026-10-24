# Anti-patrones de testing

Los anti-patrones son practicas que parecen testing pero que, en realidad, degradan la calidad de la suite y generan una falsa sensacion de seguridad. Identificarlos es el primer paso para evitarlos.

## Test de implementacion

Un test que verifica detalles internos de la implementacion en lugar del comportamiento observable. Estos tests se rompen al refactorizar aunque el comportamiento externo sea correcto.

```java
// Anti-patron: verificar estado interno
@Test
void descuentoTest() {
    Descuento desc = new Descuento();
    desc.aplicar(100);
    assertThat(desc.getValorInterno()).isEqualTo(15); // acoplado a implementacion
}

// Correcto: verificar comportamiento
@Test
void descuentoTest() {
    Descuento desc = new Descuento();
    double resultado = desc.aplicar(100);
    assertThat(resultado).isEqualTo(85); // precio final
}
```

## Logica en tests

Los tests no deben contener logica condicional, bucles, calculos complejos o manejo de excepciones. Si un test necesita un `if`, un `for` o un `try-catch`, probablemente esta haciendo demasiado.

```java
// Anti-patron: logica en el test
@Test
void filtrarTest() {
    List<Integer> resultados = servicio.filtrar(List.of(1, 2, 3, 4, 5));
    for (Integer r : resultados) {      // logica en el test
        assertThat(r).isGreaterThan(0); // asercion debil
    }
}

// Correcto: asercion declarativa
@Test
void filtrarTest() {
    assertThat(servicio.filtrar(List.of(1, 2, 3, 4, 5)))
        .containsExactly(1, 2, 3, 4, 5);
}
```

## Dependencia entre tests

Un test depende del resultado o del estado dejado por otro test. Esto produce fallos intermitentes que dependen del orden de ejecucion.

```java
// Anti-patron: estado compartido
private static int contador = 0;

@Test void t1() { contador++; }
@Test void t2() { assertThat(contador).isEqualTo(1); } // depende de t1
```

## Estado global

Usar variables estaticas, singletons o contextos globales que persisten entre tests. El problema es que un test puede contaminar el estado para los siguientes.

```java
// Anti-patron: singleton global
@Test
void configurar() {
    ConfiguracionGlobal.setModo("TEST"); // afecta a otros tests
}
```

## Orden

Confiar en que los tests se ejecuten en un orden especifico. JUnit 5 no garantiza el orden a menos que se configure explicitamente con `@TestMethodOrder`, y aun asi es una mala practica depender de el.

## Test que no fallan

Un test que nunca falla es sospechoso. Puede deberse a:

- Aserciones debiles o inexistentes.
- Excepciones capturadas silenciosamente.
- Datos de prueba que siempre producen el mismo resultado.

```java
// Anti-patron: asercion que nunca falla
@Test
void test() {
    try {
        servicio.ejecutar();
    } catch (Exception e) {
        // nunca se verifica nada
    }
}
```

## Test con nombre generico

Nombres como `test1()`, `testServicio()`, `comprobar()` no documentan el comportamiento. Cuando fallan, obligan a leer el cuerpo del test para entender que se esta verificando.

## Test comentado

Un test comentado es un test que ya no se ejecuta. Si un test ya no es valido, se elimina, no se comenta. El codigo comentado es ruido que engana sobre el estado real de la suite.

Los anti-patrones se previenen con code review, con convenciones claras y con un compromiso del equipo de tratar los tests como codigo de primera clase, no como un requisito molesto que hay que cumplir.
