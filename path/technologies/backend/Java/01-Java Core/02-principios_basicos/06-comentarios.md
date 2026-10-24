# Comentarios

Los comentarios en Java son texto ignorado por el compilador que sirve para documentar y explicar el codigo fuente. Java soporta tres tipos de comentarios: de una linea, de bloque, y de documentacion Javadoc. Cada uno tiene una finalidad distinta y unas convenciones de uso bien establecidas.

## Comentario de una linea

El comentario de una linea comienza con `//` y se extiende hasta el final de la linea. Es util para anotaciones breves, aclaraciones puntuales, o para desactivar temporalmente una linea de codigo.

```java
int contador = 0; // contador de iteraciones

// Este bucle procesa los elementos de la lista
for (int i = 0; i < lista.size(); i++) {
    // Saltamos los elementos nulos
    if (lista.get(i) == null) continue;
    procesar(lista.get(i));
}
```

Un comentario de una linea no debe explicar lo que ya es obvio. `int i = 0; // asigna 0 a i` es redundante. El comentario debe anadir valor semantico que no se deduce inmediatamente de la lectura del codigo.

## Comentario de bloque

Los comentarios de bloque envuelven el texto entre `/*` y `*/`. Pueden abarcar varias lineas y son utiles para descripciones mas largas o para desactivar temporalmente fragmentos de codigo.

```java
/*
 * Este metodo implementa el algoritmo de busqueda binaria.
 * Recibe un array ordenado y un valor a buscar.
 * Devuelve el indice o -1 si no se encuentra.
 */
public int busquedaBinaria(int[] arr, int valor) {
    // ...
}
```

Los comentarios de bloque no pueden anidarse en Java. El primer `*/` que encuentra el compilador cierra el comentario, lo que puede causar errores si se intenta comentar un bloque que ya contiene `*/`.

```java
/* Esto no funciona
   /* porque este cierre anidado */
   el parser piensa que el comentario termina aqui */
```

## Comentario de documentacion Javadoc

El Javadoc es un sistema de documentacion propio de Java que utiliza comentarios especiales `/** ... */` para generar documentacion HTML automaticamente. Es el mecanismo estandar para documentar APIs publicas.

```java
/**
 * Representa una cuenta bancaria con operaciones basicas.
 * <p>
 * Esta clase mantiene el saldo actual y permite realizar
 * depositos y retiros con validacion de fondos.
 * </p>
 *
 * @author jperez
 * @version 1.0
 * @since 2024-01-15
 */
public class CuentaBancaria {

    private double saldo;

    /**
     * Deposita una cantidad en la cuenta.
     *
     * @param cantidad  el monto a depositar, debe ser positivo
     * @return el nuevo saldo después del deposito
     * @throws IllegalArgumentException si cantidad es negativa
     */
    public double depositar(double cantidad) {
        if (cantidad < 0) {
            throw new IllegalArgumentException("La cantidad debe ser positiva");
        }
        saldo += cantidad;
        return saldo;
    }
}
```

Las etiquetas Javadoc mas comunes incluyen:

| Etiqueta       | Uso                                  |
|----------------|--------------------------------------|
| `@param`       | Documenta un parametro del metodo    |
| `@return`      | Describe el valor de retorno         |
| `@throws`      | Documenta una excepcion lanzada      |
| `@see`         | Referencia a otra clase o metodo     |
| `@since`       | Version desde la que existe el elemento |
| `@deprecated`  | Marca el elemento como obsoleto      |

> La herramienta `javadoc` procesa estos comentarios y genera documentacion HTML. Se ejecuta con `javadoc -d docs src/*.java` desde la linea de comandos, o mediante los plugins de Maven/Gradle.

## Buenas practicas con comentarios

Un comentario debe explicar el por que, no el que. El que ya lo expresa el codigo mismo si esta bien escrito. Los comentarios tienden a quedarse obsoletos cuando el codigo cambia y no se actualizan, convirtiendose en desinformacion activa.

```java
// MAL: explica lo obvio
int total = precio * cantidad; // multiplica precio por cantidad

// BIEN: explica la intencion
int total = precio * cantidad; // se aplica descuento por volumen
```

Los comentarios de codigo muerto no deben permanecer en el codigo de produccion. Si un bloque se desactiva temporalmente durante el desarrollo, debe eliminarse antes del commit. El control de versiones mantiene el historial.

En proyectos profesionales, se suele configurar una herramienta de estilo (Checkstyle, PMD) que exige Javadoc en todos los elementos publicos y rechaza el codigo que no lo tenga. Esto garantiza que la API quede documentada de forma uniforme y complete.
