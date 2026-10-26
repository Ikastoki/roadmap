# Arquitectura del sistema operativo

La arquitectura de un sistema operativo define cómo se organizan sus componentes, cómo interactúan entre sí y con el hardware, y qué nivel de abstracción proporcionan a las aplicaciones. Comprender la arquitectura del SO es fundamental para diagnosticar problemas de rendimiento, diseñar aplicaciones eficientes y administrar sistemas de forma efectiva.

## Modo usuario vs modo kernel

Los procesadores modernos implementan al menos dos niveles de privilegio que determinan qué operaciones puede ejecutar el software:

**Modo núcleo (*kernel mode*):** el nivel de privilegio más alto, donde el software tiene acceso irrestricto a todo el hardware y a todas las instrucciones del procesador. Solo el kernel y los drivers ejecutan en modo núcleo, con acceso directo a registros de hardware, tablas de páginas y dispositivos de E/S.

**Modo usuario (*user mode*):** el nivel de privilegio bajo, donde las aplicaciones ejecutan con restricciones. En modo usuario, el software no puede acceder directamente a hardware, modificar tablas de páginas o ejecutar instrucciones privilegiadas. Cuando una aplicación necesita realizar una operación que requiere modo núcleo, ejecuta una llamada al sistema que transfiere el control al kernel.

Esta separación es un mecanismo de seguridad fundamental: un error en una aplicación no puede corromper el kernel ni afectar a otras aplicaciones. El kernel actúa como árbitro que valida y ejecuta las solicitudes de las aplicaciones.

> La transición entre modo usuario y modo núcleo tiene un coste de rendimiento (típicamente 100-1000 ciclos de reloj), por lo que los diseñadores de SO buscan minimizar el número de transiciones necesarias para operaciones comunes. Las tablas de páginas y los buffers de E/S reducen la frecuencia de llamadas al sistema.

## Llamadas al sistema (*system calls*)

Las llamadas al sistema son el mecanismo mediante el cual las aplicaciones solicitan servicios al kernel. Son la interfaz programática entre el espacio de usuario y el espacio de núcleo.

Las categorías principales de llamadas al sistema incluyen:

**Proceso:** `fork()`, `exec()`, `exit()`, `wait()` — crear, ejecutar y terminar procesos.
**Archivos:** `open()`, `read()`, `write()`, `close()` — operaciones de E/S sobre archivos.
**Memoria:** `mmap()`, `brk()`, `munmap()` — gestión de memoria virtual.
**Red:** `socket()`, `bind()`, `listen()`, `accept()`, `send()`, `recv()` — comunicación de red.
**Dispositivos:** `ioctl()`, `read()`, `write()` sobre dispositivos especiales.

Cada llamada al sistema implica una transición de modo usuario a modo núcleo, la validación de parámetros, la ejecución de la operación y la transición de vuelta. El overhead de esta transición justifica técnicas como el buffering y el caching que agrupan múltiples operaciones en una sola llamada.

## Capas de abstracción

La arquitectura de capas del SO se organiza jerárquicamente:

```
┌───────────────────────────────────┐
│         Aplicaciones              │
├───────────────────────────────────┤
│   Bibliotecas del sistema (libc)  │
├───────────────────────────────────┤
│      Interfaz de llamadas al      │
│            sistema                │
├───────────────────────────────────┤
│            Kernel                 │
│  ┌─────────┬──────────┬────────┐ │
│  │ Gestión │ Gestión  │Gestión │ │
│  │procesos │ memoria  │ archivos│ │
│  ├─────────┼──────────┼────────┤ │
│  │Gestión  │  Redes   │Dispos. │ │
│  │disposit.│          │  E/S   │ │
│  └─────────┴──────────┴────────┘ │
├───────────────────────────────────┤
│           Hardware                │
└───────────────────────────────────┘
```

Cada capa se comunica solo con la capa inmediata superior e inferior a través de interfaces bien definidas. Esta modularidad permite cambiar componentes sin afectar al resto del sistema: se puede reemplazar el driver de un disco sin modificar el sistema de archivos ni las aplicaciones.

## Exokernel

El exokernel es una arquitectura de SO experimental que minimiza la abstracción, permitiendo a las aplicaciones acceder directamente al hardware con el mínimo intermediarismo posible. A diferencia de los micronúcleos que encapsulan los mecanismos de hardware detrás de interfaces de alto nivel, los exokernels exponen los recursos de hardware de forma segura pero directa.

La idea es que las aplicaciones pueden implementar sus propias abstracciones (gestión de memoria, E/S) optimizadas para sus necesidades específicas, sin pagar el overhead de abstracciones genéricas que pueden no ser óptimas para su caso de uso. Los exokernels (MIT Exokernel, Nemesis) han influido en el diseño de sistemas operativos de alto rendimiento, aunque su adopción comercial ha sido limitada.

