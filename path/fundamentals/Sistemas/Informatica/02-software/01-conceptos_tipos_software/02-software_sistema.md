# Software de sistema

El software de sistema es la capa de software que se sitúa directamente sobre el hardware, proporcionando las funcionalidades básicas que permiten el funcionamiento del equipo y sirven de base para todo el software posterior. Incluye el sistema operativo, los drivers de dispositivos, el firmware y las utilidades de diagnóstico, todos ellos componentes que trabajan en un segundo plano invisible para el usuario pero absolutamente esenciales para el correcto funcionamiento del sistema.

Sin software de sistema, un ordenador no podría arrancar, reconocer sus componentes ni ejecutar ningún programa. Es la primera capa de software que se carga al encender el equipo y permanece activa durante toda la sesión de uso, gestionando los recursos del hardware y proporcionando los servicios sobre los cuales se construyen todas las demás capas de software.

## El papel del sistema operativo

El sistema operativo (*Operating System*, OS) es el componente central del software de sistema. Su función principal es gestionar los recursos del hardware —CPU, memoria, almacenamiento, dispositivos de E/S— y ponerlos a disposición de las aplicaciones a través de interfaces estandarizadas.

Los sistemas operativos más utilizados son:

**Windows:** el sistema operativo de escritorio más extendido del mundo, desarrollado por Microsoft. Ofrece la mayor compatibilidad con software de consumo y gaming, una interfaz gráfica pulida y una ampliabase de hardware soportado. Sus versiones más recientes (Windows 10, Windows 11) incorporan funcionalidades de seguridad avanzadas, integración con servicios en la nube y soporte para aplicaciones Android.

**Linux:** un sistema operativo de código abierto basado en el kernel Unix, disponible en cientos de distribuciones (Ubuntu, Fedora, Debian, Arch, etc.). Es el sistema dominante en servidores, supercomputadores y dispositivos embebidos. Su flexibilidad, seguridad y coste cero lo hacen ideal tanto para entusiastas como para empresas.

**macOS:** el sistema operativo de Apple para sus ordenadores Mac. Basado en un kernel BSD Unix (Darwin), ofrece una integración estrecha con el ecosistema de hardware Apple, una interfaz gráfica refinada y herramientas de productividad y creatividad reconocidas.

> La elección del sistema operativo determina en gran medida qué software puede ejecutarse en el equipo, cómo se gestionan los recursos y qué nivel de personalización y control tiene el usuario. Cada sistema operativo tiene fortalezas y debilidades que lo hacen más adecuado para escenarios específicos.

## Tipos de software de soporte

### Drivers

Los drivers (*device drivers*) son programas especializados que permiten al sistema operativo comunicarse con hardware específico. Cada dispositivo que se conecta a la computadora —tarjeta gráfica, adaptador de red, impresora, ratón— requiere un driver que implemente las instrucciones específicas para controlar ese dispositivo.

Los drivers se sitúan en la frontera entre el software y el hardware, traduciendo las operaciones genéricas del sistema operativo en comandos específicos que el dispositivo puede entender. Un driver de tarjeta gráfica, por ejemplo, traduce las llamadas de la API gráfica (DirectX, Vulkan, OpenGL) en instrucciones que el procesador gráfico específico puede ejecutar.

La calidad y actualización de los drivers afecta directamente al rendimiento y la estabilidad del sistema. Drivers desactualizados o defectuosos pueden causar bloqueos, errores gráficos, incompatibilidades y vulnerabilidades de seguridad.

### Firmware

El firmware es un tipo de software de sistema embebido en un componente de hardware que controla su funcionamiento básico. A diferencia de los drivers, que residen en el almacenamiento del sistema y se cargan por el sistema operativo, el firmware se almacena en una memoria no volátil del propio componente y se ejecuta directamente.

Los ejemplos más comunes de firmware son:

- **BIOS/UEFI:** firmware de la placa base que inicializa el hardware durante el arranque.
- **Firmware de SSD:** gestiona las operaciones de lectura/escritura, el wear leveling y la corrección de errores.
- **Firmware de GPU:** controla los modos de energía, las velocidades de reloj y la gestión térmica de la tarjeta gráfica.
- **Firmware de router:** gestiona el enrutamiento de red, las funciones de seguridad y la interfaz de administración.

### Utilidades de diagnóstico

Las utilidades de diagnóstico son herramientas de software diseñadas para analizar, verificar y reparar componentes del sistema. Incluyen:

- **Herramientas de monitorización de hardware:** como HWMonitor, CPU-Z o GPU-Z, que muestran temperaturas, voltajes, velocidades de rotación y especificaciones de los componentes.
- **Herramientas de diagnóstico de memoria:** como MemTest86, que verifican la integridad de la memoria RAM ejecutando patrones de prueba exhaustivos.
- **Herramientas de diagnóstico de disco:** como CrystalDiskInfo, que reportan el estado de salud de los discos duros y SSD mediante datos S.M.A.R.T.
- **Utilidades de reparación:** como `chkdsk` en Windows o `fsck` en Linux, que verifican y reparan errores en los sistemas de archivos.

