# Test First vs Test After

La decision de escribir los tests antes o despues del codigo de produccion tiene implicaciones en el diseno, la cobertura y la dinamica del equipo. Ambas estrategias tienen cabida segun el contexto.

## TDD (Test-Driven Development)

TDD propone escribir el test antes que el codigo de produccion, siguiendo el ciclo Red-Green-Refactor:

1. **Red**: escribir un test que falle porque la funcionalidad no existe.
2. **Green**: escribir el minimo codigo necesario para que el test pase.
3. **Refactor**: mejorar el codigo manteniendo los tests verdes.

```java
// Paso 1: Red (el test falla porque la clase no existe)
@Test
void calculaDescuentoVip() {
    Descuento descuento = new Descuento();
    double resultado = descuento.calcular(100.0, "VIP");
    assertThat(resultado).isEqualTo(20.0);
}

// Paso 2: Green (codigo minimo para que el test pase)
class Descuento {
    double calcular(double total, String tipo) {
        if ("VIP".equals(tipo)) return total * 0.20;
        return 0;
    }
}
```

TDD fuerza a pensar en el comportamiento desde la perspectiva del consumidor, lo que suele conducir a disenos mas desacoplados y con mejor encapsulamiento.

## Agregar tests despues

Escribir tests despues del codigo de produccion es la practica mas comun, especialmente en codigo legacy o cuando los requisitos no estan claros desde el inicio. Tiene ventajas practicas:

- Se conoce la implementacion, por lo que los tests pueden cubrir los casos reales.
- No se invierte tiempo en tests para funcionalidades que pueden cambiar o eliminarse.
- Es mas facil prototipar y explorar sin la restriccion de mantener tests verdes.

Sin embargo, el testing posterior tiende a producir tests mas acoplados a la implementacion y a dejar lagunas de cobertura en los casos que el desarrollador no considero.

## Ventajas y desventajas

| Aspecto              | Test First (TDD)                  | Test After                        |
|----------------------|-----------------------------------|-----------------------------------|
| Diseno               | Favorece acoplamiento bajo        | Puede acoplar a implementacion    |
| Cobertura            | Alta, por construccion            | Variable, depende de disciplina   |
| Velocidad inicial    | Menor (escribir test antes)       | Mayor (codigo primero)            |
| Refactoring          | Seguro (tests desde el inicio)    | Riesgoso si no hay tests          |
| Curva de aprendizaje | Requiere practica                 | Natural para la mayoria           |
| Documentacion        | Tests como especificacion viva    | Tests como verificacion tardia    |

## Cuando usar cada uno

**Test First es preferible cuando:**

- El comportamiento esta claramente definido.
- Se construyen APIs o librerias que otros consumiran.
- El diseno del codigo es una prioridad (desacoplamiento, testabilidad).
- Se trabaja en modulos criticos donde los errores son costosos.

**Test After es aceptable cuando:**

- Se explora un dominio nuevo o los requisitos son inciertos.
- Se trabaja con codigo legacy que no tiene tests.
- El prototipado rapido es necesario.
- Los tests se escriben inmediatamente despues del codigo, no al final del sprint.

La decision no es binaria. Muchos equipos combinan ambas estrategias: TDD para la logica de negocio central y test after para UI, integraciones o codigo exploratorio.

> La posicion dogmatica "siempre TDD" o "primero el codigo, los tests nunca" son igualmente perjudiciales. La madurez de un equipo se mide por su capacidad para elegir la estrategia adecuada en cada contexto.
