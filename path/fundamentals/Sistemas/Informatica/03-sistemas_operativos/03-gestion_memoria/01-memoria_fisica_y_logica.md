# Memoria física y lógica

La gestión de memoria es una de las funciones más críticas de un sistema operativo. La memoria RAM es un recurso finito y compartido entre todos los procesos que se ejecutan simultáneamente, y el sistema operativo debe garantizar que cada proceso tenga acceso al espacio de memoria que necesita sin interferir con los demás. Este aislamiento se logra a través de la distinción entre memoria física y memoria lógica.

La separación entre dirección física (la ubicación real en los módulos de RAM) y dirección lógica (la ubicación que un proceso "ve" en su espacio de direcciones) es el fundamento de la protección de memoria y la virtualización de recursos que permite a múltiples procesos coexistir en un mismo sistema sin corromperse mutuamente.

## Dirección física vs dirección lógica

La **dirección física** es la ubicación real en los módulos de memoria RAM, representada como un número que corresponde a una posición específica del bus de direcciones. Es la dirección que la unidad de gestión de memoria (MMU) envía al hardware de memoria.

La **dirección lógica** (o dirección virtual) es la dirección que genera el proceso en ejecución. Cada proceso opera en su propio espacio de direcciones lógicas, que comienza en 0 y se extiende hasta el tamaño máximo del proceso. El sistema operativo traduce las direcciones lógicas en direcciones físicas antes de que lleguen a la memoria RAM.

```
Proceso A          Proceso B
┌──────────┐      ┌──────────┐
│ Lógica   │      │ Lógica   │
│ 0x0000   │      │ 0x0000   │
│   ↓      │      │   ↓      │
│ 0x1000   │      │ 0x1000   │
└────┬─────┘      └────┬─────┘
     │                  │
     └────┬─────────────┘
          ↓
     ┌──────────┐
     │   MMU    │
     └────┬─────┘
          ↓
     ┌──────────┐
     │ Física   │
     │ RAM      │
     └──────────┘
```

La MMU (*Memory Management Unit*) es el hardware responsable de la traducción. Opera con tablas de traducción que mapean páginas de memoria lógica a marcos de memoria física, realizando la conversión en cada acceso a memoria. Este proceso es transparente para el proceso, que solo ve su propio espacio de direcciones lineal y continuo.

## Segmentación

La segmentación es un esquema de gestión de memoria que divide el espacio de direcciones lógicas de un proceso en bloques de tamaño variable llamados segmentos. Cada segmento representa una entidad lógica del programa: código, datos, pila, etc.

Una dirección lógica en segmentación consiste en un par (segmento, desplazamiento), donde el segmento identifica la entidad lógica y el desplazamiento indica la posición dentro de ella. La tabla de segmentos almacena la dirección base y el límite de cada segmento, permitiendo al hardware verificar que el desplazamiento no exceda el tamaño del segmento.

La segmentación se adapta mejor a la estructura lógica del programa que la paginación, pero genera problemas de fragmentación externa: los segmentos de diferentes tamaños dejan huecos en la memoria que no se pueden reutilizar fácilmente.

## Paginación

La paginación divide tanto el espacio lógico como la memoria física en bloques de tamaño fijo: páginas (en el espacio lógico) y marcos (en la memoria física). La dirección lógica se divide en un número de página y un desplazamiento dentro de ella, y la tabla de páginas mapea cada número de página a un marco físico.

La paginación elimina la fragmentación externa, ya que cualquier página puede colocarse en cualquier marco disponible. Sin embargo, puede generar fragmentación interna si el tamaño de la página no es un divisor exacto del tamaño del proceso.

La paginación jerárquica organiza las tablas de páginas en múltiples niveles, reduciendo la memoria necesaria para almacenarlas. Un sistema de 64 bits con páginas de 4 KB requeriría una tabla de páginas enorme si fuera plana; la jerarquía la divide en tablas más pequeñas que se cargan bajo demanda.

## Unidad de gestión de memoria (MMU)

La MMU es el hardware que realiza la traducción de direcciones lógicas a físicas en cada acceso a memoria. Opera en nanosegundos, ya que cada instrucción puede requerir múltiples accesos a memoria (lectura de instrucción, lectura de operandos, escritura de resultados).

La MMU utiliza un *TLB* (*Translation Lookaside Buffer*) como caché de las traducciones más recientes. El TLB es una memoria asociativa de alta velocidad que almacena las últimas traducciones página-marcos. Cuando una traducción se encuentra en el TLB (*hit*), se evita el acceso a la tabla de páginas en memoria principal, acelerando significativamente la traducción.

> La gestión de memoria es un equilibrio entre velocidad (caché, TLB), capacidad (memoria principal) y coste (disco de intercambio). Los sistemas modernos utilizan múltiples niveles de caché de hardware (L1, L2, L3) y gestión de memoria por demanda del SO para optimizar este equilibrio.

