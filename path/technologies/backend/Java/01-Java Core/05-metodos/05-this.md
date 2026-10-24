# This

La palabra clave `this` es una referencia implicita al objeto actual sobre el cual se esta ejecutando un metodo de instancia. Cada metodo no estatico recibe automaticamente esta referencia, que permite al objeto acceder a sus propios miembros y distinguirlos de variables locales o parametros con el mismo nombre.

## Referencia al objeto actual

Dentro de un metodo de instancia, `this` hace referencia al objeto que invoco el metodo. Es la forma que tiene un objeto de referirse a si mismo.

```java
public class Persona {
    private String nombre;

    public void mostrar() {
        System.out.println("Objeto: " + this);
        System.out.println("Nombre: " + this.nombre);
    }
}
```

El uso de `this` es implicito cuando no hay ambiguedad. En el ejemplo anterior, `this.nombre` podria escribirse simplemente como `nombre`, y el compilador entenderia que se refiere al campo de instancia.

```java
public void mostrar() {
    System.out.println(nombre); // this es implicito
}
```

## Resolver ambiguedad entre parametro y campo

El caso mas comun donde `this` es obligatorio ocurre cuando el parametro del metodo tiene el mismo nombre que un campo de instancia. Sin `this`, el nombre referiria al parametro, no al campo.

```java
public class Producto {
    private String nombre;
    private double precio;

    public Producto(String nombre, double precio) {
        // this.nombre se refiere al campo, nombre se refiere al parametro
        this.nombre = nombre;
        this.precio = precio;
    }

    public void setNombre(String nombre) {
        this.nombre = nombre;
    }
}
```

En el constructor, la instruccion `this.nombre = nombre` asigna el valor del parametro `nombre` al campo `nombre` del objeto. Sin `this`, `nombre = nombre` seria una auto-asignacion sin efecto.

> Aunque es posible usar nombres diferentes para parametros y campos (p. ej. `String _nombre` o `String nombreParam`), la convencion en Java es usar el mismo nombre y recurrir a `this` para desambiguar.

## Constructor chaining con this()

La expresion `this()` permite invocar a otro constructor de la misma clase desde un constructor. Esto evita duplicar codigo de inicializacion y establece una cadena de constructores.

```java
public class Usuario {
    private String email;
    private String rol;
    private boolean activo;

    // Constructor principal
    public Usuario(String email, String rol, boolean activo) {
        this.email = email;
        this.rol = rol;
        this.activo = activo;
    }

    // Constructor que delega en el principal con valor por defecto
    public Usuario(String email) {
        this(email, "USUARIO", true);
    }

    // Constructor que delega en el de un parametro
    public Usuario() {
        this("anonimo@email.com");
    }
}
```

La llamada `this(...)` debe ser la primera sentencia del constructor. No se puede usar `this()` y `super()` en el mismo constructor, y no es posible llamar a `this()` dentro de un bloque try-catch ni despues de otras instrucciones.

```java
public Usuario(String email) {
    // Primera linea obligatoriamente
    this(email, "USUARIO", true);
    // Resto de inicializacion
    validarEmail(email);
}
```

Si no se encadena explicitamente con `this()`, el compilador inserta una llamada a `super()` (el constructor sin argumentos de la superclase) como primera instruccion.

## Uso de this en metodos fluent

El patron fluent (metodo que devuelve `this`) permite encadenar llamadas de forma legible:

```java
public class Configuracion {
    private String host;
    private int puerto;
    private boolean ssl;

    public Configuracion setHost(String host) {
        this.host = host;
        return this;
    }

    public Configuracion setPuerto(int puerto) {
        this.puerto = puerto;
        return this;
    }

    public Configuracion setSsl(boolean ssl) {
        this.ssl = ssl;
        return this;
    }
}

// Uso:
Configuracion config = new Configuracion()
    .setHost("localhost")
    .setPuerto(8080)
    .setSsl(true);
```

## this en clases internas

Cuando se trabaja con clases internas (inner classes), `this` se refiere a la instancia de la clase interna. Para acceder a la instancia de la clase externa se usa `ClaseExterna.this`.

```java
public class Externa {
    private String valor = "externa";

    public class Interna {
        private String valor = "interna";

        public void mostrar() {
            System.out.println(valor);          // "interna"
            System.out.println(this.valor);     // "interna"
            System.out.println(Externa.this.valor); // "externa"
        }
    }
}
```

> `this` no puede usarse en contextos estaticos, porque los metodos estaticos no tienen una instancia asociada. Intentar usar `this` dentro de un metodo `static` genera un error de compilacion.
