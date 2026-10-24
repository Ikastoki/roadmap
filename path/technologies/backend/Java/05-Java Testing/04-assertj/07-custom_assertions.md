# Custom assertions

Aunque AssertJ cubre la mayoria de tipos del JDK y ofrece un amplio repertorio de aserciones, los objetos de dominio de una aplicacion suelen requerir verificaciones especificas que no se expresan bien con las aserciones genericas. Por ejemplo, validar que una `Persona` tiene un nombre valido o que una `Factura` esta correctamente calculada implica logica que, si se escribe con aserciones genericas, resulta verbosa y poco expresiva. Las aserciones personalizadas permiten encapsular esa logica en metodos semanticos reutilizables.

## AbstractAssert subclass

La forma de crear una asercion personalizada es extender `AbstractAssert<SELF, ACTUAL>`, donde `SELF` es el tipo del asertor (para permitir el encadenamiento fluido) y `ACTUAL` es el tipo del objeto bajo prueba. La clase debe proporcionar un constructor protegido y un metodo estatico `assertThat` como punto de entrada.

```java
import org.assertj.core.api.AbstractAssert;

public class PersonaAssert extends AbstractAssert<PersonaAssert, Persona> {

    protected PersonaAssert(Persona actual) {
        super(actual, PersonaAssert.class);
    }

    public static PersonaAssert assertThat(Persona actual) {
        return new PersonaAssert(actual);
    }

    public PersonaAssert tieneNombreValido() {
        isNotNull();
        if (actual.getNombre() == null || actual.getNombre().isEmpty()) {
            failWithMessage("Se esperaba que la persona tuviera un nombre no vacio, pero era <%s>",
                actual.getNombre());
        }
        return this;
    }

    public PersonaAssert esMayorDeEdad() {
        isNotNull();
        if (actual.getEdad() < 18) {
            failWithMessage("Se esperaba que <%s> fuera mayor de edad, pero tiene <%d> anos",
                actual.getNombre(), actual.getEdad());
        }
        return this;
    }
}
```

> El parametro `SELF` debe ser la propia subclase. Esto permite que metodos heredados como `isNotNull()`, `isEqualTo()` o `as()` retornen el tipo personalizado en lugar de `AbstractAssert`, manteniendo el encadenamiento fluido con todos los metodos del asertor original y los nuevos.

## Crear assertion personalizada

Una vez definida la clase, se usa como cualquier asertor nativo de AssertJ:

```java
Persona persona = new Persona("Ana", 25);

PersonaAssert.assertThat(persona)
    .tieneNombreValido()
    .esMayorDeEdad();
```

El metodo `failWithMessage` detiene la ejecucion y produce un `AssertionError` con el mensaje formateado. Como alternativa, puede lanzarse una excepcion manualmente:

```java
if (actual.getEdad() == null) {
    throw new AssertionError("La edad no puede ser nula");
}
```

Sin embargo, `failWithMessage` es preferible porque integra automaticamente la informacion de contexto (como el valor actual) y mantiene la coherencia con los mensajes de error de AssertJ.

## Reutilizar

Las aserciones personalizadas se reutilizan importando el metodo estatico `assertThat` en las clases de test. Para evitar conflictos con el `assertThat` nativo de AssertJ, se puede usar un alias en la importacion o mantenerlas en un paquete separado:

```java
import static com.miempresa.test.assertions.PersonaAssert.assertThat;

@Test
void testPersona() {
    Persona persona = new Persona("Luis", 30);
    assertThat(persona)
        .tieneNombreValido()
        .esMayorDeEdad()
        .hasEmail("luis@example.com");
}
```

> Para conjuntos de objetos (lista de personas), AssertJ proporciona `IterableAssert` con el metodo `as`. Las aserciones personalizadas para colecciones se crean extendiendo `AbstractIterableAssert` o usando `SoftAssertions` para verificar cada elemento individualmente con el asertor personalizado.

```java
public static <T> void assertThatAll(Iterable<T> iterable, Consumer<T> asercion) {
    iterable.forEach(item -> asercion.accept(item));
}

// Uso
assertThatAll(personas, persona -> 
    PersonaAssert.assertThat(persona).esMayorDeEdad());
```

AssertJ 3.x introdujo `assertThat` generico para colecciones, que permite aplicar un asertor personalizado a cada elemento:

```java
assertThat(personas).allSatisfy(persona -> {
    PersonaAssert.assertThat(persona).tieneNombreValido();
    PersonaAssert.assertThat(persona).esMayorDeEdad();
});
```
