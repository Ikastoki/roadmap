# Paginación y segmentación

La paginación y la segmentación son los dos mecanismos fundamentales de gestión de memoria que permiten al sistema operativo proporcionar a cada proceso un espacio de direcciones aislado y protegido. Ambos resuelven el problema de asignar memoria de forma eficiente, pero lo hacen de maneras diferentes con ventajas y desventajas distintas.

La paginación es el mecanismo predominante en los sistemas modernos, pero muchos sistemas combinan ambos enfoques en un esquema conocido como segmentación con paginación, que intenta aprovechar lo mejor de ambos mundos.

## Paginación detallada

La paginación divide el espacio de direcciones lógicas en bloques de tamaño fijo llamados páginas (típicamente 4 KB, aunque también se usan 2 KB y 16 KB), y la memoria física en bloques del mismo tamaño llamados marcos o frames. Cada página se mapea a un marco físico a través de la tabla de páginas del proceso.

La dirección lógica se compone de dos campos:

- **Número de página (*p*):** índice en la tabla de páginas que identifica qué bloque lógico se necesita.
- **Desplazamiento (*d*):** posición dentro de la página (0 hasta el tamaño de página - 1).

La traducción se realiza así:

```
Dirección lógica: [p | d]
                   ↓
Tabla de páginas: [p] → marco f (f)
                   ↓
Dirección física: [f | d]
```

El desplazamiento no se modifica durante la traducción, ya que la página y el marco tienen el mismo tamaño.

**Ventajas de la paginación:**

- Elimina la fragmentación externa: cualquier página puede ocupar cualquier marco.
- La gestión es simple: las tablas de páginas son estructuras de datos lineales.
- Facilita el intercambio (*swap*): las páginas se mueven individualmente al disco.

**Desventajas de la paginación:**

- Fragmentación interna: si un proceso ocupa N páginas y N no es entero, la última página queda parcialmente llena.
- Las tablas de páginas pueden consumir mucha memoria si el espacio de direcciones es grande.

## Tablas de páginas multinivel

Para direcciones de 64 bits, una tabla de páginas plana sería enorme e impráctica. La solución es la paginación jerárquica: organizar la tabla de páginas en múltiples niveles donde cada nivel es un índice.

En un sistema de 4 niveles (x86-64), la dirección virtual se divide en 4 campos de 9 bits cada uno (más 12 bits de desplazamiento para páginas de 4 KB), formando un árbol de 4 niveles. Cada nivel apunta a una tabla del siguiente nivel, y solo se cargan en memoria las ramas del árbol que se están usando. Si un proceso solo utiliza una pequeña parte de su espacio de direcciones, solo se necesitan unas pocas tablas de un nivel para representar sus traducciones.

## Page fault (fallo de página)

Un *page fault* ocurre cuando un proceso accede a una página que no está cargada en memoria física (su bit de presencia en la tabla de páginas está en 0). La MMU genera una interrupción que el manejador de page faults del SO procesa:

1. El proceso se suspende temporalmente.
2. El SO verifica que la dirección sea válida (dentro del espacio lógico del proceso).
3. Se busca la página en el disco (en el área de intercambio o en el archivo del ejecutable).
4. Se asigna un marco libre en la memoria física.
5. Se carga la página del disco al marco asignado.
6. Se actualiza la tabla de páginas con el nuevo marco y el bit de presencia.
7. Se reanuda la ejecución del proceso desde la instrucción que causó el fallo.

Los page faults son costosos (millones de ciclos de CPU frente a nanosegundos para un acceso a RAM), por lo que los sistemas operativos utilizan estrategias como el *prefetching* (cargar páginas adyacentes antes de que se necesiten) para minimizarlos.

## Segmentación

La segmentación divide el espacio de direcciones en bloques de tamaño variable llamados segmentos, cada uno representando una entidad lógica del programa: código de ejecución, datos inicializados, datos no inicializados (BSS), pila, montículo.

Una dirección lógica en segmentación consiste en un par (segmento, desplazamiento), y la tabla de segmentos almacena para cada segmento:

- **Base:** la dirección física de inicio del segmento.
- **Límite:** el tamaño del segmento.

El hardware verifica que el desplazamiento sea menor que el límite antes de sumarlo a la base para obtener la dirección física. Esta verificación proporciona protección automática entre segmentos.

## Fragmentación interna y externa

**Fragmentación interna:** ocurre cuando se asigna más memoria de la necesaria porque el bloque asignado es más grande que lo solicitado. En paginación, la última página de un proceso puede no estar completamente llena. Por ejemplo, si un proceso ocupa 10.000 bytes y las páginas son de 4 KB, necesita 3 páginas (12.288 bytes), desperdiciando 2.288 bytes.

**Fragmentación externa:** ocurre cuando hay suficiente memoria total disponible pero no está contigua en un bloque único. Es un problema de la segmentación pura y de la asignación contigua, pero no de la paginación, donde cualquier página puede ocupar cualquier marco.

> La paginación es el mecanismo predominante en los sistemas modernos porque su simplicidad y eficiencia superan sus desventajas. La segmentación se utiliza principalmente como complemento (segmentación con paginación) o en sistemas especializados donde la protección por tipo de datos es importante.

