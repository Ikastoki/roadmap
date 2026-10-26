# Memoria virtual y paging

La memoria virtual es uno de los mecanismos más importantes y elegantes de los sistemas operativos modernos. Permite que cada proceso funcione como si dispusiera de un espacio de direcciones contiguo y exclusivo, ocultando la complejidad de la gestión de la memoria física compartida entre múltiples procesos. Sin memoria virtual, los programas tendrían que preocuparse por la ubicación física de sus datos en RAM, y un solo proceso podría agotar toda la memoria disponible del sistema.

La memoria virtual transforma la RAM en un recurso compartido y gestionado de forma transparente, permitiendo que el sistema operativo optimice el uso de la memoria, aísle los procesos entre sí y proporcione una ilusión de capacidad de memoria prácticamente ilimitada. El mecanismo central que hace posible esta abstracción es el *paging* (paginación).

## Memoria virtual

La memoria virtual es una capa de abstracción implementada por el sistema operativo que permite a cada proceso acceder a un espacio de direcciones lineal y continuo, independientemente de cómo se distribuyan esos datos en la memoria física real. Cada proceso ve un espacio de direcciones propio que comienza en una dirección baja (típicamente 0) y se extiende teóricamente hasta el límite arquitectónico (por ejemplo, 2^48 bytes en un sistema x86-64).

Esta ilusión se logra mediante una combinación de hardware (la unidad de gestión de memoria o MMU del procesador) y software (el sistema operativo). Cuando un proceso accede a una dirección de memoria virtual, la MMU traduce esa dirección a una dirección física en la RAM real antes de que la operación se complete. Si los datos no están en RAM (porque nunca se han cargado o fueron desplazados), se produce una falta de página (*page fault*) que el sistema operativo gestiona cargando los datos desde el almacenamiento.

El espacio de direcciones virtual de un proceso se divide en regiones con propósitos específicos: el segmento de código (donde se almacenan las instrucciones del programa), el segmento de datos (variables globales), el montículo (*heap*, memoria dinámica asignada con `malloc` o `new`) y la pila (*stack*, variables locales y llamadas a función). Cada una de estas regiones puede encontrarse en cualquier ubicación de la memoria física, y el proceso no percibe esta dispersión.

> La memoria virtual también proporciona aislamiento entre procesos. Un proceso no puede acceder a las direcciones virtuales de otro proceso, ya que la MMU no tiene configurada la traducción correspondiente. Este aislamiento es fundamental para la estabilidad y seguridad del sistema.

## Paging

El *paging* es la técnica de implementación de la memoria virtual más utilizada en los sistemas operativos modernos. Consiste en dividir tanto el espacio de direcciones virtuales como la memoria física en bloques de tamaño fijo: las páginas virtuales y los marcos de página respectivamente. Típicamente, cada página tiene un tamaño de 4 KB, aunque los sistemas modernos también soportan páginas de 2 MB y 1 GB (*huge pages* o *large pages*).

El sistema operativo mantiene una tabla de páginas para cada proceso, que establece la correspondencia entre cada página virtual y su marco de página en memoria física. Cuando un proceso accede a una dirección virtual, la MMU divide la dirección en número de página y desplazamiento (*offset*), consulta la tabla de páginas para encontrar el marco correspondiente y genera la dirección física resultante.

```
Dirección virtual: [Número de página | Desplazamiento]
         ↓                  ↓
       MMU         Tabla de páginas del proceso
         ↓
Dirección física: [Número de marco | Desplazamiento]
```

El desplazamiento se mantiene idéntico entre la dirección virtual y la física, ya que la traducción solo afecta al número de página. La MMU también almacena bits de control en cada entrada de la tabla de páginas: bit de validez (indica si la página está en RAM), bits de permisos (lectura, escritura, ejecución) y bit de modificado (*dirty bit*, indica si la página ha sido escrita desde su carga).

Cuando se produce una falta de página, el sistema operativo debe encontrar un marco libre en memoria física para albergar la página solicitada. Si no hay marcos disponibles, ejecuta un algoritmo de reemplazo de página para seleccionar una página víctima, escribirla en el almacenamiento si ha sido modificada (proceso de *swap out*) y cargar la página solicitada en su lugar (*swap in*).

## Archivo de paginación

El archivo de paginación (*page file* en Windows, *swap space* o *swap partition* en Linux) es una porción reservada del almacenamiento que el sistema operativo utiliza como extensión de la memoria RAM. Cuando la memoria física se llena, el sistema operativo mueve páginas menos utilizadas al archivo de paginación, liberando marcos de página en RAM para datos más activos.

En Windows, el archivo de paginación se denomina `pagefile.sys` y se configura desde las propiedades del sistema. En Linux, el *swap* puede ser una partición dedicada o un archivo de intercambio (`swapfile`). La gestión del swap varía según el sistema operativo, pero el principio es相同的: cuando un proceso accede a una página que ha sido desplazada al almacenamiento, se produce una falta de página que obliga al sistema operativo a cargarla de vuelta a RAM.

El tamaño del archivo de paginación es un parámetro importante pero frecuentemente mal comprendido. Un tamaño excesivo desperdicia espacio de almacenamiento y puede degradar el rendimiento si el sistema comienza a usar excesivamente el swap (*thrashing*). Un tamaño insuficiente puede provocar que el sistema no pueda asignar memoria cuando la RAM se agota, provocando fallos en aplicaciones. Una regla general es configurar el archivo de paginación entre 1.5 y 2 veces la cantidad de RAM instalada, aunque las necesidades reales dependen del uso específico del sistema.

> El *thrashing* es una situación donde el sistema dedica más tiempo a mover páginas entre RAM y almacenamiento que a ejecutar trabajo útil. Se produce cuando la cantidad de memoria activa requerida por los procesos supera significativamente la RAM disponible, forzando un intercambio constante de páginas. La solución inmediata es reducir el número de procesos activos o instalar más RAM.

