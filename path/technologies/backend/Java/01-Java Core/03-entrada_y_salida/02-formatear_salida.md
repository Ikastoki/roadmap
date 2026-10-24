# Formatear salida

El formateo de salida permite controlar con precisión cómo se presentan los datos: anchura, precisión decimal, alineación y relleno. Java proporciona `System.out.printf` para impresión directa y `String.format()` para generar cadenas formateadas sin imprimir inmediatamente. Ambos métodos comparten la misma sintaxis de especificadores de formato.

## Especificadores de formato basicos

Cada especificador comienza con `%` y termina con un carácter que indica el tipo de dato a formatear. Los más utilizados son `%s` para cadenas, `%d` para enteros, `%f` para punto flotante, `%c` para caracteres y `%b` para booleanos.

```java
public class EjemploBasicos {
    public static void main(String[] args) {
        String nombre = "Laura";
        int edad = 29;
        double peso = 58.6;
        char genero = 'F';
        boolean activo = true;

        System.out.printf("Nombre: %s%n", nombre);
        System.out.printf("Edad: %d anios%n", edad);
        System.out.printf("Peso: %f kg%n", peso);
        System.out.printf("Genero: %c%n", genero);
        System.out.printf("Activo: %b%n", activo);
    }
}
```

Salida:

```
Nombre: Laura
Edad: 29 anios
Peso: 58.600000 kg
Genero: F
Activo: true
```

Por defecto, `%f` muestra seis decimales. Para controlar la precisión se utiliza el modificador de precisión.

## Precision decimal

La precisión se especifica colocando un punto y un número entre `%` y el carácter de tipo. Para `%f` se interpreta como el número de cifras decimales.

```java
public class EjemploPrecision {
    public static void main(String[] args) {
        double precio = 45.678;

        System.out.printf("Sin precision: %f%n", precio);
        System.out.printf("Dos decimales: %.2f%n", precio);
        System.out.printf("Un decimal: %.1f%n", precio);
        System.out.printf("Sin decimales: %.0f%n", precio);
    }
}
```

Salida:

```
Sin precision: 45.678000
Dos decimales: 45.68
Un decimal: 45.7
Sin decimales: 46
```

> La redondeo en `printf` sigue las reglas de redondeo de la máquina (banker's rounding en algunos casos). Para control total se puede utilizar `BigDecimal` con su propio modo de redondeo.

## Anchura de campo

La anchura se indica con un número entero entre `%` y el carácter de tipo. Si el valor formateado es más corto que la anchura, se rellena con espacios por defecto. Si es más largo, se muestra completo sin truncar.

```java
public class EjemploAnchura {
    public static void main(String[] args) {
        String nombre = "Carlos";
        int codigo = 42;
        double salario = 1500.0;

        System.out.printf("[%10s]%n", nombre);
        System.out.printf("[%05d]%n", codigo);
        System.out.printf("[%10.2f]%n", salario);
        System.out.printf("[%-10s][%10s]%n", "Izq", "Der");
    }
}
```

Salida:

```
[   Carlos]
[00042]
[   1500.00]
[Izq       ][       Der]
```

El especificador `%05d` rellena con ceros a la izquierda hasta alcanzar 5 caracteres. Los especificadores `%-10s` alinean el texto a la izquierda dentro de un campo de 10 caracteres.

## Alineacion y relleno

Java permite controlar la alineación (izquierda con `-`, derecha por defecto) y el carácter de relleno. Para cambiar el carácter de relleno se coloca el carácter deseado justo después de `%`.

```java
public class EjemploAlineacion {
    public static void main(String[] args) {
        System.out.printf("[%10s]%n", "Derecha");
        System.out.printf("[%-10s]%n", "Izquierda");
        System.out.printf("[%010d]%n", 123);
        System.out.printf("[%+10d]%n", 123);
        System.out.printf("[%+10d]%n", -123);

        // Relleno con caracter personalizado
        System.out.printf("[%10s]%n", "relleno");
        System.out.printf("[%-10s]%n", "relleno");
    }
}
```

Salida:

```
[ Derecha]
[Izquierda ]
[0000000123]
[      +123]
[      -123]
[  relleno]
[relleno   ]
```

> El especificador `+` fuerza a mostrar el signo positivo (+) o negativo (-) en números. Sin `+`, solo los negativos muestran el signo.

## Formateo de cadenas

El especificador `%s` también acepta modificadores de anchura y alineación. Esto permite alinear columnas de texto en tablas simples.

```java
public class EjemploCadenas {
    public static void main(String[] args) {
        String[] productos = {"Teclado", "Raton", "Monitor"};
        double[] precios = {45.99, 29.50, 399.99};

        for (int i = 0; i < productos.length; i++) {
            System.out.printf("%-12s %10.2f€%n", productos[i], precios[i]);
        }
    }
}
```

Salida:

```
Teclado          45.99€
Raton            29.50€
Monitor         399.99€
```

## String.format()

`String.format()` funciona exactamente igual que `printf`, pero devuelve una `String` en lugar de imprimir directamente. Esto resulta útil para construir mensajes que se pasarán a otros métodos, logging o interfaces de usuario.

```java
public class EjemploStringFormat {
    public static void main(String[] args) {
        String nombre = "Pedro";
        double nota = 8.75;

        String mensaje = String.format("El alumno %s ha obtenido %.1f", nombre, nota);
        System.out.println(mensaje);

        // Construccion de un identificador unico
        String id = String.format("USR-%05d", 142);
        System.out.println(id);

        // Generacion de un reporte
        String reporte = String.format(
                "==== REPORTE ====%n" +
                "Alumno: %-20s%n" +
                "Nota:   %6.2f%n" +
                "Estado: %s%n",
                nombre, nota, nota >= 5.0 ? "Aprobado" : "Suspendido"
        );
        System.out.println(reporte);
    }
}
```

Salida:

```
El alumno Pedro ha obtenido 8.8
USR-00142
==== REPORTE ====
Alumno: Pedro
Nota:     8.75
Estado: Aprobado
```

## Conversiones de formato adicionales

Además de los especificadores básicos, Java soporta `%t` para fechas y `%h` para hashcode hexadecimal.

```java
import java.time.LocalDate;

public class EjemploConversiones {
    public static void main(String[] args) {
        LocalDate hoy = LocalDate.now();

        System.out.printf("Fecha completa: %tD%n", hoy);
        System.out.printf("Dia: %td, Mes: %tB, Anio: %tY%n", hoy, hoy, hoy);

        String texto = "Java";
        System.out.printf("HashCode de '%s': %h%n", texto, texto.hashCode());
    }
}
```

Los modificadores `%tD`, `%td`, `%tB` y `%tY` extraen componentes de una fecha: `%tD` muestra la fecha en formato MM/dd/yy, `%td` el día, `%tB` el nombre del mes y `%tY` el año con cuatro dígitos.

> Los especificadores de fecha son extensos y poco intuitivos. Para formatos de fecha complejos, `DateTimeFormatter` de la API de `java.time` es la alternativa recomendada.
