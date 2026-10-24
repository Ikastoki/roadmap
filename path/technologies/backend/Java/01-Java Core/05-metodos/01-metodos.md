# Metodos

Un metodo en Java es un bloque de codigo reutilizable que pertenece a una clase y que ejecuta una tarea especifica. Los metodos son los responsables de definir el comportamiento de los objetos: todo lo que un objeto puede hacer se modela a traves de sus metodos. En Java, a diferencia de lenguajes como C++, no existen funciones globales; toda funcion debe residir dentro de una clase.

## Definicion de un metodo

La sintaxis completa de un metodo incluye modificadores de acceso, tipo de retorno, nombre, lista de parametros (puede estar vacia) y cuerpo delimitado por llaves.

```java
public int calcularTotal(int precio, int cantidad) {
    return precio * cantidad;
}
```

Cada elemento tiene una funcion especifica:

| Elemento              | Descripcion                                    |
|-----------------------|------------------------------------------------|
| `public`              | Modificador de acceso (visibilidad)            |
| `int`                 | Tipo de retorno (puede ser `void` o un tipo)   |
| `calcularTotal`       | Nombre del metodo                              |
| `(int precio, int cantidad)` | Lista de parametros formales            |
| `{ return ... }`      | Cuerpo del metodo                              |

## Firma del metodo

La firma (signature) de un metodo esta compuesta por su nombre y el tipo y orden de sus parametros. Es lo que el compilador utiliza para distinguir un metodo de otro. El tipo de retorno no forma parte de la firma.

```java
public void imprimir(String texto)     { }
public void imprimir(int numero)       { } // misma firma? NO: cambian parametros
public int  imprimir(String texto)     { } // ERROR: misma firma que el primero
```

Las dos primeras definiciones son validas porque los parametros son diferentes (sobrecarga). La tercera no compila porque tiene la misma firma que la primera (`imprimir(String)`) aunque el retorno sea distinto.

## Modificador de acceso

Controla que otras clases pueden invocar el metodo. Los cuatro niveles de acceso son:

- `public`: accesible desde cualquier clase
- `protected`: accesible desde el mismo paquete y subclases
- `default` (sin modificador): accesible solo desde el mismo paquete
- `private`: accesible solo desde la misma clase

## Tipo de retorno

Indica el tipo del valor que el metodo devuelve mediante la sentencia `return`. Si no devuelve nada, se declara como `void`.

```java
public void mostrar(String msg) {
    System.out.println(msg);
    // no necesita return
}

public int doblar(int n) {
    return n * 2;
}
```

Si el tipo de retorno no es `void`, el metodo debe tener obligatoriamente una sentencia `return` en todas las rutas de ejecucion. El compilador verifica este requisito y rechaza el codigo si encuentra caminos sin retorno.

## Cuerpo del metodo

El cuerpo contiene las instrucciones que se ejecutan al invocar el metodo. Puede declarar variables locales, usar estructuras de control, llamar a otros metodos, y devolver un valor.

```java
public String evaluarNota(double nota) {
    if (nota < 0 || nota > 10) {
        return "Nota invalida";
    }
    if (nota >= 9) return "Sobresaliente";
    if (nota >= 7) return "Notable";
    if (nota >= 5) return "Aprobado";
    return "Suspenso";
}
```

## Convenciones de nombres

Los nombres de metodos siguen la convencion camelCase, comenzando con minuscula. El nombre debe ser un verbo o una frase verbal que describa la accion.

```
calcularTotal()      // bien
obtenerUsuario()     // bien
esValido()           // bien
setNombre()          // bien (setter)
ProcesarDatos()      // mal: mayuscula inicial
total_calcular()     // mal: snake_case en metodo
```

Los metodos que devuelven `boolean` suelen nombrarse con prefijos como `is`, `has`, `can`:

```java
public boolean isActivo() { return activo; }
public boolean hasNext() { return indice < elementos.length; }
public boolean canEdit() { return permisos.contieneEscritura(); }
```

## Ejemplo completo

```java
public class Calculadora {

    // Metodo que suma dos enteros
    public int sumar(int a, int b) {
        return a + b;
    }

    // Metodo que calcula el factorial recursivamente
    public long factorial(int n) {
        if (n <= 1) return 1;
        return n * factorial(n - 1);
    }

    // Metodo sin retorno que imprime un resultado
    public void mostrarResultado(String operacion, double resultado) {
        System.out.println("Resultado de " + operacion + ": " + resultado);
    }

    public static void main(String[] args) {
        Calculadora calc = new Calculadora();

        int suma = calc.sumar(5, 3);
        calc.mostrarResultado("suma", suma);

        long fact = calc.factorial(6);
        calc.mostrarResultado("factorial de 6", fact);
    }
}
```
