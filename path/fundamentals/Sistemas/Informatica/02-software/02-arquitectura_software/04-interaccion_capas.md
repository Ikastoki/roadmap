# Interacción entre capas

La interacción entre las capas de un sistema informático se produce mediante interfaces bien definidas que permiten la comunicación y el intercambio de datos entre componentes de diferentes niveles de abstracción. Estas interfaces —APIs, llamadas al sistema y protocolos— son los contratos que definen cómo una capa solicita servicios de otra, garantizando que los cambios en la implementación interna de una capa no afecten a las capas que dependen de ella.

Comprender cómo interactúan las capas es fundamental para diagnosticar problemas, optimizar rendimiento y diseñar sistemas que funcionen correctamente. Un error en la interfaz entre dos capas puede causar desde comportamientos inesperados hasta fallos catastróficos del sistema.

## Definición y función de las APIs

Una API (*Application Programming Interface*) es un conjunto de definiciones, protocolos y herramientas que permite la comunicación entre componentes de software. Las APIs definen los métodos, datos y formatos que una parte del software puede utilizar para solicitar servicios de otra.

Las APIs existen en múltiples niveles del sistema:

**APIs de hardware:** interfaz que los dispositivos exponen al sistema operativo a través de sus registros de control y estados. Un driver de tarjeta gráfica interactúa con la API de硬件 de la GPU para configurar modos de visualización, transferir texturas y sincronizar cuadros.

**APIs del sistema operativo:** interfaz que el kernel proporciona a las aplicaciones para acceder a los servicios del sistema. Las APIs POSIX, Win32 y las Linux syscall definen las operaciones disponibles para crear procesos, gestionar memoria, leer archivos y comunicarse por red.

**APIs de bibliotecas:** interfaz que las bibliotecas de software exponen a los programas que las utilizan. Una biblioteca de análisis numérico puede exponer una API con funciones como `suma()`, `multiplica()` y `invierte()`.

**APIs web:** interfaz que los servicios en la nube exponen a los clientes a través de HTTP. Una API REST define endpoints que aceptan y devuelven datos en formato JSON o XML, permitiendo la comunicación entre sistemas distribuidos.

> Las APIs bien diseñadas son estables, documentadas y coherentes. Una API que cambia frecuentemente rompe la compatibilidad con el software que la utiliza, causando fallos y frustración. Los proveedores de APIs invierten significativamente en diseño y versionado para mantener la estabilidad a largo plazo.

## Las System Calls (Llamadas al Sistema)

Las llamadas al sistema (*system calls*) son el mecanismo fundamental mediante el cual las aplicaciones de usuario solicitan servicios al kernel. Cuando un programa necesita realizar una operación que requiere privilegios de modo núcleo —como abrir un archivo, crear un proceso o enviar datos por la red— ejecuta una llamada al sistema que transfiere el control al kernel, que valida la solicitud y la ejecuta en nombre del programa.

Las llamadas al sistema se clasifican en varias categorías:

**Gestión de procesos:** `fork()` (crear un proceso hijo), `exec()` (ejecutar un programa), `exit()` (terminar un proceso), `wait()` (esperar a que un proceso hijo termine).

**Gestión de archivos:** `open()` (abrir un archivo), `read()` (leer datos), `write()` (escribir datos), `close()` (cerrar un archivo), `stat()` (obtener información del archivo).

**Gestión de memoria:** `mmap()` (mapear memoria), `brk()` (ajustar el tamaño del heap), `munmap()` (liberar memoria mapeada).

**Red:** `socket()` (crear un socket), `bind()` (asociar un socket a una dirección), `listen()` (escuchar conexiones), `accept()` (aceptar una conexión), `send()`/`recv()` (enviar/recibir datos).

**Gestión de dispositivos:** `ioctl()` (controlar dispositivos), `read()`/`write()` sobre dispositivos especiales.

El coste de una llamada al sistema incluye la transición de modo usuario a modo núcleo, la validación de parámetros, la ejecución de la operación y la transición de vuelta. Este overhead justifica la existencia de técnicas como el *buffering* y el *caching*, que reducen el número de llamadas al sistema agrupando múltiples operaciones en una sola.

## Ejemplos de APIs comunes (POSIX)

POSIX (*Portable Operating System Interface*) es un conjunto de estándares que define una API compatible entre sistemas operativos tipo Unix. POSIX garantiza que un programa escrito para un sistema POSIX (Linux, macOS, FreeBSD) pueda compilarse y ejecutarse en otro sin modificaciones significativas.

Las APIs POSIX más utilizadas incluyen:

- **`<unistd.h>`:** operaciones básicas de E/S (`read`, `write`, `open`, `close`), gestión de procesos (`fork`, `exec`, `_exit`) y otras utilidades del sistema.
- **`<sys/socket.h>`:** API de sockets para comunicación de red, incluyendo TCP/IP y Unix domain sockets.
- **`<pthread.h>`:** API de hilos POSIX para programación concurrente con múltiples hilos de ejecución.
- **`<stdio.h>`:** E/S estandar con buffer (`printf`, `scanf`, `fopen`, `fread`) que añade una capa de abstracción sobre las llamadas al sistema raw.

> POSIX no es una implementación sino una especificación. Cada sistema operativo implementa POSIX a su manera, manteniendo la compatibilidad con la interfaz estándar pero con diferencias en el rendimiento y las funcionalidades adicionales. Linux, por ejemplo, implementa POSIX de forma casi completa, mientras que Windows la soporta parcialmente a través del subsistema Windows Subsystem for Linux (WSL).

