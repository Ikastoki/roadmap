# Representación de números enteros

Los números enteros son la forma más básica de representación numérica en la computación. Desde un simple contador hasta las direcciones de memoria, los enteros son omnipresentes en todos los niveles de un sistema informático. La forma en que un ordenador almacena y manipula enteros tiene implicaciones directas sobre el rango de valores representables, la precisión de los cálculos y el comportamiento de los programas cuando los valores exceden los límites esperados.

La representación de enteros en computación no es tan directa como podría parecer: los números negativos requieren convenciones especiales, el tamaño fijo de los registros impone límites al rango de valores, y las operaciones aritméticas pueden producir resultados inesperados cuando los resultados exceden la capacidad de representación.

## El concepto de bit de signo

La representación más intuitiva de números negativos utiliza un bit de signo (*sign bit*): el bit más significativo indica si el número es positivo (0) o negativo (1). En un entero de 8 bits con bit de signo, el bit más a la izquierda determina el signo y los 7 bits restantes representan el valor absoluto.

Sin embargo, esta representación tiene una anomalía: el cero tiene dos representaciones (+0 y -0), lo que complica el hardware de la unidad aritmético-lógica (ALU) y puede causar errores sutiles en comparaciones y operaciones.

## El Complemento a Dos

El complemento a dos (*two's complement*) es el estándar universal para representar números enteros con signo en la computación moderna. Elimina la duplicidad del cero y permite utilizar el mismo circuito hardware para sumar números positivos y negativos, simplificando enormemente el diseño del procesador.

Para obtener el complemento a dos de un número, se invierten todos los bits y se suma 1:

```
+5  = 00000101
-5  = 11111011 (invertir bits: 11111010, sumar 1: 11111011)
```

La propiedad clave del complemento a dos es que la suma de un número y su negativo produce cero:

```
  00000101  (+5)
+ 11111011  (-5)
-----------
  00000000  (0) con carry descartado
```

Esta propiedad permite que el mismo circuito de sumador funcione tanto para la suma como para la resta, eliminando la necesidad de circuitos separados para cada operación.

## Rangos de valores

El rango de valores representables depende del número de bits utilizado:

| Tamaño | Rango (con signo) | Rango (sin signo) |
|--------|-------------------|-------------------|
| 8 bits | -128 a +127 | 0 a 255 |
| 16 bits | -32,768 a +32,767 | 0 a 65,535 |
| 32 bits | -2,147,483,648 a +2,147,483,647 | 0 a 4,294,967,295 |
| 64 bits | -9.2×10^18 a +9.2×10^18 | 0 a 1.8×10^19 |

El overflow (*desbordamiento*) ocurre cuando el resultado de una operación aritmética excede el rango representable. En un entero de 8 bits sin signo, sumar 255 + 1 produce 0 (con carry). En un entero con signo de 8 bits, sumar 127 + 1 produce -128. Este comportamiento puede causar errores graves en programas que no verifican los límites.

> El overflow de enteros es una fuente frecuente de vulnerabilidades de seguridad. Un atacante que puede inducir un overflow en un programa puede manipular el flujo de ejecución, sobrescribir datos en la pila o el montículo, y potencialmente ejecutar código arbitrario. Las protecciones contra overflow (stack canaries, ASLR, DEP) son componentes estándar de los sistemas operativos modernos.

