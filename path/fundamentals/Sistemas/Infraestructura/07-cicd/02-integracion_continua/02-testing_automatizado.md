# Testing Automatizado

Un pipeline de integracion continua sin pruebas automatizadas es una maquina de rapida compilacion de codigo defectuoso. Las pruebas automaticas son el mecanismo que verifica que los cambios introducidos por un desarrollador no rompen comportamientos existentes ni introducen regresiones. Sin ellas, la integracion continua se reduce a integrar codigo con frecuencia sin garantizar nada sobre su calidad, lo cual es potencialmente peor que integrar con poca frecuencia al menos sabiendo que se verificara manualmente antes.

La clave del testing automatizado no es solo ejecutar pruebas, sino ejecutarlas en el momento correcto, con la velocidad adecuada y reportando resultados de forma clara. Un suite de pruebas que tarda una hora en ejecutarse no puede feedback rapido a un desarrollador que acaba de hacer un push. El diseño de la suite de pruebas debe equilibrar cobertura, velocidad y mantenibilidad para que el pipeline de CI sea util en el ciclo de desarrollo diario.

## La piramide de testing

La piramide de testing es un modelo de referencia que describe la distribucion ideal de tipos de pruebas en un proyecto. En la base se encuentran las pruebas unitarias: numerosas, rapidas y aisladas. En el medio estan las pruebas de integracion: menos numerosas, mas lentas y que verifican la interaccion entre componentes. En la cima estan las pruebas end-to-end (E2E): las menos numerosas, las mas lentas y las que validan flujos completos del usuario a traves de multiples sistemas.

```
        /  E2E  \        <-- Pocas, lentas, frágiles
       /----------\
      / Integracion \    <-- Moderadas, validan interfaces
     /----------------\
    /   Unitarias      \  <-- Muchas, rápidas, aisladas
   /____________________\
```

La forma de piramide refleja una realidad practica: las pruebas unitarias se ejecutan en milisegundos y pueden ejecutarse cientos o miles en cada build sin impacto significativo en el tiempo. Las pruebas de integracion requieren levantar servicios o base de datos, por lo que son mas costosas. Las pruebas E2E, que simulan un usuario real navegando la aplicacion, son las mas valiosas para detectar problemas visibles pero las mas costosas de mantener y ejecutar. Invertir la piramide (muchas E2E, pocas unitarias) produce un pipeline lento, fragil y dificil de mantener.

Las pruebas unitarias verifican que una funcion o metodo individual produce el resultado esperado dado una entrada determinada. No dependen de bases de datos, servicios externos ni estado compartido. Su velocidad de ejecucion (microsegundos o milisegundos) permite ejecutar miles de ellas en segundos. Una buena suite unitaria cubre casos normales, bordes y errores, y su mantenimiento es directamente proporcional al acoplamiento del codigo que verifican.

Las pruebas de integracion verifican que multiples componentes funcionan correctamente cuando interactuan. Esto incluye la comunicacion entre servicios, el acceso a bases de datos, la integracion con APIs externas o el comportamiento de un middleware. Son mas lentas porque requieren levantar dependencias reales o simuladas, pero capturan problemas que las pruebas unitarias no pueden detectar: contratos de interfaces mal interpretados, transacciones de base de datos incompletas o protocolos de comunicacion mal implementados.

Las pruebas end-to-end validan flujos completos del usuario a traves de la interfaz de la aplicacion (API, navegador, CLI). Son las mas cercanas a la experiencia real del usuario pero tambien las mas fragiles: un cambio en un identificador CSS o en el texto de un boton puede romper una prueba que nada tiene que ver con el comportamiento business. Por esto se mantienen en numero reducido, enfocadas en los flujos criticos del negocio.

## Pruebas en cada push

La integracion continua exige que cada commit o push al repositorio remoto dispare la ejecucion del suite de pruebas. Esta frecuencia es intencional: cuanto mas rapido se detecta un error, mas barato es corregirlo. Un error detectado por una prueba unitaria inmediatamente despues del commit puede resolverse en minutos. El mismo error descubierto manualmente dias despues, cuando multiples commits se han acumulado, puede requerir horas de depuracion para determinar que cambio lo introdujo.

La velocidad es un requisito funcional del pipeline. Si el suite completo tarda mas de 10-15 minutos, los desarrolladores empezaran a evitarlo o a ejecutarlo localmente de forma selectiva, lo que anula el proposito de la integracion continua. Estrategias como la ejecucion paralela de pruebas, el filtrado inteligente (solo ejecutar pruebas afectadas por el cambio) y la separacion en pipelines rapido/lento ayudan a mantener tiempos de feedback aceptables.

## Analisis estatico de codigo

El analisis estatico examina el codigo fuente sin ejecutarlo, buscando problemas de calidad, seguridad y mantenibilidad. Herramientas como linters (ESLint, Flake8, RuboCop), formateadores (Prettier, Black) y analizadores de seguridad (SonarQube, Bandit, Semgrep) se ejecutan como parte del pipeline de CI, generalmente en paralelo con las pruebas.

Los linters verifican que el codigo sigue las convenciones del equipo (indentacion, nombres de variables, estructura de funciones) y detectan patrones problematicos (variables no utilizadas, imports circulares, tipos implicitos). Su ejecucion es extremadamente rapida y su valor es proporcional a la consistencia que impone en un equipo grande.

Los analizadores de seguridad estatica (SAST - Static Application Security Testing) buscan vulnerabilidades en el codigo fuente: inyeccion de SQL, cross-site scripting, uso de funciones criptograficas debiles, exposicion de secretos. Estas herramientas funcionan como un primer filtro antes de que el codigo llegue a revision manual o a pruebas de seguridad mas sofisticadas (DAST, fuzzing).

> El analisis estatico no sustituye la revision de codigo ni las pruebas de seguridad dinamicas. Es una capa adicional de defense-in-depth que captura problemas de forma rapida y economica.

## Cobertura de codigo como metrica

La cobertura de codigo mide que proporcion del codigo fuente es ejecutada por las pruebas. Una cobertura del 80% significa que el 80% de las lineas (o ramas, o instrucciones) del codigo fuente fue ejecutada al menos una vez durante la ejecucion del suite de pruebas. Herramientas como JaCoCo (Java), Istanbul (JavaScript) o Coverage.py (Python) generan reportes detallados que identifican que partes del codigo no estan siendo verificadas.

La cobertura es una metrica util pero peligrosa si se malinterpreta. Un 100% de cobertura no garantiza un codigo sin bugs: puede haber pruebas que ejecuten el codigo sin verificar sus resultados, o pruebas que no cubran combinaciones de entrada relevantes. Una cobertura alta sinaseveraciones de calidad no aporta nada. El valor real de la metrica esta en su tendencia: si la cobertura baja con un commit, ese commit introdujo codigo no verificado, lo cual es una senial de alerta.

Los equipos suelen establecer un umbral minimo de cobertura (tipicamente entre el 70% y el 85%) como condicion para que el pipeline de CI sea exitoso. Este umbral debe adaptarse al contexto: un modulo de seguridad critica puede requerir cobertura superior al 95%, mientras que un modulo de utilidades administrativas puede funcionar con menos. La metrica debe usarse como guia para mejorar, no como un fin en si misma.

> Cobertura alta sinaseveraciones meaningful es como tener un examen con todas las preguntas respondidas pero sin calificar ninguna respuesta. La metrica por si sola no dice nada sobre la calidad real de las pruebas.

