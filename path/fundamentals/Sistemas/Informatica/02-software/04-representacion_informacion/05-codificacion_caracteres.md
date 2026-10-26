# Codificación de caracteres

La codificación de caracteres es el sistema que define cómo se representan los símbolos de texto (letras, números, puntuación, caracteres especiales) como secuencias de bits en una computadora. Sin una codificación estándar, el mismo byte podría interpretarse como una letra diferente en sistemas distintos, causando corrupción de datos e incompatibilidad entre aplicaciones. La evolución de las codificaciones de caracteres refleja la creciente globalización de la informática y la necesidad de representar los sistemas de escritura de todas las lenguas del mundo.

## ASCII

ASCII (*American Standard Code for Information Interchange*), publicado originalmente en 1963, fue la primera codificación de caracteres estandarizada ampliamente adoptada. Asigna un número único (del 0 al 127) a cada uno de 128 caracteres, incluyendo letras mayúsculas y minúsculas del alfabeto inglés, dígitos, puntuación y caracteres de control.

La tabla ASCII organiza los caracteres en bloques lógicos:

- **0-31:** caracteres de control (tabulación, salto de línea, retorno de carro).
- **32:** espacio en blanco.
- **33-47:** signos de puntuación y símbolos (!, ", #, $, etc.).
- **48-57:** dígitos del 0 al 9.
- **58-64:** caracteres especiales (:, ;, <, =, >, ?, @).
- **65-90:** letras mayúsculas (A-Z).
- **91-96:** corchetes y otros símbolos ([, \, ], ^, _, `).
- **97-122:** letras minúsculas (a-z).
- **123-127:** llaves y otros símbolos ({, |, }, ~, DEL).

ASCII utiliza exactamente 7 bits por carácter, pero se almacenaba tradicionalmente en bytes de 8 bits, dejando el bit más significativo sin usar (o reservado para paridad en transmisiones).

## Limitaciones

La principal limitación de ASCII es su capacidad: 128 caracteres son suficientes para el inglés pero completamente inadecuados para otros idiomas. Los alfabetos francés, español, alemán y portugués utilizan caracteres con tildes, diéresis y otros signos diacríticos que ASCII no incluye.

Para solucionar esto, se crearon extensiones de 8 bits como ISO-8859-1 (Latin-1), que utiliza el bit adicional para ampliar la tabla a 256 caracteres, incluyendo caracteres acentuados y símbolos de idiomas europeos. Sin embargo, incluso 256 caracteres son insuficientes para representar los miles de caracteres de idiomas como chino, japonés, coreano o árabe, que utilizan sistemas de escritura completamente diferentes.

> El problema fundamental de las codificaciones de 8 bits es que son específicas de una región o familia de idiomas. Un archivo codificado en ISO-8859-1 se mostrará correctamente en español pero producirá caracteres ilegibles en chino. Esta fragmentación motivó la creación de un estándar universal.

## Unicode y UTF-8: El estándar universal

Unicode es el estándar internacional que asigna un número único (*code point*) a cada carácter de todos los sistemas de escritura del mundo, así como a símbolos técnicos, matemáticos y de puntuación. Actualmente, Unicode define más de 149,000 caracteres que cubren 159 sistemas de escritura.

Unicode puede implementarse mediante diferentes codificaciones que definen cómo se traducen los code points a secuencias de bytes:

**UTF-8:** la codificación más utilizada en la actualidad, adoptada por más del 98% de los sitios web. UTF-8 es una codificación de longitud variable que utiliza de 1 a 4 bytes por carácter:

- 1 byte (0-127): caracteres ASCII (compatibilidad total con ASCII original).
- 2 bytes (128-2047): caracteres latinos extendidos, griego, cirílico, árabe.
- 3 bytes (2048-65,535): caracteres CJK (chino, japonés, coreano), símbolos matemáticos.
- 4 bytes (65,536-1,114,111): caracteres raros, emojis, escrituras históricas.

**UTF-16:** utiliza 2 bytes por carácter para los 65,536 primeros code points y 4 bytes para el resto. Es utilizado internamente por Windows y Java, pero es menos eficiente que UTF-8 para textos predominantemente en alfabeto latino.

**UTF-32:** utiliza exactamente 4 bytes por carácter, proporcionando acceso directo a cualquier carácter por posición. Es simple pero ineficiente en espacio, ya que cada carácter ASCII ocupa 4 bytes en lugar de 1.

La eficiencia de UTF-8 radica en su compatibilidad con ASCII y en su capacidad para representar caracteres frecuentes con menos bytes que UTF-16 o UTF-32. Un documento en inglés puro codificado en UTF-8 tiene exactamente el mismo tamaño que si estuviera codificado en ASCII, mientras que los caracteres de otros idiomas se representan con el número mínimo de bytes necesario.

