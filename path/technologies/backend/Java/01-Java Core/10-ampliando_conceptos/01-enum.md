# Enum

Los tipos `enum` en Java, introducidos en Java 5, son clases completas que extienden implicitamente `java.lang.Enum`. A diferencia de los enums de otros lenguajes (que son meros enteros con nombre), los enums de Java son tipos referenciales que pueden tener campos, metodos y constructores.

## Definicion basica

```java
public enum Dia {
    LUNES, MARTES, MIERCOLES, JUEVES, VIERNES, SABADO, DOMINGO
}
```

Cada constante enum es una instancia publica, estatica y final del tipo enum. Son objetos singleton creados de forma segura por la JVM.

Uso:

```java
Dia hoy = Dia.MIERCOLES;
System.out.println(hoy); // MIERCOLES (toString() devuelve el nombre)
```

## Campos, metodos y constructores

Los enums pueden tener miembros como cualquier clase. El constructor debe ser `private` (o package-private); no se puede instanciar un enum con `new`.

```java
public enum Planeta {
    MERCURIO(3.303e23, 2.4397e6),
    VENUS(4.869e24, 6.0518e6),
    TIERRA(5.976e24, 6.37814e6),
    MARTE(6.421e23, 3.3972e6),
    JUPITER(1.9e27, 7.1492e7),
    SATURNO(5.688e26, 6.0268e7),
    URANO(8.686e25, 2.5559e7),
    NEPTUNO(1.024e26, 2.4746e7);

    private final double masa;
    private final double radio;

    Planeta(double masa, double radio) {
        this.masa = masa;
        this.radio = radio;
    }

    public double gravedadSuperficial() {
        return 6.6726e-11 * masa / (radio * radio);
    }

    public double masa() { return masa; }
    public double radio() { return radio; }
}
```

```java
double gravedad = Planeta.TIERRA.gravedadSuperficial();
```

## Metodos heredados de Enum

```java
Dia[] todos = Dia.values();         // devuelve todas las constantes
Dia d = Dia.valueOf("LUNES");       // obtiene constante por nombre (IllegalArgumentException si no existe)
String nombre = Dia.LUNES.name();   // "LUNES"
int ordinal = Dia.MIERCOLES.ordinal(); // 2 (posicion, 0-indexado)
int comparado = Dia.LUNES.compareTo(Dia.MARTES); // negativo
```

> `name()` devuelve el nombre exacto de la constante. `toString()` tambien devuelve el nombre por defecto, pero puede sobrescribirse. `ordinal()` debe usarse con cuidado: depende del orden en que se declaran las constantes.

## Switch con enums

```java
Dia dia = Dia.MARTES;
switch (dia) {
    case LUNES -> System.out.println("Inicio de semana");
    case VIERNES -> System.out.println("Finde cerca");
    case SABADO, DOMINGO -> System.out.println("Finde");
    default -> System.out.println("Dia normal");
}
```

```java
// Switch tradicional
switch (dia) {
    case LUNES:
        System.out.println("Inicio");
        break;
    default:
        System.out.println("Otro");
}
```

En un `switch` sobre un enum no se usa el nombre completo de la constante (sin `Dia.`).

## Metodos abstractos en enums

Cada constante puede implementar su propio comportamiento:

```java
public enum Operacion {
    SUMA {
        public int aplicar(int a, int b) { return a + b; }
    },
    RESTA {
        public int aplicar(int a, int b) { return a - b; }
    },
    MULTIPLICA {
        public int aplicar(int a, int b) { return a * b; }
    };

    public abstract int aplicar(int a, int b);
}
```

```java
int resultado = Operacion.SUMA.aplicar(5, 3); // 8
```

## EnumSet

`EnumSet` es una implementacion de `Set` optimizada para enums, basada en vectores de bits.

```java
EnumSet<Dia> laborables = EnumSet.range(Dia.LUNES, Dia.VIERNES);
EnumSet<Dia> finDeSemana = EnumSet.of(Dia.SABADO, Dia.DOMINGO);
EnumSet<Dia> todos = EnumSet.allOf(Dia.class);
```

## EnumMap

`EnumMap` es un `Map` especializado para claves enum, implementado como un array indexado por ordinal.

```java
EnumMap<Dia, String> horarios = new EnumMap<>(Dia.class);
horarios.put(Dia.LUNES, "09:00 - 18:00");
horarios.put(Dia.SABADO, "10:00 - 14:00");
```

## Singleton con enum

Un enum con una sola constante es la forma mas robusta de implementar el patron Singleton. La JVM garantiza una unica instancia, incluso frente a serializacion y reflexion.

```java
public enum DatabaseConnection {
    INSTANCE;

    private Connection connection;

    DatabaseConnection() {
        // inicializar conexion
    }

    public Connection getConnection() {
        return connection;
    }
}
```

```java
DatabaseConnection.INSTANCE.getConnection();
```

Ventajas frente al Singleton clasico:

- Thread-safe por definicion (la JVM garantiza una unica instancia de cada constante enum).
- Protegido contra ataques de reflexion (el constructor solo se invoca una vez).
- Serializacion segura: los enums se serializan por nombre, no por estado.
- `readResolve()` y `writeReplace()` no son necesarios.

## Limitaciones

- No se puede extender un enum: todos los enums extienden implicitamente `java.lang.Enum`.
- Un enum no puede ser superclase de otro enum ni de una clase.
- No se pueden crear instancias via reflexion.
- Los enums no pueden ser parametrizados con genericos.
- Los metodos de un enum no pueden ser `abstract` a menos que cada constante los implemente.

```java
// Incorrecto: no se puede extender un enum
public enum DiaExtendido extends Dia { } // ERROR
```
