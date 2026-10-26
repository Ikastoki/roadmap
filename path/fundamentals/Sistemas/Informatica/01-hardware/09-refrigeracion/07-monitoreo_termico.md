# Monitoreo térmico

El monitoreo térmico es la práctica de supervisar continuamente las temperaturas de los componentes de un sistema informático para garantizar que operan dentro de sus rangos seguros. Un sistema de monitoreo adecuado permite identificar problemas de refrigeración antes de que provoquen daños, optimizar la configuración de ventiladores para equilibrar rendimiento y ruido, y documentar el comportamiento térmico del sistema bajo diferentes cargas de trabajo.

La monitorización térmica no es un lujo sino una necesidad en cualquier sistema, desde un PC doméstico hasta un servidor de centro de datos. Los sensores de temperatura integrados en los componentes proporcionan datos en tiempo real que, adecuadamente interpretados, permiten tomar decisiones informadas sobre la configuración del sistema y la planificación de mantenimiento.

## Sensores de temperatura

Los sensores de temperatura están integrados en prácticamente todos los componentes modernos de un sistema informático:

**CPU:** los procesadores modernos incorporan múltiples sensores distribuidos por el die, que reportan temperaturas individuales por núcleo así como una temperatura general del paquete. Los sensores de CPU utilizan diodos de silicio cuya tensión directa varía con la temperatura, proporcionando una precisión de ±2°C.

**GPU:** las tarjetas gráficas incluyen sensores en el chip gráfico, la memoria VRAM y los VRM. La GPU reporta típicamente una temperatura de junction (la más alta de todos los sensores) y una temperatura de borde del die.

**Almacenamiento:** los SSD y HDD incluyen sensores de temperatura en sus firmware, accesibles mediante protocolo S.M.A.R.T. (*Self-Monitoring, Analysis and Reporting Technology*). Los SSD NVMe reportan temperatura en tiempo real y pueden activar protección térmica si se superan los límites.

**Placa base:** los sensores de la placa base monitorizan la temperatura del chipset, los VRM, los slots de memoria y los puertos SATA. Estos datos son especialmente útiles para identificar puntos calientes que podrían afectar a la estabilidad del sistema.

**Ambiente:** algunos sistemas incluyen sensores de temperatura ambiente que miden la temperatura del aire que entra al chasis, proporcionando una referencia para evaluar la eficiencia del sistema de refrigeración.

> La precisión de los sensores integrados varía según el fabricante y el modelo. Los sensores de CPU y GPU son generalmente fiables para fines de monitorización, pero las diferencias de ±2-5°C entre sensores de diferentes componentes son normales y no indican necesariamente un problema.

## Software de control

El software de monitorización térmica permite visualizar las temperaturas en tiempo real, configurar alertas y ajustar la velocidad de los ventiladores. Las herramientas más utilizadas incluyen:

**HWMonitor:** herramienta ligera que muestra temperaturas, voltajes y velocidades de rotación de todos los componentes del sistema. Es ideal para una supervisión rápida y sencilla sin configuración compleja.

**Core Temp:** se centra específicamente en la monitorización de la temperatura de la CPU, mostrando la temperatura de cada núcleo individual. Incluye opciones de alerta que notifican al usuario cuando se supera un umbral de temperatura configurado.

**SpeedFan:** permite no solo monitorizar temperaturas sino también controlar la velocidad de los ventiladores del chasis. Es útil para crear perfiles de ventilación personalizados que equilibren refrigeración y ruido.

**Open Hardware Monitor:** herramienta de código abierto que proporciona datos completos de temperatura, voltaje y velocidad de todos los componentes, con soporte para monitoreo remoto a través de la red.

**Software del fabricante:** los fabricantes de placas base (ASUS AI Suite, MSI Center, Gigabyte SIV) y tarjetas gráficas (MSI Afterburner, EVGA Precision X1) incluyen herramientas específicas que aprovechan las capacidades completas del hardware, incluyendo control de ventiladores, ajuste de voltajes y monitorización avanzada.

**BIOS/UEFI:** la mayoría de las BIOS/UEFI modernas incluyen monitores de temperatura integrados que permiten verificar el estado térmico del sistema antes de cargar el sistema operativo. Es una herramienta útil para diagnosticar problemas de refrigeración desde el primer momento del arranque.

La configuración de alertas de temperatura es particularmente importante: establecer un umbral de alerta a 85°C para la CPU permite recibir una notificación antes de que se alcance el límite de throttling (típicamente 95-100°C), dando tiempo al usuario para identificar y resolver el problema de refrigeración antes de que afecte al rendimiento.

