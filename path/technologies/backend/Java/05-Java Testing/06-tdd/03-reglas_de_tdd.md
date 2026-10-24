# Reglas de TDD

TDD no es un conjunto de buenas practicas difusas, sino una disciplina con reglas estrictas. Romper cualquiera de ellas invalida el ciclo y elimina los beneficios que la practica proporciona. Estas reglas no son sugerencias; son la definicion misma de TDD.

## Nunca cambiar codigo de produccion sin un test que falle

Esta es la regla fundamental. Todo codigo de produccion debe estar justificado por un test que haya fallado previamente. Si se introduce codigo sin un test que lo exija, se esta violando el principio de diseno emergente.

Las consecuencias de ignorar esta regla:
- Se anade funcionalidad no solicitada
- El codigo queda sin cobertura de tests
- El diseno se sobreingenieriza
- Se pierde la red de seguridad

```java
// INCORRECTO: anadir codigo sin test fallido
public class OrderService {
    // Este metodo no esta cubierto por ningun test
    public void applyCoupon(String code) {
        // logica de cupon sin test previo
    }
}

// CORRECTO: el test falla primero, luego se implementa
@Test
void shouldApplyCouponWhenCodeIsValid() {
    OrderService service = new OrderService();
    assertDoesNotThrow(() -> service.applyCoupon("SAVE10"));
}

// Ahora si se puede implementar applyCoupon
```

## Test falla -> codigo mal

Si un test falla despues de haber implementado el codigo, la interpretacion no es que el test este mal, sino que el codigo de produccion no cumple con el comportamiento especificado. La tentacion de modificar el test para que se ajuste a la implementacion actual es un antipatron conocido como "testing the test" o "greenwashing".

La unica razon legitima para modificar un test que falla es que el test este mal escrito (por ejemplo, el valor esperado es incorrecto). Pero incluso en ese caso, la secuencia correcta es:
1. Identificar si el fallo esta en el test o en el codigo
2. Si el test es correcto, arreglar el codigo
3. Si el test es incorrecto, corregir el test y asegurarse de que ahora falle por la razon correcta

## Test pasa -> refactorizar

Cuando un test pasa, el ciclo exige refactorizar antes de escribir el siguiente test. Esta pausa para mejorar la calidad del codigo es lo que diferencia TDD de simplemente "escribir tests primero".

La fase de refactoring incluye:
- Eliminar codigo duplicado
- Mejorar nombres de variables, metodos y clases
- Extraer metodos o clases cuando una responsabilidad no esta clara
- Simplificar expresiones condicionales
- Mejorar la legibilidad general

> Si no refactorizas cuando el test pasa, estas acumulando deuda tecnica a la misma velocidad que codigo.

## El test minimo que falle

Cada test debe ser la minima expresion posible del comportamiento deseado. Esto significa:
- Probar un solo escenario por test
- Usar los valores mas simples que demuestren el comportamiento
- No anadir aserciones innecesarias
- No probar casos de esquina en el mismo test que prueba el caso feliz

```java
// INCORRECTO: multiples escenarios en un solo test
@Test
void testMultipleScenarios() {
    Calculator calc = new Calculator();
    assertEquals(4, calc.add(2, 2));
    assertEquals(0, calc.add(-1, 1));
    assertEquals(-3, calc.add(-1, -2));
}

// CORRECTO: un escenario por test
@Test
void shouldAddTwoPositiveNumbers() {
    Calculator calc = new Calculator();
    assertEquals(4, calc.add(2, 2));
}

@Test
void shouldAddPositiveAndNegativeNumber() {
    Calculator calc = new Calculator();
    assertEquals(0, calc.add(-1, 1));
}

@Test
void shouldAddTwoNegativeNumbers() {
    Calculator calc = new Calculator();
    assertEquals(-3, calc.add(-1, -2));
}
```

## No escribir codigo de produccion nuevo si todos los tests pasan

Si todos los tests actuales pasan, no hay razon para anadir codigo de produccion. Cualquier codigo anadido en este estado no estara cubierto por tests y no respondera a un comportamiento requerido.

> El unico permiso para escribir codigo de produccion es la existencia de un test que falle.

## Mantener los tests verdes todo el tiempo

Salvo durante el breve instante en que se escribe un test nuevo (que debe fallar), el conjunto de tests debe estar siempre en verde. Trabajar con tests en rojo durante periodos prolongados es señal de que el ciclo se ha roto o de que los tests son inestables.
