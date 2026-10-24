# Constantes

Java proporciona la palabra reservada `final` para declarar constantes, esto es, variables cuyo valor no puede ser modificado una vez asignado. La semantica de `final` es mas rica de lo que parece a simple vista, pues su comportamiento depende del contexto en que se aplique: variables locales, campos de instancia, campos de clase, parametros de metodo, e incluso metodos y clases enteras.

Cuando una variable primitiva se declara como `final`, su valor queda fijado de forma permanente. Si se trata de una referencia a un objeto, `final` impide reasignar la referencia, pero no hace inmutable el objeto al que apunta. Este matiz es fundamental para evitar suposiciones incorrectas sobre la inmutabilidad.

## Constantes de clase (static final)

La combinacion `static final` crea constantes verdaderas a nivel de clase. Al ser `static`, el campo pertenece a la clase y no a cada instancia; al ser `final`, su valor no cambia jamas.

```java
public class ConstantesMatematicas {
    public static final double PI = 3.141592653589793;
    public static final double E  = 2.718281828459045;
    public static final int DIAS_SEMANA = 7;
}
```

Estas constantes se acceden mediante el nombre de la clase: `ConstantesMatematicas.PI`. El compilador puede incluso optimizar las constantes de tipo primitivo o `String` marcadas como `static final` reemplazando su uso directo en el codigo (compile-time constants).

## Constantes de instancia

Un campo `final` no `static` es una constante de instancia: cada objeto puede tener su propio valor, pero ese valor no puede cambiar tras la construccion.

```java
public class Usuario {
    private final long id;
    private final String fechaCreacion;

    public Usuario(long id) {
        this.id = id;
        this.fechaCreacion = LocalDate.now().toString();
    }
}
```

El valor debe asignarse en el constructor o en el mismo momento de la declaracion. No hay setters posibles. Esto es util para campos que definen la identidad del objeto.

> El compilador garantiza que todo campo `final` de instancia sea asignado exactamente una vez durante la construccion. Si existe alguna ruta de ejecucion donde el campo quede sin asignar, se produce un error de compilacion.

## Convencion UPPER_SNAKE_CASE

Las constantes en Java se nombran con mayusculas y guiones bajos separando palabras:

```java
public static final int MAX_CONEXIONES = 100;
public static final String NOMBRE_APP = "SistemaInventario";
```

Esta convencion, establecida en el code conventions de Sun Microsystems, permite identificar rapidamente que un identificador es una constante inmutable. Se aplica tanto a constantes de clase como de instancia cuando su semantica es la de un valor fijo.

## Constantes vs enums

Cuando un conjunto de constantes esta relacionado conceptualmente, la alternativa preferida desde Java 5 es el tipo `enum`:

```java
// Constantes sueltas - valido pero menos expresivo
public static final int ESTADO_ACTIVO = 1;
public static final int ESTADO_INACTIVO = 2;

// Enum - semanticamente superior
public enum Estado {
    ACTIVO, INACTIVO, PENDIENTE
}
```

Los `enum` ofrecen type-safety, metodos propios, y la posibilidad de anadir campos y comportamientos. Las constantes con `static final` siguen siendo adecuadas para valores numericos, configuraciones, o cuando no existe una relacion logica entre ellas.

> Los `enum` en Java son clases completas que extienden `java.lang.Enum` de forma implicita. No pueden instanciarse con `new` y cada constante es una instancia singleton del tipo enum.

## Compile-time constants

Una constante es considerada de compilacion (compile-time constant) cuando se declara como `static final` de tipo primitivo o `String`, y su valor es una expresion constante. El compilador reemplaza todas las referencias a dicha constante por su valor literal en el codigo compilado.

```java
public static final int TAMANO_BUFFER = 1024 * 4; // compile-time constant
public static final String NOMBRE = "Sistema";    // compile-time constant
public static final LocalDate HOY = LocalDate.now(); // NO es compile-time
```

Este comportamiento permite al compilador realizar optimizaciones como la propagacion de constantes y la eliminacion de codigo muerto. Sin embargo, introduce un riesgo: si se cambia el valor de una constante publica en una libreria y no se recompilan los modulos que la usan, estos seguiran utilizando el valor antiguo incrustado en su propio bytecode.
