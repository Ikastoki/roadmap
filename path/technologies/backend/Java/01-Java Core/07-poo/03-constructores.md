# Constructores

Un constructor es un metodo especial que se ejecuta automaticamente cuando se crea un objeto con `new`. Su proposito es inicializar el estado del objeto recien creado, asignando valores iniciales a sus campos y preparando el objeto para su uso. A diferencia de los metodos normales, un constructor no tiene tipo de retorno, no puede ser `static`, `final` ni `abstract`, y su nombre coincide exactamente con el nombre de la clase.

```java
public class Producto {
    String nombre;
    double precio;

    // Constructor
    public Producto(String nombre, double precio) {
        this.nombre = nombre;
        this.precio = precio;
    }
}

Producto p = new Producto("Laptop", 999.99);
```

Cuando se ejecuta `new Producto("Laptop", 999.99)`, Java reserva memoria para el objeto, luego invoca el constructor pasandole los argumentos `"Laptop"` y `999.99`, que se asignan a los campos mediante la sentencia `this.nombre = nombre`.

## Constructor por defecto

Si no se define ningun constructor en una clase, Java proporciona automaticamente un constructor sin parametros llamado **constructor por defecto**. Este constructor no realiza ninguna inicializacion adicional mas alla de asignar valores por defecto a los campos.

```java
public class Animal {
    String nombre;
    int patas;
}

// Java genera implicitamente:
// public Animal() {}

Animal a = new Animal();  // Funciona gracias al constructor por defecto
```

En el momento en que se define cualquier constructor explicito, el constructor por defecto deja de generarse. Si se necesita un constructor sin parametros junto con otros constructores, debe declararse manualmente.

## Constructores parametrizados

Un constructor parametrizado acepta argumentos que permiten inicializar los campos con valores especificos en el momento de la creacion del objeto. Esto garantiza que el objeto nace con un estado valido.

```java
public class Cuenta {
    private String titular;
    private double saldo;

    public Cuenta(String titular, double saldoInicial) {
        this.titular = titular;
        this.saldo = saldoInicial;
    }
}

Cuenta c = new Cuenta("Maria", 5000.0);
```

Los constructores parametrizados son la forma mas comun de inicializar objetos en Java. Permiten evitar la construccion de objetos en estados incompletos o inconsistentes.

## Constructor sobrecargado

Una clase puede tener multiples constructores siempre que sus listas de parametros sean distintas. A este fenomeno se le llama **sobrecarga de constructores** y permite crear objetos de diferentes maneras segun las necesidades del momento.

```java
public class Rectangulo {
    private double ancho;
    private double alto;

    public Rectangulo(double ancho, double alto) {
        this.ancho = ancho;
        this.alto = alto;
    }

    public Rectangulo(double lado) {
        this.ancho = lado;
        this.alto = lado;
    }

    public Rectangulo() {
        this.ancho = 1.0;
        this.alto = 1.0;
    }
}

Rectangulo r1 = new Rectangulo(10, 5);
Rectangulo r2 = new Rectangulo(7);       // Cuadrado de lado 7
Rectangulo r3 = new Rectangulo();         // Cuadrado unitario
```

Java resuelve cual constructor invocar en tiempo de compilacion, comparando el numero y tipo de argumentos con las listas de parametros disponibles.

## This en constructores

La palabra clave `this` hace referencia al objeto actual. Dentro de un constructor, `this.param` distingue un campo de la clase de un parametro con el mismo nombre.

```java
public class Persona {
    private String nombre;
    private int edad;

    public Persona(String nombre, int edad) {
        this.nombre = nombre;  // this.nombre es el campo, nombre es el parametro
        this.edad = edad;
    }
}
```

Ademas, `this()` permite invocar otro constructor de la misma clase, facilitando la reutilizacion de codigo de inicializacion entre constructores.

```java
public class Persona {
    private String nombre;
    private int edad;
    private boolean activo;

    public Persona(String nombre, int edad) {
        this(nombre, edad, true);
    }

    public Persona(String nombre, int edad, boolean activo) {
        this.nombre = nombre;
        this.edad = edad;
        this.activo = activo;
    }
}
```

La llamada a `this()` debe ser la primera sentencia del constructor. No es posible usar `this()` y `super()` en el mismo constructor, ya que ambos deben ejecutarse al inicio.

## Inicializacion de campos

Ademas de los constructores, Java ofrece otros mecanismos para asignar valores iniciales a los campos.

### Inicializadores de instancia

Un bloque de inicializacion de instancia es un bloque de codigo dentro de la clase que se ejecuta despues de asignar los valores por defecto y antes del constructor. Se ejecuta cada vez que se crea un objeto.

```java
public class Conexion {
    private String id;
    private long timestamp;

    {
        this.id = generarId();
        this.timestamp = System.currentTimeMillis();
    }

    private String generarId() {
        return "CONN-" + System.nanoTime();
    }
}
```

Si hay multiples bloques de inicializacion, se ejecutan en el orden en que aparecen en el codigo fuente.

### Inicializadores estaticos

Un bloque estatico se ejecuta una sola vez, cuando la clase se carga por primera vez en el ClassLoader. Se utiliza para inicializar campos estaticos con valores que requieren logica compleja.

```java
public class Configuracion {
    private static Map<String, String> propiedades;

    static {
        propiedades = new HashMap<>();
        propiedades.put("db.url", "jdbc:mysql://localhost:3306/miDB");
        propiedades.put("db.user", "admin");
    }
}
```

Los bloques estaticos son utiles cuando la inicializacion de un campo estatico no puede realizarse en una sola expresion.

### Orden de inicializacion

El orden completo de ejecucion al crear un objeto es el siguiente:

1. Se asignan valores por defecto a todos los campos (0, `null`, `false`).
2. Se ejecutan los bloques de inicializacion de instancia en orden de aparicion.
3. Se ejecuta el constructor (`super()` primero si se invoca).
4. Se asignan los valores proporcionados en el constructor a los campos.

```java
public class Orden {
    private int a = 1;
    {
        System.out.println("Bloque de instancia: a = " + a);
    }
    private int b = a + 1;

    public Orden() {
        System.out.println("Constructor: a = " + a + ", b = " + b);
    }
}

// Salida al crear new Orden():
// Bloque de instancia: a = 1
// Constructor: a = 1, b = 2
```

> Los inicializadores de instancia se ejecutan antes del cuerpo del constructor, pero despues de la asignacion de valores por defecto. Esto significa que si un inicializador de instancia depende de un campo que el constructor asigna, ese campo tendra el valor por defecto (0 o null) en el momento de la ejecucion del inicializador.
