# BIOS y UEFI

La BIOS (*Basic Input/Output System*) y su sucesora UEFI (*Unified Extensible Firmware Interface*) son el firmware que reside en un chip de la placa base y se ejecuta al momento de encender el sistema. Su función principal es inicializar y probar los componentes del hardware antes de cargar el sistema operativo. Sin este firmware, la CPU no sabría cómo comunicarse con la memoria, el almacenamiento ni los dispositivos de E/S.

La transición de BIOS a UEFI representa uno de los cambios más significativos en la arquitectura de los ordenadores personales en las últimas décadas. Mientras que la BIOS es un sistema de 16 bits con limitaciones severas, UEFI es un entorno moderno, modular y seguro que permite funcionalidades impensables en la era de la BIOS clásica.

## Función

Tanto la BIOS como la UEFI cumplen las mismas funciones básicas:

- **Inicialización del hardware:** al encender la CPU, el firmware se carga y ejecuta una serie de pruebas para verificar que los componentes críticos —CPU, memoria, tarjetas de expansión— estén funcionando correctamente.
- **POST (*Power-On Self-Test*):** durante la inicialización, se realizan pruebas diagnósticas que verifican la integridad del hardware. Si se detecta un error, el sistema emite una secuencia de pitidos o muestra un código de error en un display LED.
- **Carga del sistema operativo:** una vez que el hardware está inicializado, el firmware busca un gestor de arranque (*bootloader*) en el dispositivo de almacenamiento indicado y transfiere el control al sistema operativo.
- **Gestión de la configuración:** el firmware proporciona una interfaz (tradicionalmente accesible con la tecla Suprimir o F2) donde el usuario puede modificar la configuración del hardware, como la secuencia de arranque, las opciones de overclock o la habilitación/deshabilitación de dispositivos.

> El firmware es la primera capa de software que se ejecuta en una computadora. Incluso antes de que el sistema operativo se cargue, el firmware ya ha tomado decisiones críticas sobre cómo se configuran los componentes del sistema.

## Diferencias entre BIOS y UEFI

La BIOS clásica es un sistema de 16 bits con una interfaz de texto primitiva y limitaciones técnicas severas:

- Solo puede acceder a un espacio de direccionamiento de 1 MB de memoria.
- No soporta particiones de disco duro mayores a 2.2 TB (límite del esquema de partición MBR).
- La interfaz es basada en texto, con navegación limitada por teclado.
- El código del firmware es rígido y difícil de actualizar.

UEFI supera estas limitaciones con un diseño moderno:

- Opera en modo de 32 o 64 bits, con acceso a grandes volúmenes de memoria.
- Utiliza la tabla de particiones GPT (*GUID Partition Table*), que soporta discos de hasta 9.4 ZB (zettabytes).
- Incluye una interfaz gráfica navegable con ratón.
- Permite cargar controladores de dispositivos de forma modular.
- Soporta autenticación de firmware y arranque seguro (*Secure Boot*).

La compatibilidad con BIOS (*Legacy BIOS*) se mantiene en muchos sistemas UEFI para permitir la ejecución de sistemas operativos antiguos, pero se recomienda utilizar el modo UEFI nativo siempre que sea posible.

## Proceso de arranque

El proceso de arranque de un sistema UEFI moderno sigue estas fases:

**Fase 1: Inicialización del firmware.** La CPU ejecuta el código UEFI desde el chip flash de la placa base. Se inicializan los componentes críticos: la controladora de memoria, los carriles PCIe y los dispositivos de almacenamiento.

**Fase 2: POST.** Se realizan las pruebas de hardware. En sistemas UEFI avanzados, se pueden monitorizar los voltajes, temperaturas y estados de los componentes durante esta fase.

**Fase 3: Búsqueda del gestor de arranque.** El firmware busca un gestor de arranque firmado en la partición EFI del disco duro (una partición especial de tipo FAT32). En sistemas con Secure Boot, el gestor de arranque debe estar firmado con un certificado reconocido por el firmware.

**Fase 4: Carga del sistema operativo.** El gestor de arranque carga el núcleo del sistema operativo en memoria y transfiere el control. En Linux, esto implica cargar el kernel y el initrd; en Windows, el gestor de arranque de Windows (bootmgr).

**Fase 5: Ejecución del sistema operativo.** El sistema operativo toma el control completo del hardware y completa la inicialización.

## Configuración básica

La interfaz de configuración del firmware permite al usuario modificar parámetros críticos del sistema:

- **Secuencia de arranque:** el orden en que el firmware busca dispositivos de arranque. Por defecto, busca en el SSD principal, pero se puede configurar para arrancar desde USB, red o dispositivos ópticos.
- **Habilitación/deshabilitación de dispositivos:** se pueden desactivar componentes integrados como el audio, la red o los puertos USB que no se utilicen.
- **Configuración de memoria:** ajustes de frecuencia, perfiles XMP/EXPO y timings de la memoria RAM.
- **Opciones de overclock:** multiplicador de CPU, voltajes y frecuencias de memoria para usuarios avanzados.
- **Virtualización:** habilitar o deshabilitar VT-x (Intel) o AMD-V para la ejecución de máquinas virtuales.

## CMOS y pila

La configuración del firmware se almacena en un chip CMOS (*Complementary Metal-Oxide Semiconductor*) alimentado por una pila botón (típicamente CR2032). Cuando el sistema está apagado, la pila mantiene la configuración guardada en el CMOS.

Si la pila se agota, la configuración del firmware se pierde y el sistema vuelve a los valores por defecto de fábrica. Este es un mecanismo de seguridad: si un usuario configura opciones incorrectas que impiden el arranque, desconectar la pila durante unos segundos restablece la configuración original.

> El chip CMOS moderno ha sido parcialmente reemplazado por la memoria flash del firmware. Sin embargo, la pila sigue siendo necesaria para mantener la hora del sistema (*Real-Time Clock*) y ciertos registros de configuración que se cargan inmediatamente al encender.

## Seguridad

La seguridad del firmware se ha convertido en un área crítica con el aumento de amenazas que operan a nivel de hardware.

### Secure Boot

Secure Boot es un estándar de la UEFI que garantiza que solo firmware y sistemas operativos firmados por entidades reconocidas puedan ejecutarse durante el arranque. Esto impide que malware de bajo nivel (*bootkits* y *rootkits*) se cargue antes que el sistema operativo.

El mecanismo funciona utilizando una cadena de confianza: cada componente del proceso de arranque debe estar firmado digitalmente, y el firmware verifica estas firmas antes de cargar el componente. Si la verificación falla, el firmware bloquea la ejecución.

> Secure Boot es especialmente relevante en entornos corporativos donde la seguridad es prioritaria. Sin embargo, puede generar complicaciones con sistemas operativos no comerciales o con hardware personalizado, ya que las firmas deben ser registradas en el firmware.

### TPM y firmware TPM

El TPM (*Trusted Platform Module*) es un chip dedicado a operaciones criptográficas: generación de claves, almacenamiento seguro de certificados y mediciones de integridad del sistema. Windows 11 requiere TPM 2.0 como requisito obligatorio.

El *firmware TPM* (fTPM) implementa las mismas funcionalidades del TPM físico pero ejecutándose como software dentro del firmware UEFI. Esto elimina la necesidad de un chip adicional en la placa base, pero puede ser vulnerable a ataques de software.

### Contraseñas de arranque y de BIOS

El firmware permite establecer contraseñas para:

- **Contraseña de BIOS/UEFI:** impide el acceso a la interfaz de configuración sin la contraseña correcta.
- **Contraseña de usuario:** solicita una contraseña cada vez que el sistema se enciende, impidiendo el uso no autorizado del ordenador.
- **Contraseña de disco:** protege los dispositivos de almacenamiento, requiriendo una clave para acceder a los datos.

### Protección contra malware en el arranque

Más allá de Secure Boot, los firmware modernos incorporan protecciones adicionales como el *Measured Boot*, que registra métricas de integridad de cada componente del proceso de arranque y las almacena en el TPM. Estas métricas pueden ser verificadas remotamente para garantizar que el sistema no ha sido comprometido.

## Herramientas avanzadas (según placa)

Las placas base de gama alta incluyen herramientas de diagnóstico y configuración avanzada en el firmware:

### Monitorización de voltajes y temperaturas

La interfaz del firmware puede mostrar en tiempo real los voltajes de la CPU, la memoria y los reguladores de voltaje, así como las temperaturas de los componentes críticos. Esta información es esencial para detectar problemas de alimentación o refrigeración antes de que causen daños.

### Control de ventiladores

Los firmware modernos permiten configurar curvas de velocidad de los ventiladores en función de la temperatura. Esto permite establecer perfiles de ruido/rendimiento personalizados sin necesidad de software de terceros.

### Actualización de BIOS (Flashback, EZ Flash, Q-Flash)

La actualización del firmware es un proceso crítico que corrige bugs, añade compatibilidad con nuevos procesadores y mejora la seguridad. Los sistemas modernos ofrecen herramientas que permiten actualizar el firmware sin necesidad de un sistema operativo funcional:

- **BIOS Flashback:** permite actualizar el firmware usando solo una memoria USB y la fuente de alimentación, sin necesidad de CPU ni memoria instalados. Es extremadamente útil para resolver problemas de compatibilidad con procesadores nuevos.
- **EZ Flash (ASUS), Q-Flash (Gigabyte):** utilidades integradas en el firmware que permiten actualizar desde una memoria USB.

### Opciones de overclock

Los firmware de placas de gama alta incluyen opciones detalladas de overclock para CPU y memoria, incluyendo ajustes de multiplicador, voltaje del núcleo, frecuencia del bus, timings de memoria y perfiles de alimentación avanzados.

