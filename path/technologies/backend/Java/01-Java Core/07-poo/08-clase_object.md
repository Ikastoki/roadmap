# Clase Object

La clase `java.lang.Object` es la raiz de la jerarquia de clases en Java. Toda clase, directa o indirectamente, hereda de `Object`. Esto significa que todos los objetos comparten un conjunto basico de metodos que definen comportamiento fundamental: identificacion, igualdad, representacion como cadena, sincronizacion y clonacion.

## toString()

El metodo `toString()` devuelve una representacion en forma de cadena del objeto. La implementacion por defecto en `Object` devuelve el nombre de la clase seguido de la direccion de memoria en hexadecimal.

```java
Object obj = new Object();
System.out.println(obj.toString());
// Salida tipica: java.lang.Object@1a2b3c4
```

La mayoria de las clases sobrescriben `toString()` para proporcionar informacion significativa:

```java
public class Usuario {
    private String nombre;
    private String email;

    @Override
    public String toString() {
        return "Usuario{nombre='" + nombre + "', email='" + email + "'}";
    }
}

Usuario u = new Usuario("Ana", "ana@mail.com");
System.out.println(u); // Usuario{nombre='Ana', email='ana@mail.com'}
```

El metodo `toString()` se invoca implicitamente cuando se concatena un objeto con una cadena (`"Valor: " + objeto`) o cuando se pasa a `System.out.println()`.

## equals()

El metodo `equals()` compara dos objetos para determinar si son logicamente iguales. La implementacion por defecto en `Object` usa `==`, que compara referencias: dos objetos son iguales solo si son el mismo objeto en memoria.

```java
Usuario u1 = new Usuario("Ana", "ana@mail.com");
Usuario u2 = new Usuario("Ana", "ana@mail.com");

System.out.println(u1 == u2);         // false (distintas referencias)
System.out.println(u1.equals(u2));    // false (usa == por defecto)
```

Para que `equals()` compare contenido en lugar de referencias, debe sobrescribirse siguiendo el contrato establecido por la documentacion de `Object`:

1. **Reflexivo:** `x.equals(x)` debe ser `true`
2. **Simetrico:** `x.equals(y)` implica `y.equals(x)`
3. **Transitivo:** si `x.equals(y)` e `y.equals(z)`, entonces `x.equals(z)`
4. **Consistente:** multiples invocaciones deben dar el mismo resultado
5. **No-null:** `x.equals(null)` debe ser `false`

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;

    Usuario usuario = (Usuario) o;
    return Objects.equals(nombre, usuario.nombre)
        && Objects.equals(email, usuario.email);
}
```

## hashCode()

Cada objeto tiene un codigo hash (entero de 32 bits) utilizado por las colecciones basadas en hash (`HashMap`, `HashSet`, `HashTable`). El contrato entre `equals` y `hashCode` establece:

- Si dos objetos son iguales segun `equals()`, deben tener el mismo `hashCode()`
- Si dos objetos tienen el mismo `hashCode()`, pueden ser iguales o no (colision)
- Si `equals()` se sobrescribe, `hashCode()` debe sobrescribirse siempre

```java
@Override
public int hashCode() {
    return Objects.hash(nombre, email);
}
```

> La violacion de este contrato es uno de los errores mas comunes en Java. Si una clase sobrescribe `equals()` sin sobrescribir `hashCode()`, las colecciones basadas en hash se comportan incorrectamente: dos objetos iguales pueden terminar en buckets distintos.

La implementacion tipica usa `Objects.hash()` que combina los codigos hash de los campos relevantes con un multiplicador primo para reducir colisiones.

## clone()

`clone()` crea y devuelve una copia del objeto. La implementacion por defecto en `Object` realiza una copia superficial (shallow copy): copia los campos primitivos y las referencias, pero no clona los objetos referenciados.

Para que un objeto sea clonable, la clase debe implementar la interfaz `Cloneable` (un marker interface sin metodos) y sobrescribir `clone()` como `public`.

```java
public class Direccion implements Cloneable {
    private String calle;
    private int numero;

    public Direccion(String calle, int numero) {
        this.calle = calle;
        this.numero = numero;
    }

    @Override
    public Direccion clone() {
        try {
            return (Direccion) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e); // no deberia ocurrir
        }
    }
}
```

## finalize()

`finalize()` es llamado por el recolector de basura (GC) antes de liberar la memoria de un objeto. Su uso esta **deprecado desde Java 9** y no se recomienda por varias razones: no se garantiza que se ejecute, impacta negativamente en el rendimiento del GC, y puede causar problemas de concurrencia.

```java
@Override
@Deprecated(since = "9")
protected void finalize() throws Throwable {
    // liberar recursos no administrados
    super.finalize();
}
```

Desde Java 9, las alternativas recomendadas son `try-with-resources` (para recursos que implementan `AutoCloseable`) y `Cleaner` (para limpieza mas sofisticada).

## getClass()

`getClass()` devuelve la clase en tiempo de ejecucion del objeto como una instancia de `java.lang.Class`. Es el mecanismo de reflection mas basico.

```java
Object obj = "Hola Mundo";
System.out.println(obj.getClass().getName());             // java.lang.String
System.out.println(obj.getClass().getSimpleName());       // String
System.out.println(obj.getClass().getPackageName());      // java.lang
```

`getClass()` es util para implementar `equals()` (garantizando que ambas clases sean identicas) y para trabajar con reflection dinamica.

## notify, notifyAll, wait

Estos metodos estan relacionados con la sincronizacion entre hilos (concurrencia). Solo pueden llamarse desde un bloque sincronizado (`synchronized`).

- `wait()`: libera el monitor y bloquea el hilo hasta que otro hilo llame a `notify()` o `notifyAll()`
- `notify()`: despierta un hilo que esta esperando en el monitor
- `notifyAll()`: despierta todos los hilos que esperan en el monitor

```java
public synchronized void esperar() throws InterruptedException {
    wait(); // el hilo espera
}

public synchronized void despertar() {
    notifyAll(); // despierta todos los hilos en espera
}
```

## Importancia de sobrescribir equals y hashCode

Cualquier clase que necesite comparacion logica o que se use como clave en colecciones basadas en hash **debe** sobrescribir `equals()` y `hashCode()`.

```java
// Sin sobrescritura: comportamiento incorrecto
Map<Usuario, String> tareas = new HashMap<>();
tareas.put(new Usuario("Ana"), "Tarea 1");
System.out.println(tareas.get(new Usuario("Ana"))); // null!

// Con sobrescritura: funciona correctamente
// (asumiendo que equals y hashCode estan implementados)
Map<Usuario, String> tareas2 = new HashMap<>();
tareas2.put(new Usuario("Ana"), "Tarea 1");
System.out.println(tareas2.get(new Usuario("Ana"))); // "Tarea 1"
```

Los IDE modernos (IntelliJ, Eclipse, VS Code) generan automaticamente implementaciones correctas de `equals()` y `hashCode()` seleccionando los campos relevantes. Es recomendable usar esta generacion en lugar de escribir la implementacion a mano.
