# Java

Java es un lenguaje de programacion orientado a objetos de proposito general, disenado para tener la menor dependencia de implementacion posible. Fue creado por James Gosling en Sun Microsystems y lanzado publicamente en 1995. Su filosofia central se resume en el lema "Write Once, Run Anywhere" (WORA): un programa compilado en una maquina puede ejecutarse en cualquier otra que disponga de una maquina virtual de Java (JVM) compatible, sin necesidad de recompilacion.

## La plataforma Java

Java no es solo un lenguaje, sino una plataforma completa compuesta por tres componentes fundamentales. La JVM (Java Virtual Machine) es la pieza central: interpreta el bytecode generado por el compilador y lo traduce a instrucciones nativas del sistema operativo subyacente. El JRE (Java Runtime Environment) incluye la JVM junto con las bibliotecas de clases necesarias para ejecutar programas. El JDK (Java Development Kit) incorpora el JRE mas herramientas de desarrollo como el compilador javac, el depurador y otras utilidades.

### Write Once, Run Anywhere

El modelo WORA se basa en un nivel de abstraccion intermedio: el bytecode. Cuando se compila un archivo `.java`, el compilador no genera codigo maquina para un procesador especifico, sino un formato intermedio universal representado en archivos `.class`. Estos archivos contienen instrucciones para la JVM, no para el hardware directamente. Cada plataforma implementa su propia JVM que traduce ese bytecode a instrucciones nativas, garantizando que el comportamiento sea consistente independientemente del sistema operativo o arquitectura de hardware.

> Este enfoque elimina el problema clasico de las plataformas nativas, donde el mismo programa debia compilarse por separado para Windows, Linux y macOS. Con Java, el bytecode es el contrato comun entre el desarrollador y la plataforma.

## Ediciones de Java

Java se organiza en tres ediciones principales, cada una orientada a un dominio de aplicacion distinto:

- **Java SE (Standard Edition):** La edicion base que proporciona la API principal del lenguaje. Incluye colecciones, concurrencia, E/S basica, expresiones regulares y la mayor parte de la funcionalidad que se usa en desarrollo general. Es la base sobre la que se construyen las demas ediciones.

- **Java EE (Enterprise Edition):** Amplia la SE con APIs para desarrollo de aplicaciones empresariales a gran escala. Incluye soporte para servlets, JSP, EJB, JPA, JMS y servicios web. A partir de Jakarta EE, pasó a ser mantenido por la Eclipse Foundation bajo el nombre Jakarta EE.

- **Java ME (Micro Edition):** Disenada para dispositivos con recursos limitados como telefonos moviles antiguos, dispositivos IoT y sistemas embebidos. Utiliza un subconjunto reducido de la API de Java SE y una JVM optimizada para entornos con poca memoria y capacidad de procesamiento.

## Versiones relevantes

Java ha evolucionado significativamente desde su lanzamiento. Algunas versiones marcaron puntos de inflexion en la plataforma:

- **Java 8 (2014):** Introdujo las expresiones lambda, el API Stream, Optional y los metodos por defecto en interfaces. Es considerada una de las versiones mas influyentes y aun se mantiene en muchos entornos productivos por su estabilidad y compatibilidad.

- **Java 11 (2018):** LTS (Long-Term Support) que elimino el JRE independiente, introdujo el operador de string (`var` en lambdas), la HttpClient API y la ejecucion directa de scripts con `jshell`.

- **Java 17 (2021):** LTS que incluyo records, sealed classes, pattern matching para instanceof y el operador de text blocks. Consolido multiples previews como estables.

- **Java 21 (2023):** LTS mas reciente con virtual threads (Project Loom), pattern matching para switch, sequenced collections y el operador de string.

> Las versiones LTS (Long-Term Support) reciben actualizaciones de seguridad y correcciones de bugs durante varios anos, lo que las convierte en la eleccion preferida para entornos de produccion.

## JIT Compiler

La JVM utiliza un compilador JIT (Just-In-Time) que optimiza el rendimiento en tiempo de ejecucion. Cuando un programa se ejecuta, la JVM comienza interpretando el bytecode instruccion por instruccion. A medida que se identifican las partes del codigo que se ejecutan con frecuencia (hot spots), el compilador JIT las compila a codigo maquina nativo y las almacena en cache. Este proceso de compilacion selectiva permite que el programa se acelere progresivamente sin necesidad de precompilacion completa.

El JIT también realiza optimizaciones avanzadas que no son posibles en compilacion estatica, como la especializacion de metodos segun los tipos reales en tiempo de ejecucion, la eliminacion de llamadas a metodos polimorficos y la reordenacion de codigo basada en perfiles de ejecucion reales.

## Garbage Collector

Java gestiona la memoria automaticamente mediante un garbage collector (recolector de basura). A diferencia de lenguajes como C o C++, donde el programador debe reservar y liberar memoria manualmente, la JVM se encarga de identificar y recuperar el memoria que ya no esta siendo utilizada por ningun objeto activo.

El garbage collector opera sobre el heap, la zona de memoria donde se almacenan los objetos. Cuando se crea un nuevo objeto con `new`, la JVM reserva espacio en el heap. Cuando una referencia a ese objeto ya no es accesible desde ningun hilo de ejecucion, el objeto se considera candidato a ser recolectado. El GC recorre el grafo de objetos alcanzables desde las raices de referencia (variables estaticas, locales del marco de pila, registros de registradores) y marca como alcanzables todo lo que puede seguir siendo utilizado. El resto se libera.

Las JVM modernas utilizan garbage collectors como G1 (por defecto desde Java 9), ZGC y Shenandoah, que estan disenados para minimizar las pausas de recoleccion y proporcionar un rendimiento predecible en aplicaciones con alta carga.


