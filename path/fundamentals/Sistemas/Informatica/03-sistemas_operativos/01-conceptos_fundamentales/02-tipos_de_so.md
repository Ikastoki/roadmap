# Tipos de sistema operativo

Los sistemas operativos se clasifican según múltiples criterios que reflejan sus diferentes propósitos, restricciones de recursos y modelos de interacción. La elección del tipo de SO adecuado depende del hardware disponible, los requisitos de rendimiento, las necesidades de seguridad y el contexto de uso previsto.

## SO de servidor vs SO de escritorio vs SO embebido

**SO de servidor:** están diseñados para gestionar peticiones de múltiples usuarios simultáneamente, priorizando la disponibilidad, el rendimiento bajo carga y la gestión de recursos. Los servidores suelen ejecutarse sin interfaz gráfica (solo línea de comandos) para maximizar los recursos disponibles para las aplicaciones. Ejemplos: Ubuntu Server, CentOS Stream, Windows Server, FreeBSD.

Las características distintivas incluyen: gestión avanzada de usuarios y permisos, soporte para servicios de red (web, correo, bases de datos), capacidades de clustering y alta disponibilidad, y herramientas de monitoreo y administración remota.

**SO de escritorio:** están diseñados para un usuario individual que interactúa directamente con el sistema mediante una interfaz gráfica. Priorizan la facilidad de uso, la experiencia del usuario y la compatibilidad con aplicaciones de consumo. Ejemplos: Windows 11, macOS Sonoma, Ubuntu Desktop, Fedora Workstation.

Los SO de escritorio incluyen entornos de escritorio completos (GNOME, KDE Plasma, Explorer) con gestores de archivos, navegadores web, suites de oficina y utilidades de configuración integradas.

**SO embebido:** están diseñados para ejecutarse en dispositivos con recursos limitados donde el software no es el producto principal sino un componente de un sistema mayor. Priorizan el arranque rápido, el consumo mínimo de recursos y la fiabilidad en operación continua. Ejemplos: FreeRTOS, Zephyr, embedded Linux, Windows IoT.

Los dispositivos embebidos incluyen electrodomésticos, sistemas de navegación, dispositivos médicos, controladores industriales y routers.

## SO de tiempo compartido vs tiempo real

**Tiempo compartido (*time-sharing*):**允许多个 usuario interactúan con el sistema simultáneamente, alternando rápidamente entre ellos para dar la ilusión de que cada uno tiene el sistema para sí mismo. El planificador de CPU alterna entre procesos cientos o miles de veces por segundo, proporcionando tiempos de respuesta interactivos.

**Tiempo real (*real-time*):** garantiza que las operaciones se completen dentro de plazos estrictamente definidos (*deadlines*). Se subdivide en:

- **Tiempo real duro (*hard real-time*):** el incumplimiento de un deadline es un fallo del sistema. Ejemplos: sistemas de control de vuelo, marcapasos, ABS de automóviles.
- **Tiempo real blando (*soft real-time*):** el incumplimiento de un deadline degrada el rendimiento pero no provoca un fallo catastrófico. Ejemplos: streaming de vídeo, videojuegos, sistemas de telecomunicaciones.

Los SO de tiempo real (QNX, VxWorks, FreeRTOS) utilizan planificadores deterministas que garantizan tiempos de respuesta predecibles, a diferencia de los SO de tiempo compartido que priorizan el throughput sobre la previsibilidad.

## SO monolítico vs microkernel vs híbrido

**Monolítico:** el kernel se ejecuta como un único proceso en modo núcleo, con todos los subsistemas (gestión de memoria, planificación, drivers, redes) compartiendo el mismo espacio de direcciones. Ofrece el mejor rendimiento pero es vulnerable a errores: un driver defectuoso puede colapsar todo el kernel. Linux es el ejemplo más utilizado de SO monolítico con módulos cargables dinámicamente.

**Microkernel:** solo las funciones más esenciales residen en el kernel, mientras que los drivers, sistemas de archivos y protocolos de red se ejecutan como procesos de usuario regulares. Ofrece mayor fiabilidad pero introduce overhead por la comunicación interprocesos. QNX y Minix son ejemplos de micronúcleos.

**Híbrido:** combina elementos de ambos enfoques, manteniendo algunos servicios críticos en el kernel mientras permite que otros se ejecuten como procesos de usuario. macOS (basado en XNU, un kernel híbrido de Mach y BSD) y Windows NT son ejemplos.

> La elección entre monolítico, micronúcleo e híbrido no es binaria: Linux, aunque técnicamente monolítico, permite la carga dinámica de módulos que se comportan funcionalmente como servidores de micronúcleo. La arquitectura real de los SO modernos a menudo difiere de las categorías teóricas.

