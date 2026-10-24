# Introduccion a clases y objetos

Una clase en Java es la unidad fundamental de organizacion del codigo. Define un tipo personalizado que agrupa datos (campos) y comportamiento (metodos) en una sola entidad. Cada clase actua como una plantilla que describe que tiene un objeto y que puede hacer, sin reservar memoria por si misma. Solo cuando se instancia un objeto se asignan recursos reales en el heap para almacenar su estado.

La sintaxis basica de una clase combina modificadores de acceso, nombre, campos y metodos. Los campos representan el estado del objeto y los metodos definen las operaciones que puede realizar.

```java
public class Vehiculo {
    String marca;
    int velocidad;

    void acelerar(int incremento) {
        velocidad += incremento;
    }

    void frenar() {
        velocidad = 0;
    }
}
```

## Creacion de objetos

Para crear un objeto se utiliza el operador `new`, que reserva memoria en el heap y devuelve una referencia al objeto recien creado. Esta referencia se almacena en una variable local o de instancia.

```java
Vehiculo miCoche = new Vehiculo();
```

La ejecucion de esta linea implica varios pasos: primero se reserva memoria suficiente para todos los campos de la clase, luego se inicializan esos campos a sus valores por defecto (0 para enteros, `null` para referencias, `false` para booleanos) y finalmente se invoca el constructor de la clase.

```java
Vehiculo v1 = new Vehiculo();
Vehiculo v2 = new Vehiculo();

v1.marca = "Seat";
v2.marca = "Ford";

v1.acelerar(50);
```

`v1` y `v2` son dos objetos independientes. Modificar el estado de uno no afecta al otro, ya que cada uno tiene su propio espacio de memoria en el heap.

## Campos y atributos

Los campos (tambien llamados atributos o variables de instancia) definen el estado de un objeto. Cada objeto tiene su propia copia de los campos de instancia, con valores independientes.

```java
public class Persona {
    String nombre;
    int edad;
    boolean activo;
}
```

Cuando se crea un objeto de `Persona`, tiene sus propios valores de `nombre`, `edad` y `activo`. Si no se asigna un valor explicito, Java aplica valores por defecto:

| Tipo | Valor por defecto |
|---|---|
| `int`, `long`, `short`, `byte` | `0` |
| `float`, `double` | `0.0` |
| `char` | `'\u0000'` |
| `boolean` | `false` |
| Referencias (`String`, objetos) | `null` |

Los campos pueden tener modificadores que afectan su comportamiento: `static` comparte el valor entre todas las instancias, `final` lo hace constante una vez asignado, y los modificadores de acceso controlan la visibilidad.

## Metodos

Los metodos definen el comportamiento de un objeto. Representan acciones que el objeto puede realizar, y pueden modificar su estado interno, recibir parametros y devolver resultados.

```java
public class CuentaBancaria {
    double saldo;

    void depositar(double cantidad) {
        if (cantidad > 0) {
            saldo += cantidad;
        }
    }

    boolean retirar(double cantidad) {
        if (cantidad > 0 && cantidad <= saldo) {
            saldo -= cantidad;
            return true;
        }
        return false;
    }

    double consultarSaldo() {
        return saldo;
    }
}
```

Los metodos pueden tener cualquier modificador de acceso. Si un metodo es `void`, no devuelve nada; en caso contrario, el tipo de retorno indica que tipo de valor devuelve. Los parametros formales definen que datos acepta el metodo, y los argumentos reales son los valores que se pasan al invocarlo.

## Referencias de objeto

En Java, las variables de tipo objeto no contienen el objeto en si, sino una referencia (un apuntador) hacia la posicion de memoria donde reside el objeto en el heap.

```java
String mensaje = new String("Hola");
```

La variable `mensaje` almacena una referencia, no el texto "Hola". El texto vive en el heap y `mensaje` es el camino para llegar a el. Esto tiene implicaciones importantes: dos variables pueden referenciar el mismo objeto.

```java
Vehiculo a = new Vehiculo();
Vehiculo b = a;  // b apunta al mismo objeto que a

b.marca = "Renault";
System.out.println(a.marca);  // Imprime "Renault"
```

Asignar `a` a `b` no crea un nuevo objeto; simplemente copia la referencia. Ambas variables apuntan a la misma instancia en el heap.

## Null

Una referencia puede no apuntar a ningun objeto. En ese caso, su valor es `null`. Intentar acceder a campos o metodos a traves de una referencia `null` provoca una `NullPointerException` en tiempo de ejecucion.

```java
Vehiculo v = null;
v.acelerar(30);  // NullPointerException
```

`null` es un literal que representa la ausencia de referencia. Es importante inicializar las variables antes de usarlas o verificar que no son `null` antes de invocar metodos sobre ellas.

## Estado y comportamiento

Un objeto combina estado y comportamiento en una sola entidad. El estado son los valores actuales de sus campos; el comportamiento son las operaciones que puede realizar mediante sus metodos. Esta combinacion es la esencia de la POO: los datos y las operaciones sobre ellos viven juntos, protegidos por la encapsulacion.

```java
public class Termostato {
    private double temperaturaActual;
    private double temperaturaObjetivo;

    public void subirTemperatura(double grados) {
        temperaturaObjetivo += grados;
    }

    public void bajarTemperatura(double grados) {
        temperaturaObjetivo -= grados;
    }

    public boolean necesitaCalentar() {
        return temperaturaActual < temperaturaObjetivo;
    }
}
```

El termostato tiene estado (temperatura actual y objetivo) y comportamiento (subir, bajar, comparar). Nadie fuera del objeto puede modificar directamente la temperatura; solo puede interactuar a traves de la interfaz publica. Esta separacion entre estado interno y superficie de interaccion es el fundamento de la encapsulacion.
