# Que es Property-Based Testing

El property-based testing (PBT) es un enfoque de testing donde, en lugar de escribir ejemplos concretos con valores especificos, se definen propiedades que deben cumplirse para cualquier entrada dentro de un dominio. La herramienta se encarga de generar automaticamente miles de combinaciones de datos y verificar que la propiedad se sostiene para todas ellas.

## Example-based vs Property-based

El testing tradicional se basa en ejemplos. Se escogen unos valores representativos, se invoca la funcion y se verifica el resultado esperado.

```java
@Test
void sumaEsConmutativa() {
    assertThat(sumar(2, 3)).isEqualTo(5);
    assertThat(sumar(3, 2)).isEqualTo(5);
}
```

Este enfoque tiene un problema fundamental: solo prueba los casos que el desarrollador imagino. Los casos extremos, las combinaciones inesperadas o los valores limite quedan fuera a menos que se escriban explicitamente.

El property-based testing invierte la logica. Se define una propiedad universal y se deja que la herramienta genere los datos.

```java
@Property
boolean sumaEsConmutativa(@ForAll int a, @ForAll int b) {
    return sumar(a, b) == sumar(b, a);
}
```

Si la propiedad falla para algun valor, la herramienta encuentra el contraejemplo mas pequeno posible (shrinking) y lo presenta para su analisis.

## Generacion de datos

Las librerias de PBT incluyen generadores para tipos nativos y permiten construir generadores personalizados. Los generadores producen valores aleatorios dentro de un rango, pero con sesgos hacia casos interesantes: cero, valores negativos, numeros grandes, cadenas vacias, nulos, etc.

```java
@Property
boolean longitudConcatenacion(@ForAll @StringLength(min=0, max=100) String a,
                               @ForAll @StringLength(min=0, max=100) String b) {
    return (a + b).length() == a.length() + b.length();
}
```

## Shrinking

Cuando se encuentra un contraejemplo, el shrinking intenta reducirlo al caso mas simple que sigue reproduciendo el fallo. Si la propiedad falla con una cadena de 500 caracteres, el algoritmo acorta progresivamente hasta hallar la cadena minima que provoca el error, facilitando la depuracion.

Por ejemplo, si la propiedad falla con el array `[1024, -1, 0, 32767]`, el shrinking podria reducirlo a `[0]` o incluso `[]`, revelando que el problema es el array vacio y no una combinacion compleja de valores.

## jqwik

jqwik es la libreria de property-based testing mas madura para Java, con integracion nativa en JUnit 5. Se basa en el mismo modelo que JUnit 5 (extensiones, anotaciones, lifecycle) y ofrece:

- Generadores para todos los tipos de Java
- Combinacion y creacion de generadores personalizados
- Shrinking automatico
- Configuracion de casos extremos (@EdgeCases)
- Estadisticas de cobertura de generacion

```java
import net.jqwik.api.*;

class PropiedadesMatematicasTest {

    @Property
    boolean identidadMultiplicativa(@ForAll int n) {
        return n * 1 == n;
    }
}
```

El PBT no reemplaza al testing tradicional, lo complementa. Las propiedades son mas dificiles de definir que los ejemplos, pero una vez escritas cubren un espectro de casos que serian imposibles de enumerar manualmente. Es especialmente potente en algoritmos, transformaciones de datos, parsing, serializacion y logica financiera.
