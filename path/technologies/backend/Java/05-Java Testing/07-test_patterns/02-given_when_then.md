# Given-When-Then

Given-When-Then es la formulacion de AAA adoptada por BDD (Behavior-Driven Development). Donde AAA es una estructura tecnica para organizar el codigo del test, Given-When-Then es una convencion narrativa que enfatiza el comportamiento desde la perspectiva del usuario o del sistema. Ambas representan la misma estructura logica, pero con diferente lenguaje.

## Given (contexto)

Given establece el estado inicial del sistema. Describe las condiciones previas que deben cumplirse para que el escenario tenga sentido. En terminos de AAA, es la fase Arrange, pero expresada en un lenguaje que cualquier stakeholder pueda entender.

```java
@Test
void shouldWithdrawCashWhenBalanceIsSufficient() {
    // Given: una cuenta con saldo suficiente
    Account account = new Account("ES123456789", 1000.0);
    ATM atm = new ATM(account);

    // When: el usuario solicita un reintegro
    WithdrawalResult result = atm.withdraw(500.0);

    // Then: el sistema entrega el efectivo y descuenta el saldo
    assertTrue(result.isSuccess());
    assertEquals(500.0, result.getDispensedAmount());
    assertEquals(500.0, account.getBalance(), 0.001);
}
```

## When (accion)

When describe la accion que desencadena el comportamiento. Es el evento que ocurre en el sistema: el usuario pulsa un boton, se recibe una peticion HTTP, vence un plazo. En el test, es la invocacion al metodo bajo prueba.

La accion debe estar expresada en terminos de comportamiento, no de implementacion. No se dice "se llama al metodo calculate() con parametro X", sino "el usuario solicita un calculo de precio".

## Then (resultado)

Then especifica el resultado esperado de la accion. Puede ser un cambio de estado, un valor de retorno, una interaccion con un tercero o una excepcion. El resultado debe ser observable y verificable.

```java
    // Then: el sistema rechaza la operacion
    assertFalse(result.isSuccess());
    assertEquals("Saldo insuficiente", result.getErrorMessage());
}
```

## BDD style

BDD lleva Given-When-Then un paso mas alla, separando la descripcion del escenario en un lenguaje natural que puede ser entendido por tecnicos y no tecnicos. Herramientas como Cucumber o JGiven permiten escribir los escenarios en ficheros independientes y vincularlos al codigo de prueba.

```gherkin
Scenario: Retirar efectivo con saldo suficiente
  Given una cuenta con saldo de 1000 EUR
  When el usuario solicita un reintegro de 500 EUR
  Then el sistema entrega 500 EUR
  And el saldo de la cuenta es 500 EUR
```

La implementacion en Java con JGiven queda:

```java
class WithdrawalTest extends ScenarioTest<GivenSteps, WhenSteps, ThenSteps> {

    @Test
    void withdrawingWithSufficientBalance() {
        given().anAccountWithBalance(1000.0);
        when().userRequestsWithdrawal(500.0);
        then().systemDispenses(500.0);
        and().accountBalanceIs(500.0);
    }
}
```

> Given-When-Then convierte el test en una especificacion ejecutable. La misma estructura que usa el desarrollador en JUnit puede ser la base para la documentacion funcional del proyecto.

## Como especificacion

Cuando los tests se escriben en formato Given-When-Then, la suite de tests funciona como una especificacion viva del sistema. Cada escenario describe un comportamiento completo y autocontenido, y el conjunto de escenarios cubre la totalidad de los requisitos.

Para maximizar este efecto:
- Usar nombres de test que describan el escenario completo: `shouldRejectWithdrawalWhenDailyLimitExceeded`
- Mantener un escenario por comportamiento
- Expresar Given, When, Then en el lenguaje del dominio
- Evitar terminos tecnicos en la descripcion del escenario

```java
@Test
void shouldNotAllowWithdrawalWhenDailyLimitExceeded() {
    // Given: una cuenta con limite diario de 300 EUR
    Account account = new Account("ES123456789", 5000.0, dailyLimit(300.0));
    account.recordWithdrawal(250.0);

    // When: el usuario intenta retirar 100 EUR adicionales
    WithdrawalResult result = account.withdraw(100.0);

    // Then: la operacion es rechazada por exceder el limite diario
    assertFalse(result.isSuccess());
    assertEquals("Limite diario excedido", result.getErrorMessage());
}
```
