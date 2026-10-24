# Condicionales

Los condicionales permiten que un programa ejecute distintos bloques de código según se cumpla o no una expresión booleana. Java ofrece la sentencia `if` con sus variantes y la sentencia `switch`, que ha evolucionado significativamente desde sus orígenes hasta incorporar pattern matching en versiones recientes.

## if, else if y else

La sentencia `if` evalúa una expresión booleana y ejecuta el bloque asociado si el resultado es `true`. Cuando se necesita ramificar entre más de dos caminos se encadenan `else if`. El `else` final captura todos los casos no contemplados.

```java
public class EjemploIf {
    public static void main(String[] args) {
        int nota = 7;

        if (nota >= 9) {
            System.out.println("Sobresaliente");
        } else if (nota >= 7) {
            System.out.println("Notable");
        } else if (nota >= 5) {
            System.out.println("Aprobado");
        } else {
            System.out.println("Suspenso");
        }
    }
}
```

Java no posee un tipo booleano distinto de `boolean`, por lo que las condiciones deben ser estrictamente booleanas. No se permite `if (x)` donde `x` es un entero, a diferencia de C o Python.

```java
int x = 5;
// if (x) {} // ERROR de compilacion
if (x != 0) {} // Correcto
```

## Operadores ternarios

El operador ternario `? :` ofrece una forma concisa de expresar una elección entre dos valores. Es especialmente útil para asignaciones condicionales.

```java
int edad = 20;
String tipo = (edad >= 18) ? "Mayor de edad" : "Menor de edad";
```

El ternario no reemplaza a `if` complejos. Cuando la expresión a evaluar es larga o los bloques contienen múltiples sentencias, `if` resulta más legible.

## Switch clasico

La sentencia `switch` compara una expresión contra múltiples valores常量 de forma más estructurada que un encadenamiento de `if`. Cada caso se marca con `case` y debe terminar con `break` para evitar la ejecución del caso siguiente (caída o *fall-through*).

```java
public class EjemploSwitch {
    public static void main(String[] args) {
        int dia = 3;
        String nombreDia;

        switch (dia) {
            case 1:
                nombreDia = "Lunes";
                break;
            case 2:
                nombreDia = "Martes";
                break;
            case 3:
                nombreDia = "Miercoles";
                break;
            case 4:
                nombreDia = "Jueves";
                break;
            case 5:
                nombreDia = "Viernes";
                break;
            default:
                nombreDia = "Fin de semana";
                break;
        }

        System.out.println("Dia: " + nombreDia);
    }
}
```

El `break` es obligatorio en cada caso. Sin él, el flujo de ejecución cae al caso siguiente (*fall-through*), lo cual puede ser intencional pero generalmente es un error.

> El *fall-through* intencional es útil cuando múltiples casos comparten la misma lógica. Por ejemplo, `case 6:` y `case 7:` pueden ejecutar el mismo código si no se pone `break` entre ellos.

## Fall-through intencional

```java
int mes = 2;
String estacion;

switch (mes) {
    case 12: case 1: case 2:
        estacion = "Invierno";
        break;
    case 3: case 4: case 5:
        estacion = "Primavera";
        break;
    case 6: case 7: case 8:
        estacion = "Verano";
        break;
    case 9: case 10: case 11:
        estacion = "Otonio";
        break;
    default:
        estacion = "Mes invalido";
        break;
}
```

Este patrón de agrupar múltiples `case` sin `break` intermedio es la forma idiomática de expresar rangos de valores en `switch` clásico.

## Switch expressions con yield

Desde Java 14 (preview) y Java 15 (estable), `switch` puede usarse como expresión que devuelve un valor. Se emplea la flecha `->` en lugar de `case: break`, y el valor se asigna directamente.

```java
public class EjemploSwitchExpression {
    public static void main(String[] args) {
        int codigo = 2;

        String resultado = switch (codigo) {
            case 1 -> "Uno";
            case 2 -> "Dos";
            case 3 -> "Tres";
            default -> "Desconocido";
        };

        System.out.println(resultado);
    }
}
```

Cuando el caso necesita ejecutar múltiples sentencias antes de devolver un valor, se usa `yield` para indicar el valor de retorno del caso.

```java
int temperatura = 35;

String nivel = switch (temperatura) {
    case 1 -> "Bajo";
    case 2 -> "Medio";
    case 3 -> {
        System.out.println("Alerta: nivel critico");
        yield "Alto";
    }
    default -> "Sin dato";
};
```

> Las expresiones `switch` con `->` no permiten *fall-through*. Cada rama es independiente y autocontenida, lo cual elimina una fuente frecuente de errores.

## Switch con Strings y enums

`switch` funciona con tipos `String` y enumeraciones (`enum`), además de tipos numéricos y `char`. Los Strings se comparan por contenido (igualdad de valores), no por referencia.

```java
public class EjemploSwitchString {
    enum DiaSemana {
        LUNES, MARTES, MIERCOLES, JUEVES, VIERNES, SABADO, DOMINGO
    }

    public static void main(String[] args) {
        String color = "azul";

        switch (color) {
            case "rojo":
                System.out.println("Color pasion");
                break;
            case "azul":
                System.out.println("Color calma");
                break;
            case "verde":
                System.out.println("Color naturaleza");
                break;
            default:
                System.out.println("Color desconocido");
                break;
        }

        DiaSemana hoy = DiaSemana.MARTES;

        switch (hoy) {
            case LUNES, MARTES, MIERCOLES, JUEVES, VIERNES:
                System.out.println("Dia laborable");
                break;
            case SABADO, DOMINGO:
                System.out.println("Fin de semana");
                break;
        }
    }
}
```

> Los `enum` en `switch` se referencian sin el nombre del tipo (`LUNES`, no `DiaSemana.LUNES`). Esto es posible porque Java infiere el tipo de la expresión evaluada.

## Pattern matching para switch (Java 21)

Java 21 introduce pattern matching para `switch`, que permite combinar la comparación de valores con la verificación de tipo y la extracción de variables. Esto elimina la necesidad de múltiples `instanceof` encadenados.

```java
public class EjemploPatternSwitch {
    static String describir(Object obj) {
        return switch (obj) {
            case Integer i -> "Entero: " + i;
            case String s  -> "Cadena de longitud " + s.length();
            case double[] d -> "Array de " + d.length + " elementos";
            case null      -> "Es nulo";
            default        -> "Tipo desconocido: " + obj.getClass().getSimpleName();
        };
    }

    public static void main(String[] args) {
        System.out.println(describir(42));
        System.out.println(describir("Hola"));
        System.out.println(describir(new double[]{1.0, 2.0}));
        System.out.println(describir(null));
    }
}
```

El orden de los patrones importa. Los patrones más específicos deben ir antes de los más generales, ya que el primer patrón que coincida captura el valor. Un patrón `default` siempre debe ir al final.

```java
static String clasificar(Object obj) {
    return switch (obj) {
        case Number n when n.intValue() > 0 -> "Numero positivo: " + n;
        case Number n when n.intValue() < 0 -> "Numero negativo: " + n;
        case Number n -> "Numero cero";
        case String s -> "Cadena: " + s;
        default -> "Otro";
    };
}
```

Los *guarded patterns* (`when`) añaden una condición adicional al patrón de tipo. Se evalúan en orden: el primer `when` que se cumpla determina la rama ejecutada.

> Pattern matching para `switch` requiere Java 21 o superior. En versiones anteriores, la alternativa es una serie de `if (x instanceof Tipo t)` encadenados.
