# Correlacion de Logs

Tener los logs centralizados en un solo lugar escondicion necesario pero no suficiente. Cuando una peticion de usuario atraviesa seis microservicios, el sistema centralizado contiene las entradas de los seis, pero nada indica automaticamente que esas seis entradas corresponden a la misma operacion. Sin un mecanismo de correlacion, el operador obtiene un resultado de busqueda con miles de entradas que comparten el mismo rango temporal y no tiene forma de saber cual de esas entradas pertenece a su peticion. Buscar el camino completo de una sola solicitud entre millones de logs es como buscar una aguja en un pajar, y si ademas hay multiples pajares (diferentes servicios), la tarea se multiplica exponencialmente.

## Identificadores de correlacion

Un identificador de correlacion (correlation ID) es un valor unico y aleatorio que se genera en el punto de entrada del sistema (normalmente el API gateway o el load balancer) y se propaga a traves de toda la cadena de servicios que procesan la peticion. Cada servicio incluye este identificador en sus logs, lo que permite reconstruir la secuencia completa de eventos filtrando por ese unico valor.

El formato mas comun es un UUID v4, aunque en entornos de alta demanda de rendimiento se utilizan identificadores mas compactos como Ulid o incluso enteros secuenciales dentro de un dominio. Lo critico no es el formato sino la garantia de que el mismo identificador viaja intacto desde el borde hasta el componente mas profundo de la llamada.

La generacion y propagacion del correlation ID normalmente ocurre a nivel de middleware o filtros en el framework de cada servicio. El primer componente que recibe la peticion genera el ID y lo inserta en el contexto de la peticion. Cada servicio subsiguiente lo extrae del contexto y lo incluye en sus propios logs, metricas y trazas. Si un servicio llama a otro via HTTP, el ID se propaga como cabecera. Si la comunicacion es asincrona a traves de una cola de mensajes, el ID se incluye en los metadatos del mensaje.

## Contexto de trazado

El correlation ID es el concepto base, pero en sistemas maduros se combina con un contexto de trazado mas completo que no solo identifica la peticion, sino que tambien documenta la jerarquia de llamadas. En este modelo, cada operacion dentro de una peticion recibe un **span** (un tramo) con su propio identificador, y cada span referencia al span padre que lo origino. Esto crea un arbol de llamadas que representa exactamente que servicio llamo a que otro servicio y que operaciones internas se ejecutaron dentro de cada uno.

El estandar mas adoptado para este contexto es el definido por OpenTelemetry, que especifica tres identificadores en cada span: el trace ID (comun a toda la peticion), el span ID (unico para cada operacion), y el parent span ID (que enlaza con la operacion que origino esta). Esta estructura no solo permite buscar todos los logs de una peticion, sino tambien reconstruir el grafo de dependencias entre servicios y medir cuantitativamente el tiempo que cada tramo consume en el total.

La propagacion de este contexto entre servicios se realiza generalmente a traves de cabeceras HTTP estandarizadas. El formato W3C Trace Context, que define las cabeceras `traceparent` y `tracestate`, se ha convertido en el mecanismo interoperable por defecto. Cada servicio que recibe una peticion con estas cabeceras las procesa, genera sus propios spans, y las reenvia al siguiente servicio de la cadena si las hay.

## Busqueda de extremo a extremo

Con la correlacion implementada, el operador puede realizar busquedas de extremo a extremo que reconstruyen el camino completo de una peticion. En lugar de buscar "errores en el servicio de pagos", puede buscar "todos los logs con correlation ID abc-123" y ver exactamente que sucedio en cada punto del sistema: el API gateway accepto la peticion, el servicio de usuarios la autentico correctamente, el servicio de inventario verifico stock, el servicio de pagos proceso el cobro, y el servicio de notificaciones envio el email de confirmacion. Si hay un error, el operador ve inmediatamente en que servicio fallo y cual fue el contexto exacto del fallo.

Sin correlacion, la experiencia es fundamentalmente diferente. El operador debe intuir que los logs de un servicio estan relacionados con los de otro basandose en proximidad temporal y coincidencia parcial de campos como el ID de usuario o el ID de transaccion. Esto funciona en sistemas pequenos con poco trafico, pero se vuelve inmanejable cuando cada segundo procesan cientos de peticiones concurrentes que se mezclan en los logs de cada servicio.

> La correlacion de logs es el prerequisito operativo para el tracing distribuido. Sin ella, las herramientas de tracing pierden gran parte de su valor porque no pueden enlazar los spans con la evidencia textual que los logs proporcionan.

