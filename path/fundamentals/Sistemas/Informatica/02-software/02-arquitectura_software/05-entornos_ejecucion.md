# Introducción a entornos de ejecución

Un entorno de ejecución (*runtime environment*) es el conjunto de componentes de software que proporcionan las condiciones y servicios necesarios para que un programa se ejecute. Incluye todo lo que un programa necesita para funcionar desde que se inicia hasta que termina: la memoria asignada, los recursos del sistema disponibles, las bibliotecas vinculadas y las convenciones de llamada que el programa debe seguir.

Los entornos de ejecución no son sinónimos de sistemas operativos, aunque están estrechamente relacionados. Un entorno de ejecución puede ser tan amplio como un sistema operativo completo o tan específico como una máquina virtual que ejecuta un único programa. Comprender los entornos de ejecución es fundamental para entender por qué un programa funciona correctamente en un contexto pero falla en otro, y cómo se gestionan los recursos durante la ejecución.

## Para qué sirven

Los entornos de ejecución resuelven varios problemas fundamentales:

**Portabilidad:** permiten que el mismo programa se ejecute en diferentes sistemas operativos y arquitecturas de hardware sin modificaciones. Una aplicación Java, por ejemplo, se ejecuta sobre la JVM (*Java Virtual Machine*), que abstracte las diferencias entre Windows, Linux y macOS.

**Gestión de recursos:** el entorno de ejecución gestiona la memoria, los hilos de ejecución y las conexiones de red que el programa necesita, sin que el programador deba preocuparse por los detalles de implementación.

**Seguridad:** los entornos de ejecución pueden imponer restricciones sobre qué operaciones puede realizar el programa, limitando su acceso a archivos, red y otros recursos del sistema.

**Depuración y diagnóstico:** los entornos de ejecución proporcionan herramientas para monitorizar el comportamiento del programa, detectar errores y analizar el rendimiento.

## Tipos de runtimes

**Runtime del sistema operativo:** el conjunto de bibliotecas y servicios que el SO proporciona a los programas compilados nativamente. En Linux, la biblioteca C (glibc o musl) proporciona las funciones estándar que los programas utilizan para interactuar con el kernel. En Windows, las DLL del sistema (kernel32.dll, ntdll.dll) cumplen una función similar.

**Máquinas virtuales (VM):** entornos de ejecución completos que emulan una arquitectura de hardware sobre otra. La JVM ejecuta bytecode Java sobre cualquier hardware compatible con una JVM. El CLR (*Common Language Runtime*) ejecuta código .NET sobre Windows, Linux o macOS. Las VM de virtualización (VirtualBox, VMware) ejecutan sistemas operativos completos sobre hardware físico.

**Entornos de ejecución de lenguajes interpretados:** intérpretes que ejecutan código directamente sin compilación previa. CPython ejecuta código Python, Node.js ejecuta JavaScript, y Ruby ejecuta código Ruby. Estos runtimes incluyen el intérprete, la gestión de memoria (recolector de basura) y las bibliotecas estándar.

**Contenedores:** entornos de ejecución que encapsulan una aplicación con todas sus dependencias usando las funcionalidades del kernel del SO (namespaces, cgroups en Linux). Docker y containerd son los runtimes de contenedores más utilizados. Los contenedores ofrecen aislamiento ligero sin el overhead de una máquina virtual completa.

**Serverless:** el entorno de ejecución más abstracto, donde el desarrollador despliega código sin preocuparse por el servidor, el SO o la infraestructura. AWS Lambda, Google Cloud Functions y Azure Functions ejecutan funciones en respuesta a eventos, escalando automáticamente según la demanda.

## Componentes principales

Todo entorno de ejecución, independientemente de su nivel de abstracción, incluye ciertos componentes fundamentales:

**Gestión de memoria:** asignación y liberación de memoria para el programa y sus datos. Puede ser manual (malloc/free en C), semiautomática (Rust) o completamente automática (recolector de basura en Java, Python, Go).

**Gestión de hilos y procesos:** el runtime controla la creación, planificación y terminación de hilos de ejecución. En lenguajes con concurrencia integrada (Go, Erlang), el runtime gestiona el *scheduler* de goroutines o procesos ligeros.

**Bibliotecas estándar:** conjunto de utilidades predefinidas que el programador puede utilizar sin implementarlas desde cero. La biblioteca estándar de C, las bibliotecas de Python (os, sys, math) y la JDK de Java son ejemplos.

**Gestión de errores y excepciones:** el runtime detecta y maneja errores durante la ejecución, proporcionando información diagnóstica y mecanismos de recuperación.

## Relación con el sistema operativo

El entorno de ejecución se sitúa entre el programa y el sistema operativo, actuando como intermediario. Un programa que se ejecuta sobre una JVM no interactúa directamente con el SO: la JVM se encarga de traducir las operaciones del bytecode a llamadas al sistema nativas del SO subyacente.

Esta relación de capas tiene implicaciones de rendimiento: cada capa de abstracción introduce un overhead. Un programa compilado nativamente tiene acceso directo al SO con mínimo overhead. Un programa que se ejecuta sobre una VM tiene un nivel adicional de indirección que puede afectar al rendimiento, especialmente en operaciones intensivas de CPU.

> La elección del entorno de ejecución implica un trade-off entre portabilidad, rendimiento y facilidad de desarrollo. Los runtimes de VM ofrecen máxima portabilidad pero menor rendimiento. Los entornos nativos ofrecen máximo rendimiento pero menor portabilidad. Los contenedores ofrecen un equilibrio intermedio que ha conquistado la industria.

