# Fundamentos de Logging

Un sistema sin logs es como un barco sin cuaderno de bitacola: navega sin dejar rastro, y cuando algo sale mal, no hay forma de reconstruir lo ocurrido. Los logs son el registro cronologico y persistente de lo que sucede dentro de una aplicacion o infraestructura. Cada linea de un log es una pieza de evidencia que, bien estructurada, permite diagnosticar fallos, entender comportamientos y cumplir con requisitos de auditoria. Sin embargo, no todo registro es igual de valioso: un log excesivamente detallado ahoga al operador en ruido, mientras que uno demasiado escaso no ofrece contexto suficiente para resolver un incidente.

## Que es un log y que debe contener

Un log es un registro inmutable que captura un evento en el momento en que ocurre. No es un indicador de salud (eso son las metricas) ni una grafica de comportamiento temporal (eso es el tracing): es una fotografia de un instante especifico. Cada entrada de log debe contener, como minimo, cuatro elementos fundamentales: **timestamp** preciso (con zona horaria UTC), el **nivel de severidad**, un **mensaje descriptivo** del evento, y un **identificador de contexto** que permita agrupar entradas relacionadas. Sin estas piezas basicas, el log pierde la mitad de su valor diagnostico.

La analogia del cuaderno de bitacola de un barco es util: un capitan no anota solo "algo raro paso en la noche". Registra la hora exacta, las condiciones del mar, la decision tomada y el resultado. Los logs siguen este mismo principio: cada entrada debe ser suficiente por si misma para que otro operador, a los tres dias, pueda entender que ocurrio sin necesidad de preguntar a nadie.

## Niveles de severidad

Los niveles de log establecen una jerarquia de importancia que permite filtrar y priorizar informacion. No son etiquetas decorativas: cada nivel tiene un proposito especifico y un impacto diferente en la operacion del sistema.

- **TRACE**: Nivel de traza, el mas granular posible. Registra el flujo interno de una funcion o algoritmo paso a paso. Se usa durante el desarrollo para entender el camino exacto que toma el codigo. En produccion, generalmente se desactiva porque genera un volumen ingente de datos que ocultan lo relevante.

- **DEBUG**: Informacion de depuracion, util para entender el estado interno del sistema en un momento dado. Muestra variables intermedias, resultados parciales de procesos, o el estado de una conexion antes de una operacion. En produccion solo se activa temporalmente cuando se investiga un problema especifico.

- **INFO**: Mensajes informativos sobre el flujo normal de la aplicacion. Representan eventos esperados y correctos: "conexion establecida", "solicitud procesada", "tarea completada". Son el nivel minimo recomendado para operacion en produccion, ya que permiten reconstruir la secuencia de operaciones exitosas.

- **WARN**: Indica una condicion inesperada que no impide el funcionamiento, pero que podria convertirse en un problema si persiste. Un timeout que se resuelve en el segundo intento, un recurso que se agota temporalmente, o una configuracion que usa un valor por defecto en lugar del esperado. Es una senal de que algo merece atencion preventiva.

- **ERROR**: Un evento de error que indica que una operacion especifica ha fallado. La solicitud del usuario no pudo completarse, una consulta a la base de datos fallo, o un servicio externo no responde. El sistema sigue operando, pero algo concreto ha fallado y requiere investigacion.

- **FATAL**: Error critico que causa la terminacion del proceso o servicio. No hay forma de recuperarse automaticamente. Ejemplos incluyen la incapacidad de conectar con la base de datos al iniciar, fallo en la verificacion de integridad de datos criticos, o agotamiento de memoria que impide continuar. Despues de un FATAL, el servicio deja de estar disponible.

> La recomendacion practica es usar INFO como nivel minimo en produccion y subir o bajar segun el ruido que se este generando. Un sistema con demasiados WARN puede indicar una configuracion inadecuada, no un sistema saludable.

## Logs estructurados vs no estructurados

Los logs no estructurados son lineas de texto libre: el desarrollador escribe lo que le parece relevante en el formato que le place. Esto genera entradas como:

```
2025-03-15 14:23:01 ERROR - La consulta fallo porque la tabla no existe, usuario: admin
```

Este formato es legible para un humano, pero problematico para una maquina. Extraer el usuario, la hora exacta o el tipo de error requeriria expresiones regulares fragiles que se rompen ante cualquier variacion en el formato.

Los logs estructurados resuelven este problema al codificar cada campo de forma explicita y predecible. JSON se ha convertido en el estandar de facto para logs estructurados porque es nativo en la mayoria de lenguajes de programacion, soporta tipos anidados y es understood por praticamente todas las herramientas de gestion de logs.

Un log estructurado del mismo evento anterior se veria asi:

```json
{
  "timestamp": "2025-03-15T14:23:01Z",
  "level": "ERROR",
  "message": "Consulta fallida: tabla inexistente",
  "user": "admin",
  "query": "SELECT * FROM reports",
  "table": "reports"
}
```

Cada campo es identificable, buscable y filtrable de forma mecanica. No hay ambiguedad sobre que parte del texto corresponde al usuario o a la consulta. Las herramientas de analisis pueden indexar cada campo individualmente, ejecutar agregaciones sobre valores especificos y establecer alertas basadas en condiciones concretas sobre campos determinados.

## Por que la estructuracion es critica

Cuando un sistema crece de un solo servidor a decenas o cientos de instancias distribuidas, la capacidad de buscar y filtrar logs se convierte en una necesidad operativa, no en un lujo. Los logs estructurados permiten que herramientas como Elasticsearch, Loki o CloudWatch indexen cada campo y permitan consultas como "mostrame todos los errores del usuario X en la ultima hora" o "cuantas veces fallo la conexion a la base de datos Y en los ultimos 7 dias". Con logs no estructurados, estas consultas son impracticables o requieren procesamiento previo costoso.

Ademas, los logs estructurados eliminan la variabilidad del formato. Cuando cada desarrollador escribe logs en su propio estilo, el analisis centralizado se vuelve una tarea de estandarizacion continua. Un esquema JSON comun, definido y documentado, garantiza que todos los componentes del sistema produzcan logs compatibles entre si, independientemente de quien los haya escrito o en que lenguaje este implementado.

