# Custom matchers

Aunque Hamcrest proporciona decenas de matchers predefinidos, en aplicaciones reales surgen con frecuencia condiciones de validacion especificas del dominio que no se expresan adecuadamente con los matchers existentes. Crear matchers personalizados permite encapsular esas condiciones en objetos reutilizables, componibles y con mensajes de error descriptivos. Hamcrest ofrece varias clases base para esta tarea, cada una con un nivel distinto de control y complejidad.

## TypeSafeMatcher

`TypeSafeMatcher<T>` es la clase base recomendada para la mayoria de matchers personalizados. Hereda de `BaseMatcher` y anade comprobacion de tipos segura en tiempo de ejecucion, evitando `ClassCastException`. Solo requiere implementar dos metodos: `matchesSafely(T item)` y `describeTo(Description description)`.

```java
import org.hamcrest.Description;
import org.hamcrest.TypeSafeMatcher;

public class EsDivisiblePor extends TypeSafeMatcher<Integer> {

    private final int divisor;

    public EsDivisiblePor(int divisor) {
        this.divisor = divisor;
    }

    @Override
    protected boolean matchesSafely(Integer valor) {
        return valor % divisor == 0;
    }

    @Override
    public void describeTo(Description description) {
        description.appendText("un numero divisible por ")
                   .appendValue(divisor);
    }

    public static EsDivisiblePor esDivisiblePor(int divisor) {
        return new EsDivisiblePor(divisor);
    }
}
```

El metodo `matchesSafely` recibe el valor ya casteado al tipo parametrizado, por lo que no necesita verificaciones de tipo manuales. `describeTo` construye la descripcion del matcher para los mensajes de error.

```java
assertThat(10, esDivisiblePor(2));
assertThat(7, not(esDivisiblePor(2)));
```

## TypeSafeDiagnosingMatcher

`TypeSafeDiagnosingMatcher<T>` extiende `TypeSafeMatcher` anadiendo la capacidad de describir por que fallo una coincidencia. En lugar de `matchesSafely`, implementa `matchesSafely(T item, Description mismatchDescription)`, donde se puede escribir una descripcion especifica del fallo.

```java
public class EstaEnRango extends TypeSafeDiagnosingMatcher<Integer> {

    private final int min;
    private final int max;

    public EstaEnRango(int min, int max) {
        this.min = min;
        this.max = max;
    }

    @Override
    protected boolean matchesSafely(Integer valor, Description mismatchDescription) {
        if (valor < min) {
            mismatchDescription.appendText(valor + " es menor que " + min);
            return false;
        }
        if (valor > max) {
            mismatchDescription.appendText(valor + " es mayor que " + max);
            return false;
        }
        return true;
    }

    @Override
    public void describeTo(Description description) {
        description.appendText("un numero entre ")
                   .appendValue(min).appendText(" y ").appendValue(max);
    }

    public static EstaEnRango estaEnRango(int min, int max) {
        return new EstaEnRango(min, max);
    }
}
```

```java
assertThat(15, estaEnRango(10, 20));
assertThat(25, estaEnRango(10, 20)); // falla: "25 es mayor que 20"
```

> La diferencia entre `TypeSafeMatcher` y `TypeSafeDiagnosingMatcher` es sutil pero importante. `TypeSafeMatcher` solo puede describir lo que esperaba, no por que fallo. `TypeSafeDiagnosingMatcher` permite describir exactamente que salio mal, lo que resulta en mensajes de error mucho mas utiles para condiciones complejas.

## BaseMatcher

`BaseMatcher<T>` es la clase base de mas bajo nivel. Implementa la interfaz `Matcher<T>` directamente y requiere implementar tres metodos: `matches(Object item)`, `describeTo(Description description)` y opcionalmente `describeMismatch(Object item, Description mismatchDescription)`. No realiza comprobacion de tipos automatica, por lo que el desarrollador debe verificar el tipo manualmente.

```java
import org.hamcrest.BaseMatcher;
import org.hamcrest.Description;
import org.hamcrest.Matcher;

public class EsPalindromo extends BaseMatcher<String> {

    @Override
    public boolean matches(Object item) {
        if (!(item instanceof String)) {
            return false;
        }
        String s = (String) item;
        return new StringBuilder(s).reverse().toString().equals(s);
    }

    @Override
    public void describeTo(Description description) {
        description.appendText("una cadena palindroma");
    }

    public static Matcher<String> esPalindromo() {
        return new EsPalindromo();
    }
}
```

```java
assertThat("reconocer", esPalindromo());
assertThat("casa", not(esPalindromo()));
```

> En general, `TypeSafeMatcher` es preferible a `BaseMatcher` porque la comprobacion de tipos automatica evita errores rutinarios y reduce el codigo boilerplate. `TypeSafeDiagnosingMatcher` se recomienda cuando se necesita un control preciso sobre el mensaje de fallo. `BaseMatcher` solo se utiliza cuando se requiere compatibilidad con tipos no genericos o cuando se implementan matchers para tipos cuyo `Class` no esta disponible en tiempo de compilacion.

Todos los matchers personalizados deben proporcionar un metodo estatico fabrica que facilite su importacion estatica y uso en las aserciones:

```java
import static com.miempresa.test.matchers.EsDivisiblePor.esDivisiblePor;
import static com.miempresa.test.matchers.EstaEnRango.estaEnRango;

assertThat(100, esDivisiblePor(10));
assertThat(50, estaEnRango(1, 100));
```
