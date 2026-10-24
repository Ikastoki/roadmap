# Red-Green-Refactor

El ciclo Red-Green-Refactor es el ritmo fundamental del Desarrollo Guiado por Pruebas (TDD). Este patron de tres pasos convierte el testing en una herramienta de diseno de software, no solo de verificacion. La idea central es que el test se escribe antes que el codigo de produccion, y que el codigo de produccion se escribe unicamente para hacer pasar ese test.

## Fase roja: escribir un test que falle

El ciclo comienza escribiendo un test para una funcionalidad que aun no existe. Como el codigo de produccion no implementa esa funcionalidad, el test debe fallar (de ahi el nombre "rojo", por el color tipico de los fallos en los runners de testing). Este fallo es importante: confirma que el test esta correctamente escrito y que la funcionalidad realmente no existe. Si el test pasara en esta fase, indicaria que el test es defectuoso o que la funcionalidad ya esta implementada sin que el desarrollador lo supiera. El objetivo de esta fase es definir el comportamiento esperado desde la perspectiva del consumidor del codigo, lo que fuerza a pensar en la interfaz antes que en la implementacion.

> Escribir el test antes que el codigo obliga a disenar una API limpia y minimalista. Si el test es dificil de escribir, la API probablemente sea compleja. Esta retroalimentacion temprana es una de las mayores contribuciones del TDD a la calidad del diseno.

## Fase verde: hacer pasar el test

Una vez que el test falla, se escribe el minimo codigo de produccion necesario para que pase. En esta fase no se busca elegancia, rendimiento ni completitud. La unica meta es lograr que el test se vuelva verde. Esto puede implicar devolver un valor literal, escribir una condicion rudimentaria o incluso duplicar codigo. La razon de esta disciplina es mantener el enfoque en el comportamiento requerido: si se escribe mas codigo del necesario, se introduce el riesgo de comportamientos no especificados y no testeados. El codigo optimo, refactorizado y completo llegara en la siguiente fase.

## Fase azul: refactorizar

Con el test en verde, el codigo de produccion se refactoriza para eliminar duplicacion, mejorar la estructura y aplicar principios de diseno (SOLID, patrones, nombres expresivos). La red de seguridad del test garantiza que la refactorizacion no altera el comportamiento esperado. En esta fase se permite cualquier mejora siempre que todos los tests sigan en verde. La refactorizacion puede incluir extraer metodos, renombrar variables, cambiar la estructura de clases o introducir abstracciones. El ciclo se repite para cada nueva funcionalidad o caso borde.

```java
// Fase Roja: escribir test
@Test
void deberiaCalcularDescuentoParaClienteVip() {
    var precio = new Precio(100.0);
    var descuento = precio.calcularDescuento(TipoCliente.VIP);
    assertEquals(20.0, descuento, 0.001);
}

// Fase Verde: implementacion minima
public double calcularDescuento(TipoCliente tipo) {
    if (tipo == TipoCliente.VIP) return 20.0;
    return 0.0;
}

// Fase Azul: refactorizar
public double calcularDescuento(TipoCliente tipo) {
    return tipo.getFactorDescuento() * precioBase;
}
```

## Boy Scout Rule

La regla del boy scout ("deja el campamento mas limpio de como lo encontraste") aplicada al testing significa que cada vez que se trabaja sobre un fragmento de codigo, se debe dejar en mejor estado del que estaba. Esto incluye renombrar tests con nombres confusos, extraer metodos duplicados en helpers de testing, mejorar la legibilidad de las aserciones o anadir tests para casos borde que se identifiquen durante la refactorizacion. Esta practica evita la degradacion gradual de la calidad tanto del codigo de produccion como de los tests.

## Triangulacion de testing

La triangulacion es una tecnica dentro del TDD que consiste en anadir tests progresivamente mas especificos para forzar que la implementacion pase de ser generica a ser correcta. Si la primera implementacion devuelve un valor literal, el segundo test obliga a generalizar. El tercer test puede anadir una condicion adicional, y asi sucesivamente. La triangulacion evita la tentacion de escribir implementaciones complejas antes de tiempo y garantiza que cada linea de codigo de produccion responde a un test concreto que la justifica.

```java
// Test 1: fuerza implementacion basica
assertEquals("I", romanos.convertir(1));

// Test 2: fuerza generalizacion (no puede devolver siempre "I")
assertEquals("II", romanos.convertir(2));

// Test 3: fuerza logica de resta
assertEquals("IV", romanos.convertir(4));
```
