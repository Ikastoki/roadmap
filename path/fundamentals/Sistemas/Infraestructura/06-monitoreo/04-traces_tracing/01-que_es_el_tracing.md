# Que es el tracing

En un sistema distribuido donde una sola peticion de usuario puede atravesar una docena de servicios, bases de datos y colas de mensajes, identificar donde se produce un retardo o un error es una tarea que desafia la capacidad humana de razonamiento secuencial. Las metricas告诉你 que algo va lento; los logs告诉你 que algo fallo; pero solo el tracing告诉你 exactamente por donde viajó la peticion, cuanto tiempo permanecio en cada servicio y que rel causado entre componentes. El tracing distribuido transforma la invisibilidad de las comunicaciones inter-servicio en un mapa completo y navegable del recorrido completo de cada peticion, desde que el cliente la inicia hasta que recibe la respuesta. Sin esta herramienta, un equipo de operaciones investigando una caida en produccion estaria基本mente adivinando entre docenas de servicios candidatos.

## La analogia del servicio postal

Imagina que envias una carta desde Barcelona hasta Tokio. La carta no viaja directamente: pasa por la oficina de correos local, un centro de distribucion regional, un hub internacional, aduanas, otra oficina regional en Japon y finalmente la oficina local de destino. Cada oficina por la que pasa le pone un sello con la fecha de entrada y salida, el operador que la gestionó y el destino al que la reenvia. Si la carta se retrasa, puedes mirar los sellos y descubrir que estuvo 3 dias bloqueada en aduanas, no 2 horas en el vuelo como sospechabas.

El tracing distribuido funciona exactamente igual. Una peticion HTTP que llega al gateway de una aplicacion es como la carta: cada servicio por el que pasa registra un registro de llegada y salida con metadatos relevantes. Estos registros se denominan spans, y el conjunto completo de spans de una peticion forma una traza (trace) que reconstruye el viaje completo de la peticion a traves del sistema.

```text
Cliente ──[peticion]──> API Gateway ──> Servicio Usuarios ──> Base de Datos
                           │                   │                    │
                         span A              span B              span C
                     (recibe, procesa,     (consulta user,      (query SQL,
                      responde)            busca cache)         retorna resultado)
```

## Span: la unidad de trabajo

Un span es la unidad atomica del tracing. Representa una operacion individual dentro de un servicio: una llamada HTTP, una consulta a base de datos, el procesamiento de un mensaje de una cola o cualquier otra accion que tenga un inicio y un fin claramente definidos. Cada span contiene varios campos obligatorios que lo hacen util para diagnostica:

- **Trace ID**: identificador unico que agrupa todos los spans de una misma peticion. Es el numero de seguimiento de la carta postal.
- **Span ID**: identificador unico de este span concreto. Es el sello de una oficina concreta.
- **Parent Span ID**: referencia al span que originó este span. Crea la jerarquia de la traza.
- **Nombre**: descripcion de la operacion, como `GET /api/users/:id` o `MySQL SELECT`.
- **Timestamp de inicio y fin**: determinan la duracion del span.
- **Estado**: OK, ERROR o UNSET, indicando si la operacion tuvo exito.
- **Atributos**: pares clave-valor con metadata contextual, como el metodo HTTP, el codigo de respuesta, el nombre de la tabla consultada o el identificante del usuario.

La duracion de un span se calcula como la diferencia entre su timestamp de inicio y su timestamp de fin. Cuando un span contiene otros spans dentro de si (llamadas a servicios secundarios o consultas a bases de datos), estos forman una relacion padre-hijo que se visualiza como un arbol jerarquico.

```text
Span raiz (API Gateway)
├── Span hijo 1 (Servicio Usuarios)
│   ├── Span nieto 1a (Cache Redis)
│   └── Span nieto 1b (Base de Datos MySQL)
└── Span hijo 2 (Servicio Pagos)
    └── Span nieto 2a (Pasarela de Pagos externa)
```

## Trace: el recorrido completo

Una traza es el conjunto completo de todos los spans generados por una unica peticion a traves del sistema distribuido. Mientras un span representa una operacion aislada, la traza representa la historia completa del viaje de la peticion. El trace ID es el hilo conductor que permite reunir todos los spans dispersos en multiples servicios y reconstruir el arbol completo de ejecucion.

Una traza comienza cuando el primer servicio recibe la peticion del cliente (o cuando se inicia una accion interna como un cron job). A partir de ese punto, cada vez que el servicio hace una llamada a otro componente, se crea un nuevo span con el mismo trace ID pero un nuevo span ID, referenciando al span padre. Este proceso continua hasta que la peticion regresa al servicio original y se envia la respuesta al cliente.

> Una traza no es simplemente una lista de spans; es un arbol dirigido donde la estructura padre-hijo revela las dependencias causales. Si el span de "Servicio Pagos" es hijo del span de "API Gateway", significa que Gateway llamó a Pagos, no al reves. Esta informacion causal es fundamental para diagnosticar problemas de rendimiento.

## Propagacion de contexto

El mecanismo que permite que los spans de distintos servicios se mantengan conectados en una misma traza se denomina propagacion de contexto. Cuando un servicio realiza una llamada a otro, debe transmitirle el trace ID y el parent span ID para que el servicio destinatario pueda continuar la traza correctamente. Sin esta propagacion, cada servicio generaria trazas aisladas e inconnexas, perdiendo la visibilidad del recorrido completo.

La propagacion mas comun se realiza a traves de headers HTTP. El servicio emisor incluye headers estandarizados como `traceparent` (definido por el estandar W3C Trace Context) en la peticion saliente. El servicio receptor lee estos headers y los utiliza para crear un span hijo que referencia al span padre del servicio emisor.

```text
Servicio A envia:
  POST /api/pagos HTTP/1.1
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

Servicio B recibe:
  Lee traceparent → trace_id = 4bf92f3577b34da6a3ce929d0e0e4736
                   → parent_span_id = 00f067aa0ba902b7
                   → Crea span hijo con ese parent
```

Este mecanismo de inyeccion/extraccion de headers es completamente transparente para la logica de negocio. Los servicios no necesitan conocer los detalles de la propagacion; solo necesitan tener habilitada la instrumentacion correspondiente. Sin embargo, el propagador debe instalarse explicitamente en cada punto de comunicacion inter-servicio, incluyendo clientes HTTP, bibliotecas de gRPC, middlewares de colas de mensajes y cualquier otro canal de comunicacion.

Los formatos de propagacion mas extendidos son el header `traceparent` del estandar W3C y formatos proprietary como B3 de Zipkin. El ecosistema OpenTelemetry soporta ambos y permite configurar el formato segun la infraestructura existente.

> Un error comun en sistemas distribuidos es olvidar instrumentar algun punto de comunicacion. Si un servicio usa una libreria HTTP personalizada que no esta instrumentada, los headers de contexto no se propagan y la traza se rompe en ese punto, creando una brecha de visibilidad que puede ser muy dificil de identificar sin una revision exhaustiva de la propagacion.

