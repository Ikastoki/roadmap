# Lambdas

Una expresion lambda es una funcion anonima compacta que puede ser tratada como un valor. Antes de Java 8, pasar comportamiento como argumento requeria crear clases anonimas con una sintaxis verbosa. Las lambdas reducen esa ceremonia al minimo expresivo, permitiendo escribir codigo funcional directamente en el lenguaje.

```java
// Clase anonima (antes de Java 8)
Collections.sort(lista, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareToIgnoreCase(b);
    }
});

// Lambda (Java 8+)
Collections.sort(lista, (a, b) -> a.compareToIgnoreCase(b));
```

## Sintaxis: (parametros) -> cuerpo

La sintaxis de una lambda tiene tres partes: la lista de parametros entre parentesis, el operador flecha `->`, y el cuerpo. El cuerpo puede ser una unica expresion (sin llaves ni `return` explicito) o un bloque de codigo entre llaves con `return` cuando es necesario.

```java
// Expresion simple
(int a, int b) -> a + b

// Inferencia de tipos (los tipos se omiten)
(a, b) -> a + b

// Un solo parametro, sin parentesis
s -> s.length()

// Sin parametros
() -> System.out.println("Hola")

// Cuerpo de bloque con return
(x, y) -> {
    int resultado = x * y;
    return resultado;
}
```

La inferencia de tipos es posible porque el compilador deduce los tipos de los parametros a partir del contexto, un mecanismo conocido como _target typing_.

## Target typing

Una lambda no tiene tipo por si misma; su tipo se infiere del contexto donde se usa. El compilador examina la interfaz funcional esperada en ese contexto y verifica que la firma de la lambda coincida con el metodo abstracto de dicha interfaz. Este proceso se denomina _target typing_.

```java
// El contexto determina el tipo
Predicate<String> esVacia = s -> s.isEmpty();
Function<String, Integer> longitud = s -> s.length();
Consumer<String> imprimir = s -> System.out.println(s);
```

La misma lambda `s -> s.length()` podria ser un `Function<String, Integer>` o un `ToIntFunction<String>` dependiendo del tipo destino. El compilador resuelve la ambiguedad segun la asignacion o el argumento del metodo.

```java
// Ambiguedad resuelta por target type
ToIntFunction<String> f1 = s -> s.length();
Function<String, Integer> f2 = s -> s.length();
```

> El target typing tambien funciona con metodos sobrecargados. Si un metodo tiene varias versiones que aceptan distintas interfaces funcionales, el compilador elige la mas especifica que coincida con la lambda. En caso de ambiguedad irresoluble, el codigo no compila y debe usarse un cast explicito.

## Closures y variables effectively final

Una lambda puede acceder a variables locales del ambito circundante, siempre que sean _effectively final_: no se les reasigna un valor despues de la inicializacion. Este comportamiento se conoce como _closure_, porque la lambda captura el valor de esas variables en el momento de su creacion.

```java
int factor = 2;
List<Integer> duplicados = numeros.stream()
    .map(n -> n * factor)
    .collect(Collectors.toList());
```

Si se intenta modificar `factor` dentro o fuera de la lambda, el compilador lo rechaza:

```java
int factor = 2;
// factor = 3; // Error de compilacion si se descomenta
List<Integer> duplicados = numeros.stream()
    .map(n -> n * factor)
    .collect(Collectors.toList());
```

La restriccion de _effectively final_ elimina los problemas de race conditions que surgirian si varias lambdas compartieran una misma variable mutable. Cada lambda captura una copia inmutable del valor, garantizando seguridad en entornos concurrentes.

```java
// Variables de instancia y estaticas NO necesitan ser effectively final
private int contador = 0;

public void procesar() {
    list.forEach(x -> {
        contador++; // Esto compila, pero NO es seguro en concurrencia
    });
}
```

> Las variables de instancia y estaticas no estan sujetas a la restriccion _effectively final_ porque se almacenan en el heap y son compartidas. Sin embargo, modificarlas desde una lambda no es seguro en hilos y va contra el espiritu funcional.

## Lambdas vs clases anonimas

Aunque las lambdas y las clases anonimas pueden usarse en contextos similares, existen diferencias fundamentales:

Las lambdas solo pueden implementar interfaces funcionales (un unico metodo abstracto). Las clases anonimas pueden extender cualquier clase o implementar cualquier interfaz, independientemente del numero de metodos.

```java
// Clase anonima con varios metodos
ActionListener listener = new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Accion");
    }
    // Puede tener metodos adicionales
};
```

El ambito `this` es otra diferencia critica. En una clase anonima, `this` se refiere a la instancia de la propia clase anonima. En una lambda, `this` se refiere a la instancia de la clase circundante:

```java
public class Ejemplo {
    private String nombre = "Clase";

    public void test() {
        Runnable anonima = new Runnable() {
            private String nombre = "Anonima";
            @Override
            public void run() {
                System.out.println(this.nombre); // "Anonima"
            }
        };

        Runnable lambda = () -> {
            System.out.println(this.nombre); // "Clase"
        };
    }
}
```

Las lambdas tampoco introducen un nuevo nivel de ambito; las variables declaradas en el bloque circundante son accesibles directamente sin cualificacion. Esto hace que las lambdas sean mas ligeras semanticamente: son funciones, no objetos con identidad propia.
