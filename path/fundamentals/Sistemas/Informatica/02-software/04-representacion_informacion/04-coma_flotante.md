# Representación de números reales (coma flotante)

Los números reales —aquellos que incluyen fracciones decimales como 3.14, -0.001 o 6.022×10^23— requieren una representación especial en la computación, ya que el hardware solo opera nativamente con enteros en formato binario. La representación de coma flotante (*floating point*) resuelve este problema utilizando una notación científica binaria que permite representar un rango extremadamente amplio de valores, desde números increíblemente pequeños hasta enormemente grandes, con una precisión que depende del número de bits utilizados.

La coma flotante es fundamental para la ciencia, la ingeniería, las finanzas y prácticamente cualquier dominio que requiera cálculos con números no enteros. Sin embargo, su representación introduce matices que los programadores deben comprender para evitar errores sutiles que pueden tener consecuencias significativas.

## El estándar IEEE 754

El estándar IEEE 754, publicado por primera vez en 1985 y actualizado por última vez en 2019, define cómo se representan los números de coma flotante en prácticamente todos los sistemas informáticos modernos. El estándar especifica formatos de precisión simple (32 bits), doble (64 bits) y extendida, así como reglas para operaciones aritméticas, redondeo y manejo de condiciones especiales.

Un número IEEE 754 se compone de tres campos:

**Signo (1 bit):** indica si el número es positivo (0) o negativo (1).

**Exponente (8 bits en precisión simple, 11 bits en doble):** representa el exponente de la potencia de 2, con un sesgo (*bias*) que permite representar tanto exponentes positivos como negativos sin utilizar un bit de signo adicional. En precisión simple, el sesgo es 127, por lo que un exponente almacenado como 150 representa 150 - 127 = 23.

**Mantisa (23 bits en precisión simple, 52 bits en doble):** representa los dígitos significativos del número (la *mantisa* o *significand*). Se asume implícitamente un 1 antes del punto decimal, lo que proporciona un bit adicional de precisión sin coste de almacenamiento.

```
S | Exponente (8 bits) | Mantisa (23 bits)
1 |     10000010       | 10010010000000000000000
```

Este formato permite representar valores desde aproximadamente ±1.18×10^-38 hasta ±3.4×10^38 en precisión simple, y desde ±2.23×10^-308 hasta ±1.80×10^308 en precisión doble.

## Mantisa, exponente y signo

La mantisa determina la *precisión* del número: cuántos dígitos significativos se pueden representar con exactitud. En precisión simple, la mantisa de 23 bits proporciona aproximadamente 7 dígitos decimales de precisión. En precisión doble, los 52 bits de mantisa proporcionan aproximadamente 15-16 dígitos decimales de precisión.

El exponente determina el *rango* del número: qué tan grande o pequeño puede ser. Un exponente mayor permite representar números más grandes, mientras que uno menor permite números más pequeños. La combinación de mantisa y exponente permite representar valores que van desde el orden de los átomos hasta el de las galaxias con el mismo formato.

> El número 0.1 no se puede representar exactamente en binario de coma flotante, del mismo modo que 1/3 no se puede representar exactamente en decimal. Este hecho aparentemente trivial es la fuente de innumerables errores en cálculos financieros y científicos. La acumulación de errores de redondeo en cálculos repetidos puede producir resultados significativamente diferentes de los esperados.

## Precisión simple y doble

**Precisión simple (*float*, 32 bits):** proporciona suficiente precisión para la mayoría de cálculos gráficos, simulaciones físicas y aplicaciones donde la exactitud extrema no es crítica. Es el formato nativo de muchas GPUs y se utiliza en aplicaciones donde el rendimiento y el consumo de memoria son prioritarios.

**Precisión doble (*double*, 64 bits):** proporciona el doble de precisión y es el formato predeterminado en la mayoría de lenguajes de programación para números de coma flotante. Es el estándar en cálculos científicos y financieros donde la precisión es crítica.

**Precisión extendida (*long double*, 80-128 bits):** utilizada en aplicaciones que requieren precisión excepcional, como cálculos astronómicos o modelado financiero de alta frecuencia.

## Limitaciones y errores de redondeo

Los errores de redondeo son inherentes a la representación de coma flotante y surgen porque la mayoría de los números reales no se pueden representar exactamente con un número finito de bits. Cuando un número no representable se aproxima al valor más cercano representable, se produce un error de redondeo.

Estos errores se acumulan en cálculos sucesivos, potencialmente produciendo resultados significativamente diferentes de los esperados. Un ejemplo clásico es la sucesión:

```python
x = 1.0
for i in range(10):
    x = x - 0.1
# Resultado esperado: 0.0
# Resultado real: 6.938893903907228e-18 (aprox. 7×10^-18)
```

El problema no es un error del programador sino una limitación fundamental de la representación: 0.1 no tiene una representación exacta en binario de coma flotante, por lo que cada resta introduce un pequeño error que se acumula.

Para cálculos financieros donde la precisión decimal es crítica, se utilizan formatos alternativos como `decimal` en Python o `BigDecimal` en Java, que representan números decimales de forma exacta a costa de mayor consumo de memoria y menor rendimiento.

