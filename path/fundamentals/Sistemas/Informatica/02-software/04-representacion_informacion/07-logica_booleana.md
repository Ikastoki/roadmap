# Lógica booleana y operaciones a nivel de bit

La lógica booleana es la columna vertebral de toda la computación digital. Fue formalizada por George Boole en el siglo XIX y adoptada por los ingenieros de la computación en el siglo XX como el marco matemático que gobierna las operaciones de los circuitos digitales. Cada instrucción que un procesador ejecuta, cada decisión que un programa toma y cada cálculo que se realiza se reduce, en su nivel más fundamental, a operaciones booleanas sobre bits.

La lógica booleana opera con solo dos valores: verdadero (1) y falso (0). Esta simplicidad, lejos de ser una limitación, es la clave del poder de la computación: los circuitos booleanos pueden implementarse de forma eficiente y fiable en silicio, y la combinación de operaciones booleanas simples puede producir resultados arbitrariamente complejos.

## Álgebra de Boole

El álgebra de Boole define las operaciones lógicas y sus propiedades matemáticas. Las operaciones fundamentales son:

**AND (conjunction):** el resultado es verdadero (1) solo si ambos operandos son verdaderos.

```
0 AND 0 = 0    1 AND 0 = 0
0 AND 1 = 0    1 AND 1 = 1
```

**OR (disjunction):** el resultado es verdadero (1) si al menos uno de los operandos es verdadero.

```
0 OR 0 = 0    1 OR 0 = 1
0 OR 1 = 1    1 OR 1 = 1
```

**NOT (negation):** invierte el valor del operando.

```
NOT 0 = 1    NOT 1 = 0
```

**XOR (exclusive or):** el resultado es verdadero (1) solo si los operandos son diferentes. Es particularmente útil en criptografía y verificación de integridad.

```
0 XOR 0 = 0    1 XOR 0 = 1
0 XOR 1 = 1    1 XOR 1 = 0
```

Estas operaciones satisfacen propiedades algebraicas como la conmutatividad, la asociatividad y las leyes de De Morgan, que permiten simplificar expresiones lógicas y optimizar circuitos digitales.

> La operación XOR tiene una propiedad criptográfica fundamental: aplicar XOR dos veces con la misma clave recupera el valor original. `A XOR K XOR K = A`. Esta propiedad es la base de muchos algoritmos de cifrado simétrico.

## Operadores lógicos (AND, OR, NOT, XOR)

En la programación, los operadores lógicos se utilizan para tomar decisiones y manipular bits:

**Máscara de bits con AND:** para aislar bits específicos de un valor. Por ejemplo, para verificar si el bit 3 de un byte está encendido:

```python
valor = 0b11010110
mascara = 0b00001000
resultado = valor & mascara  # 0b00001000 (diferente de 0, bit encendido)
```

**Establecer bits con OR:** para encender bits específicos sin afectar a los demás:

```python
valor = 0b11010110
mascara = 0b00001000
resultado = valor | mascara  # 0b11011110 (bit 3 encendido)
```

**Invertir bits con XOR:** para invertir bits específicos:

```python
valor = 0b11010110
mascara = 0b00001000
resultado = valor ^ mascara  # 0b11011110 (bit 3 invertido)
```

**Comprobar paridad con XOR:** el XOR acumulativo de todos los bits de un byte determina si el número de bits encendidos es par o impar, una propiedad utilizada en detección de errores.

## Operaciones de desplazamiento (*Shift*)

Los operadores de desplazamiento mueven los bits de un número hacia la izquierda o derecha:

**Desplazamiento a la izquierda (`<<`):** mueve todos los bits hacia la izquierda, rellenando con ceros por la derecha. Cada posición de desplazamiento equivale a multiplicar por 2:

```python
5 << 1 = 10   # 0b00000101 << 1 = 0b00001010
5 << 3 = 40   # 0b00000101 << 3 = 0b00101000
```

**Desplazamiento a la derecha (`>>`):** mueve los bits hacia la derecha, rellenando con el bit de signo (desplazamiento aritmético) o con ceros (desplazamiento lógico). Cada posición equivale a dividir por 2 (truncando):

```python
20 >> 1 = 10  # 0b00010100 >> 1 = 0b00001010
20 >> 2 = 5   # 0b00010100 >> 2 = 0b00000101
```

Las operaciones de desplazamiento son significativamente más rápidas que la multiplicación y división aritmética, por lo que se utilizan ampliamente en optimización de código y en operaciones de bajo nivel como el procesamiento de imágenes y la criptografía.

## Cómo hace una suma una computadora

La suma binaria es la operación aritmética más fundamental de un procesador. La ALU (*Arithmetic Logic Unit*) realiza sumas binarias bit a bit, desde el bit menos significativo hasta el más significativo, propagando acarreos (*carry*) cuando la suma de dos bits supera 1:

```
  0110 (6)
+ 0101 (5)
------
  1011 (11)
```

Para sumar un bit: 0+0=0, 0+1=1, 1+0=1, 1+1=0 con carry 1. Esta operación elemental se implementa en un circuito llamado *full adder*, que toma tres entradas (dos bits a sumar y un carry de entrada) y produce dos salidas (el resultado y el carry de salida).

Los procesadores modernos utilizan adders de propagación de acarreos de 64 bits que pueden sumar dos enteros de 64 bits en un solo ciclo de reloj. Para sumas de alta precisión (cientos o miles de bits), se utilizan algoritmos como el algoritmo de Karatsuba o multiplicación por convolución de Fourier rápida (FFT), que reducen la complejidad computacional.

> La multiplicación en hardware se implementa como sumas repetidas o, más eficientemente, mediante árboles de adders que combinan parcialmente los productos intermedios en paralelo. Los procesadores modernos incluyen multiplicadores dedicados en la ALU que pueden ejecutar multiplicaciones de 64×64 bits en pocos ciclos de reloj.

