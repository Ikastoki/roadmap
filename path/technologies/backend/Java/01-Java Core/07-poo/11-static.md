# Static

La palabra clave `static` en Java modifica miembros que pertenecen a la clase en lugar de a instancias individuales. Un miembro estatico existe independientemente de que haya objetos creados, y todos los objetos de la clase comparten el mismo miembro estatico. Es el equivalente en Java a las variables y funciones de clase de otros lenguajes.

## Campos estaticos

Un campo estatico es una variable que pertenece a la clase, no a cada objeto. Existe una unica copia del campo, compartida por todas las instancias.

```java
public class Contador {
    private static int totalInstancias = 0;
    private final int id;

    public Contador() {
        totalInstancias++;
        this.id = totalInstancias;
    }

    public static int getTotalInstancias() {
        return totalInstancias;
    }

    public int getId() {
        return id;
    }
}

Contador c1 = new Contador();  // totalInstancias = 1
Contador c2 = new Contador();  // totalInstancias = 2
Contador c3 = new Contador();  // totalInstancias = 3

System.out.println(Contador.getTotalInstancias()); // 3
```

Los campos estaticos se inicializan cuando la clase se carga por primera vez en la JVM, antes de que se cree cualquier instancia. Se acceden mediante el nombre de la clase (`Clase.campo`), aunque tambien es posible accederlos a traves de una instancia (no recomendado).

```java
System.out.println(Contador.getTotalInstancias()); // forma correcta
// System.out.println(c1.getTotalInstancias()); // posible pero confuso
```

Los usos tipicos de campos estaticos incluyen contadores de instancias, constantes compartidas, caches y configuracion global.

## Metodos estaticos

Un metodo estatico pertenece a la clase y puede invocarse sin necesidad de crear un objeto. Los metodos estaticos no pueden acceder a miembros de instancia (`this`, campos no estaticos) directamente.

```java
public class UtilidadesMatematicas {
    public static int maximo(int a, int b) {
        return a > b ? a : b;
    }

    public static double gradosARadianes(double grados) {
        return grados * Math.PI / 180.0;
    }
}

int mayor = UtilidadesMatematicas.maximo(10, 20);
double rad = UtilidadesMatematicas.gradosARadianes(90);
```

Las limitaciones de los metodos estaticos son:

- No pueden acceder a campos de instancia directamente
- No pueden invocar metodos de instancia directamente
- No pueden usar `this` ni `super`
- No pueden ser sobrescritos (ocultan al metodo de la superclase)

```java
public class Ejemplo {
    private int valor = 10;

    public static void metodoEstatico() {
        // System.out.println(valor);  // ERROR: no hay instancia
        // this.valor = 5;              // ERROR: no existe this
    }
}
```

Los metodos estaticos son adecuados para funciones utilitarias que no dependen del estado del objeto: `Math.pow()`, `Collections.sort()`, `Integer.parseInt()`.

## Bloques estaticos

Un bloque estatico es un bloque de codigo ejecutado cuando la clase se carga por primera vez. Se usa para inicializar campos estaticos complejos o ejecutar logica unica al inicio.

```java
public class Configuracion {
    public static final Map<String, String> PROPIEDADES;
    public static final LocalDateTime INICIO;

    static {
        System.out.println("Inicializando configuracion...");
        PROPIEDADES = new HashMap<>();
        PROPIEDADES.put("url", "jdbc:mysql://localhost:3306/db");
        PROPIEDADES.put("usuario", "admin");
        PROPIEDADES.put("timeout", "5000");

        INICIO = LocalDateTime.now();
    }
}
```

Pueden existir multiples bloques estaticos en una clase, y se ejecutan en el orden en que aparecen. Todos se ejecutan antes de que cualquier metodo estatico pueda invocarse o cualquier instancia pueda crearse.

```java
public class OrdenInicializacion {
    public static int a = 1;
    public static int b;

    static {
        b = a * 2;  // ejecutado durante carga de clase
        System.out.println("Bloque estatico 1: a=" + a + ", b=" + b);
    }

    public static int c = 3;

    static {
        System.out.println("Bloque estatico 2: c=" + c);
    }
}
```

## Nested classes estaticas

Java permite definir una clase dentro de otra clase. Si la clase anidada se declara como `static`, se denomina clase anidada estatica (static nested class). Se comporta como una clase independiente pero agrupada logicamente dentro de la clase externa.

```java
public class Sistema {
    private static String version = "2.0";

    public static class Logger {
        public void log(String mensaje) {
            System.out.println("[" + version + "] " + mensaje);
        }

        public static void error(String msg) {
            System.err.println("[ERROR] " + msg);
        }
    }
}

// Uso sin instancia de Sistema
Sistema.Logger.error("Fallo critico");

Sistema.Logger logger = new Sistema.Logger();
logger.log("Inicio del sistema");
```

A diferencia de las clases internas (inner classes), las clases anidadas estaticas:

- No requieren una instancia de la clase externa para crearse
- Pueden tener miembros estaticos propios
- Solo pueden acceder a miembros estaticos de la clase externa
- Son utiles para agrupar clases logicamente relacionadas

```java
public class Calculadora {
    // Clase anidada estatica para representar resultados
    public static class Resultado {
        private final double valor;
        private final String operacion;

        public Resultado(double valor, String operacion) {
            this.valor = valor;
            this.operacion = operacion;
        }

        public double getValor() { return valor; }
        public String getOperacion() { return operacion; }
    }

    public static Resultado calcular(double a, double b, String op) {
        double resultado = switch (op) {
            case "suma" -> a + b;
            case "resta" -> a - b;
            case "multiplicacion" -> a * b;
            default -> throw new IllegalArgumentException("Operacion desconocida");
        };
        return new Resultado(resultado, op);
    }
}
```

## Factory methods estaticos

Un factory method estatico es un metodo estatico que devuelve una instancia de la clase. Es una alternativa al constructor tradicional que ofrece ventajas como nombres descriptivos, control de instancias y la posibilidad de devolver subtipos.

```java
public class Usuario {
    private final String email;
    private final String rol;

    // Constructor privado
    private Usuario(String email, String rol) {
        this.email = email;
        this.rol = rol;
    }

    // Factory methods estaticos
    public static Usuario crearAdmin(String email) {
        return new Usuario(email, "ADMIN");
    }

    public static Usuario crearUsuario(String email) {
        return new Usuario(email, "USUARIO");
    }

    public static Usuario invitado() {
        return new Usuario("invitado@sistema.com", "INVITADO");
    }
}

// Uso
Usuario admin = Usuario.crearAdmin("admin@sistema.com");
Usuario invitado = Usuario.invitado();
```

Las ventajas de los factory methods sobre los constructores son:

- Tienen nombre descriptivo (`crearAdmin`, `invitado`)
- No crean necesariamente un objeto nuevo (pueden devolver uno en cache)
- Pueden devolver cualquier subtipo de la clase
- Pueden tener logica previa a la creacion

Ejemplos clasicos en la JDK: `Integer.valueOf()`, `List.of()`, `Collections.unmodifiableList()`, `LocalDate.now()`.

> Desde Java 9, la documentacion oficial recomienda considerar factory methods estaticos como alternativa preferible a constructores publicos en muchos casos. El libro Effective Java (Item 1) de Joshua Bloch dedica varias paginas a este patron.
