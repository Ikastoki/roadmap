# Palabras reservadas

Java define un conjunto fijo de palabras reservadas (keywords) que el compilador reconoce como parte integral de la sintaxis del lenguaje. Estas palabras tienen un significado predefinido y no pueden utilizarse como identificadores (nombres de variables, metodos, clases o paquetes) en ningun contexto.

## Lista completa

Las palabras reservadas de Java se organizan funcionalmente en varias categorias:

### Definicion de tipos

```java
class MiClase { }          // define una clase
interface MiInterfaz { }    // define una interfaz
enum MiEnum { A, B, C }    // define una enumeracion
record MiRegistro(int x) { } // define un record (Java 16+)
```

**class** declara una nueva clase, la unidad fundamental de modelado en Java. **interface** define un contrato que las clases pueden implementar. **enum** crea un tipo con un conjunto finito de constantes nombradas. **record** (desde Java 16) declara una clase inmutable para transportar datos, generando automaticamente constructor, getters, `equals`, `hashCode` y `toString`.

### Herencia e implementacion

```java
class Hija extends Padre { }           // herencia de clase
class Implementadora implements Interfaz { }  // implementacion de interfaz
```

**extends** establece la relacion de herencia entre clases. **implements** indica que una clase satisface el contrato de una o mas interfaces. Java permite herencia simple de clases pero implementacion multiple de interfaces.

### Creacion y referencia de objetos

```java
MiObjeto obj = new MiObjeto();  // crea una instancia
this.valor = 5;                 // referencia al objeto actual
super.constructor();            // invoca al constructor del padre
```

**new** reserva memoria en el heap y ejecuta el constructor del objeto. **this** es una referencia al objeto sobre el que se esta ejecutando el metodo, util para desambiguar campos y pasar la instancia actual. **super** accede a miembros de la superclase directa, especialmente util para invocar el constructor o metodos sobreescritos del padre.

### Modificadores

```java
public class Ejemplo {
    static int contador = 0;
    final String id;
    abstract void metodo();
    synchronized void critico() { }
    volatile boolean activo;
    transient int temporal;
    native void metodoNativo();
    strictfp double calculo() { return 0.0; }
}
```

**static** pertenece a la clase, no a instancias. **final** impide modificacion o sobreescritura. **abstract** requiere implementacion en subclases. **synchronized** controla acceso concurrente a un recurso. **volatile** garantiza visibilidad entre hilos. **transient** excluye un campo de la serializacion. **native** indica que el metodo esta implementado en codigo nativo (C/C++) a traves de JNI. **strictfp** fuerza calculos de punto flotante conformes al estandar IEEE 754, eliminando optimizaciones dependientes de la plataforma.

> Antes de Java 17, `const` era una palabra reservada no utilizada (nunca implementada). Fue eliminada del listado oficial de keywords, aunque no se recomienda usarla como identificador por compatibilidad.

### Control de flujo

```java
// Condicionales
if (condicion) { }
else { }
switch (valor) {
    case 1: break;
    default: break;
}

// Bucles
for (int i = 0; i < 10; i++) { }
while (condicion) { }
do { } while (condicion);

// Salto
break;      // sale del bucle o switch
continue;   // salta a la siguiente iteracion
return;     // devuelve valor y sale del metodo
yield;      // produce valor en un switch expresion (Java 14+)
```

**if/else** ejecuta codigo segun una condicion booleana. **switch** selecciona un bloque de codigo entre multiples casos. **for** ejecuta un bucle con inicializacion, condicion e incremento. **while** repite mientras la condicion sea verdadera. **do-while** es como while pero evaluando la condicion al final, garantizando al menos una ejecucion. **break** termina el bucle o switch mas interno. **continue** salta directamente a la siguiente iteracion. **return** devuelve un valor (o void) y termina la ejecucion del metodo. **yield** se usa dentro de expresiones switch para producir un valor sin salir del bloque.

### Excepciones

```java
try {
    // codigo que puede lanzar excepciones
} catch (Excepcion e) {
    // manejo de la excepcion
} finally {
    // siempre se ejecuta
}

throw new Excepcion("mensaje");
void metodo() throws Excepcion { }
```

**try** define el bloque de codigo protegido. **catch** captura y maneja una excepcion especifica. **finally** contiene codigo que se ejecuta siempre, con o sin excepcion. **throw** lanza una excepcion explicitamente. **declara** en la firma del metodo que puede propagar excepciones controladas.

### Literales y tipos

```java
boolean verdadero = true;
boolean falso = false;
int nulo = 0;  // null no es aplicable a primitivos
String s = null;
```

**true**, **false** y **null** son literales, no keywords tecnicamente, pero tampoco pueden usarse como identificadores. **var** (desde Java 10) permite inferencia de tipos en declaraciones locales.

### Modulos (Java 9+)

```java
module mi.modulo {
    requires java.sql;
    requires transitive java.logging;
    exports com.mi.paquete to otromodulo;
    opens com.mi.paquete;
}
```

**module** declara un modulo del Sistema de Modulos (JPMS). **requires** establece una dependencia. **requires transitive** propaga la dependencia a los consumidores del modulo. **exports** expone paquetes a otros modulos. **opens** permite reflexion sobre paquetes internos.

### Otros

**instanceof** verifica si un objeto es instancia de un tipo especifico o compatible. Desde Java 16 funciona con pattern matching: `if (obj instanceof String s)`. **assert** valida condiciones que deben ser ciertas en el codigo; si la condicion es falsa lanza `AssertionError`. Se activan con el flag `-ea` al ejecutar la JVM.

> Usar palabras reservadas como identificadores genera un error de compilacion inmediato: `error: <identifier> expected`. El compilador no permite ninguna excepcion a esta regla.
