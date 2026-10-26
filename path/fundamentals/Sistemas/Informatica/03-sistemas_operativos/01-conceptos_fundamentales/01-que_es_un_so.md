# Qué es un sistema operativo

Un sistema operativo (*Operating System*, OS) es el software de sistema que gestiona los recursos de hardware de una computadora y proporciona servicios a las aplicaciones que se ejecutan sobre él. Es la primera capa de software que se carga al encender el equipo y permanece activa durante toda la sesión de uso, actuando como intermediario entre el hardware y el software de aplicación.

Sin un sistema operativo, cada aplicación necesitaría implementar su propia gestión de hardware: comunicarse directamente con el procesador, gestionar la memoria, controlar los dispositivos de E/S y coordinar el acceso concurrente a los recursos compartidos. Esto resultaría en código duplicado, conflictos entre aplicaciones y una fragilidad extrema del sistema.

## Función como intermediario

El sistema operativo abstracte la complejidad del hardware detrás de interfaces simplificadas que las aplicaciones pueden utilizar sin conocer los detalles de implementación. Cuando un programa solicita leer un archivo, el sistema operativo se encarga de localizar los bloques de datos en el disco, comunicarse con el controlador de almacenamiento, transferir los datos a la memoria y devolverlos al programa.

Esta función intermediaria se extiende a todos los recursos del sistema:

**CPU:** el SO distribuye el tiempo de procesador entre múltiples procesos mediante la planificación, garantizando que cada proceso reciba una porción justa de tiempo de ejecución.

**Memoria:** el SO asigna y gestiona la memoria virtual de cada proceso, aislándolos entre sí y proporcionando una ilusión de espacio de direcciones contiguo.

**Almacenamiento:** el SO gestiona los sistemas de archivos, controla el acceso a los discos y proporciona una vista lógica de los archivos sin que las aplicaciones necesiten conocer los detalles físicos del almacenamiento.

**Dispositivos:** el SO controla la comunicación con teclados, ratones, pantallas, impresoras y otros periféricos a través de drivers especializados.

> El concepto de intermediario es clave: el sistema operativo es simultáneamente un administrador de recursos (garantiza que cada proceso reciba lo que necesita), un protector (aisla los procesos entre sí y del hardware) y un traductor (convierte operaciones de alto nivel en comandos de hardware).

## Evolución histórica

La historia de los sistemas operativos refleja la evolución de la informática desde sistemas de lote hasta plataformas distribuidas modernas:

**Sistemas de lote (*batch systems*, 1950s-1960s):** los primeros SO permitían ejecutar trabajos en secuencia sin interacción del usuario. Los operadores agrupaban trabajos en lotes que se ejecutaban uno tras otro, minimizando el tiempo muerto del procesador pero ofreciendo cero interactividad.

**Sistemas de tiempo compartido (*time-sharing*, 1960s-1970s):** introdujeron la multitarea y la interacción simultánea de múltiples usuarios. Unix, creado en 1969 en los Laboratorios Bell, sentó las bases de los SO modernos con su jerarquía de archivos, su shell y su diseño modular.

**SO de escritorio (1980s-presente):** MS-DOS, Windows, macOS y Linux introdujeron interfaces gráficas que hicieron la informática accesible al público general. La evolución de monolíticos (DOS) a multitarea con protección de memoria (Windows NT, Linux) mejoró drásticamente la estabilidad y la seguridad.

**SO de dispositivos móviles (2007-presente):** iOS y Android adaptaron los conceptos de SO a dispositivos con recursos limitados, pantallas táctiles y conectividad siempre activa, introduciendo nuevos paradigmas de seguridad y gestión de energía.

**SO distribuidos y cloud (2010s-presente):** Kubernetes, Docker y las plataformas cloud representan la evolución hacia SO que gestionan clústeres de miles de servidores como una sola entidad computacional.

## Arquitectura de un SO moderno

Un SO moderno se compone de varios componentes principales:

**Kernel:** el núcleo del sistema operativo, que tiene acceso directo al hardware y gestiona los recursos más fundamentales. El kernel implementa la gestión de procesos, memoria, archivos, dispositivos y redes.

**Shell e interfaz gráfica:** la capa de interacción con el usuario. El shell de línea de comandos (bash, zsh, PowerShell) proporciona una interfaz textual para ejecutar comandos y scripts. Las interfaces gráficas (GUI) ofrecen una interacción visual más intuitiva.

**Bibliotecas del sistema:** colecciones de funciones predefinidas que las aplicaciones pueden utilizar para interactuar con el SO. La biblioteca C (glibc, musl) es la más fundamental, pero existen bibliotecas para gráficos, redes, audio y otras funcionalidades.

**Servicios del sistema:** procesos background que proporcionan funcionalidades continuas: gestión de red, programación de tareas, logs del sistema, sincronización de tiempo.

**Drivers:** módulos de kernel que permiten la comunicación con hardware específico. Los drivers pueden estar integrados en el kernel o cargados dinámicamente.

