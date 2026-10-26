# Sistemas de numeración

Los sistemas de numeración son marcos matemáticos que utilizan un conjunto de símbolos y reglas para representar cantidades numéricas. La computación se fundamenta en el sistema binario, pero los humanos utilizamos cotidianamente el sistema decimal, y los programadores frecuentemente emplean el hexadecimal y el octal. La capacidad de convertir entre estos sistemas es una habilidad fundamental para comprender cómo los ordenadores representan y procesan datos.

La comprensión de los sistemas de numeración no es solo un ejercicio teórico: tiene implicaciones prácticas directas en la programación, la depuración, el análisis de protocolos de red y la interpretación de valores de hardware. Un programador que entiende el sistema hexadecimal puede interpretar direcciones de memoria, valores de registros y códigos de color de forma mucho más intuitiva que uno que solo conoce el decimal.

## La base de todo: Binario (Base 2)

El sistema binario utiliza únicamente dos cifras: 0 y 1. Cada posición de un número binario representa una potencia de 2, comenzando por la posición más a la derecha (2^0 = 1):

```
1101₂ = 1×2³ + 1×2² + 0×2¹ + 1×2⁰ = 8 + 4 + 0 + 1 = 13₁₀
```

El binario es el lenguaje nativo de la computadora porque los circuitos electrónicos pueden representar dos estados de forma fiable y barata: encendido/apagado, voltaje alto/bajo, carga/no carga. Cada transistor en un procesador es esencialmente un interruptor binario, y miles de millones de estos interruptores trabajando juntos permiten realizar cálculos complejos.

Aunque el binario es el sistema fundamental de la computadora, es extremadamente verboso para los humanos: el número decimal 255 se escribe como `11111111` en binario. Esta verbosidad motivó la adopción de sistemas de numeración alternativos que son más compactos pero mantienen una correspondencia directa con el binario.

## Conversión entre sistemas

### Binario a Decimal

Para convertir un número binario a decimal, se suman las potencias de 2 correspondientes a las posiciones que tienen un 1:

```
10110₂ = 1×16 + 0×8 + 1×4 + 1×2 + 0×1 = 22₁₀
```

### Decimal a Binario

La conversión de decimal a binario se realiza mediante divisiones sucesivas por 2, registrando los restos de abajo hacia arriba:

```
22 ÷ 2 = 11  resto 0
11 ÷ 2 = 5   resto 1
5  ÷ 2 = 2   resto 1
2  ÷ 2 = 1   resto 0
1  ÷ 2 = 0   resto 1
```
Resultado: 10110₂

### Binario a Hexadecimal

Cada dígito hexadecimal corresponde exactamente a 4 bits, lo que hace la conversión directa:

```
1011 0110₂ = B6₁₆
1011 = 11 = B en hexadecimal
0110 = 6 en hexadecimal
```

### Decimal a Hexadecimal

Se realiza mediante divisiones sucesivas por 16:

```
255 ÷ 16 = 15  resto 15 = F
15  ÷ 16 = 0   resto 15 = F
```
Resultado: FF₁₆

## La importancia del Hexadecimal en la programación

El hexadecimal (base 16) es el sistema de numeración más utilizado en programación y arquitectura de computadoras por varias razones:

**Compactacidad:** un dígito hexadecimal representa exactamente 4 bits, lo que permite expresar un byte completo con solo 2 dígitos. El valor `0xFF` es mucho más legible que `11111111` y más compacto que `255`.

**Correspondencia directa con bits:** la conversión entre hexadecimal y binario es trivial, ya que cada dígito hex se mapea directamente a 4 bits. Esto hace que el hexadecimal sea ideal para inspeccionar y manipular bits individuales.

**Direcciones de memoria:** las direcciones de memoria se expresan habitualmente en hexadecimal, ya que reflejan la organización de la memoria en palabras alineadas a potencias de 2.

**Códigos de color:** los colores en formato RGB se expresan típicamente en hexadecimal (por ejemplo, `#FF5733` para un color naranja).

**Valores de hardware:** los registros de hardware, las configuraciones de firmware y los protocolos de comunicación utilizan hexadecimal como notación estándar.

> Hexadecimal y octal (base 8) son ambos sistemas que facilitan la manipulación de bits, pero hexadecimal ha prevalecido porque cada dígito hex equivale a 4 bits (un nibble), mientras que cada dígito octal equivale a 3 bits. Dado que los datos se organizan en múltiplos de 4 y 8 bits, hexadecimal es más natural.

