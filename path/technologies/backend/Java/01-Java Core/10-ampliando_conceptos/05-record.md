# Record

Los records, introducidos como preview en Java 14 y estabilizados en Java 16, son una forma concisa de declarar clases portadoras de datos (data carriers). Un record declara automaticamente los campos, el constructor canonico, los accesores y los metodos `equals()`, `hashCode()` y `toString()` basados en todos los componentes.

## Declaracion basica

```java
public record Persona(String nombre, int edad) {}
```

Esta unica linea genera:

- Un constructor con parametros `(String nombre, int edad)`.
- Campos `private final` para cada componente.
- Accesores `nombre()` y `edad()` (sin prefijo `get`).
- `equals()`, `hashCode()` y `toString()` basados en todos los componentes.

```java
Persona p = new Persona("Ana", 30);
String nombre = p.nombre(); // "Ana"
int edad = p.edad();        // 30
String texto = p.toString(); // "Persona[nombre=Ana, edad=30]"
```

## Constructor canonico

El constructor canonico tiene los mismos parametros que los componentes. Puede definirse explicitamente para validacion o normalizacion.

```java
public record Rango(int inicio, int fin) {
    public Rango {
        if (inicio > fin) {
            throw new IllegalArgumentException("inicio debe ser <= fin");
        }
    }
}
```

Notese que no se usan parametros explicitos ni `this.inicio = inicio`. El compilador espera que en el cuerpo del constructor canonico los parametros tengan los mismos nombres que los componentes, y la asignacion a los campos es implicita.

## Compact constructor

Java permite una sintaxis compacta donde los parametros son implicitos y las asignaciones tambien:

```java
public record Persona(String nombre, String email) {
    public Persona {
        if (email != null && !email.contains("@")) {
            throw new IllegalArgumentException("email invalido");
        }
        nombre = capitalizar(nombre);
    }

    private static String capitalizar(String s) {
        if (s == null || s.isEmpty()) return s;
        return s.substring(0, 1).toUpperCase() + s.substring(1).toLowerCase();
    }
}
```

Al final del compact constructor, el compilador anade las asignaciones `this.nombre = nombre; this.email = email;` con los valores ya modificados.

## Constructor adicional

Se pueden definir constructores adicionales que deleguen en el canonico:

```java
public record Persona(String nombre, int edad) {
    public Persona(String nombre) {
        this(nombre, 0); // debe llamar al constructor canonico
    }
}
```

## Metodos generados

```java
Persona a = new Persona("Ana", 30);
Persona b = new Persona("Ana", 30);

a.equals(b);           // true (mismos valores)
a.hashCode() == b.hashCode(); // true
a.toString();          // "Persona[nombre=Ana, edad=30]"
```

Los records son inmutables: los campos son `final` y no hay setters.

## Metodos adicionales

Se pueden anadir metodos o sobrescribir los generados:

```java
public record Punto(int x, int y) {
    public double distanciaAlOrigen() {
        return Math.sqrt(x * x + y * y);
    }

    public Punto sumar(Punto otro) {
        return new Punto(this.x + otro.x, this.y + otro.y);
    }
}
```

## Records como DTOs

Los records son ideales para DTOs (Data Transfer Objects) y clases de valor:

```java
public record UsuarioDTO(long id, String nombre, String email) {}

// Uso con streams
List<UsuarioDTO> dtos = usuarios.stream()
    .map(u -> new UsuarioDTO(u.getId(), u.getNombre(), u.getEmail()))
    .collect(Collectors.toList());
```

### Records con validacion

```java
public record Email(String direccion) {
    public Email {
        if (direccion == null || !direccion.matches("^[\\w.-]+@[\\w.-]+\\.\\w{2,}$")) {
            throw new IllegalArgumentException("Email invalido");
        }
    }
}
```

## Records con componentes de tipo record

Los records pueden anidarse y combinarse:

```java
public record Direccion(String calle, String ciudad, String codigoPostal) {}
public record Cliente(String nombre, Direccion direccion) {}

Cliente c = new Cliente("Juan", new Direccion("Calle Mayor 1", "Madrid", "28001"));
```

## Records y colecciones

```java
record Producto(String id, String nombre, double precio) {}

List<Producto> productos = List.of(
    new Producto("001", "Laptop", 1200.0),
    new Producto("002", "Monitor", 350.0),
    new Producto("003", "Teclado", 80.0)
);

double total = productos.stream()
    .filter(p -> p.precio() > 100)
    .mapToDouble(Producto::precio)
    .sum();
```

### Records como claves en mapas

Los records son excelentes como claves porque implementan `equals()` y `hashCode()` correctamente basados en sus componentes.

```java
Map<Coordenada, String> mapa = new HashMap<>();
mapa.put(new Coordenada(10, 20), "punto A");
```

## Limitaciones

- No pueden extender otra clase (todos los records extienden implicitamente `java.lang.Record`).
- Son implicitamente `final`; no pueden ser abstractos ni extendidos.
- No pueden tener campos de instancia adicionales (solo los componentes).
- No pueden tener campos mutables (todos los componentes son `final`).
- No pueden tener setters.
- Los componentes no pueden ser anotados con anotaciones de retention `SOURCE` (como `@NonNull` de Lombok) a menos que se especifique `@Target(TYPE_USE)`.

```java
record IntentarExtender() extends OtraClase {} // ERROR
```

```java
record ConCampoExtra() {
    private int extra; // ERROR: campo de instancia adicional no permitido
}
```

```java
// Permitido: campos estaticos
record Configuracion(String url, int puerto) {
    private static final int PUERTO_DEFAULT = 8080;
}
```

## Records y serializacion

Los records tienen soporte especial para serializacion. El proceso de deserializacion usa el constructor canonico, lo que garantiza que las invariantes de validacion se mantengan. No es necesario declarar `serialVersionUID` explicitamente.

```java
import java.io.Serializable;

public record Datos(String nombre, int valor) implements Serializable {}
```

## Records nativos

Java 17 anadio la posibilidad de que los records usen metodos nativos para computo de hashCode, como parte de la optimizacion de la JVM.

```java
// Los records pueden declarar metodos nativos para ciertos propositos
public record Punto(int x, int y) {}
```
