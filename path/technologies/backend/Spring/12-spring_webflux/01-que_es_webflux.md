# Que es WebFlux

El modelo tradicional de programacion en Java, basado en el paradigma request-response con un hilo por cada peticion, presenta limitaciones claras cuando el volumen de conexiones concurrentes supera la capacidad del pool de hilos. Spring WebFlux introduce un modelo de programacion reactiva que permite manejar miles de conexiones concurrentes consumiendo recursos del sistema de forma eficiente, especialmente en escenarios de alta concurrencia con operaciones de entrada/salida intensivas.

La programacion reactiva se fundamenta en el paradigma de flujo orientado por eventos (*dataflow*), donde las transformaciones de datos se propagan automaticamente a traves de una cadena de operadores. En lugar de bloquer el hilo mientras se espera una respuesta de red o base de datos, el sistema registraCallbacks que se ejecutaran cuando el dato este disponible. Este enfoque libera los hilos del contenedor para procesar otras peticiones mientras la operacion de E/S se completa en segundo plano.

WebFlux implementa la especificacion **Reactive Streams**, un estandar publicado como JSR-365 que define una API estandar para la transmision asincrona de secuencias de datos con backpressure controlado. La especificacion consta de cuatro interfaces fundamentales:

- `Publisher<T>`: emisor que produce elementos de tipo `T`
- `Subscriber<T>`: receptor que consume los elementos
- `Subscription`: enlace entre publisher y subscriber que permite controlar la tasa de emision
- `Operator`: transformador que adapta los flujos entre componentes

Detras de WebFlux opera **Project Reactor**, la implementacion de referencia de Reactive Streams para el ecosistema Java. Reactor proporciona dos tipos de secuencias reactivas fundamentales:

`Mono<T>` representa un valor reactivo que emitira **cero o un elemento**, seguido opcionalmente de un evento de completado o error. Es la abstraccion mas comun para operaciones que producen un unico resultado, como una consulta a una base de datos que retorna un registro, una llamada a un servicio externo o una operacion CRUD.

```java
Mono<User> user = userRepository.findById(1L);
Mono<String> greeting = user.map(u -> "Hola " + u.getNombre());
```

`Flux<T>` representa un flujo reactivo que emite **cero a N elementos**, terminando con un evento de completado o error. Se utiliza para colecciones, secuencias temporales, eventos de suscripcion o cualquier escenario que produzca multiples valores a lo largo del tiempo.

```java
Flux<Product> products = productRepository.findAllByCategoria("electronica");
Flux<ProductDTO> dtos = products.map(this::toDTO);
```

Ambos tipos implementan la interfaz `org.reactivestreams.Publisher`, lo que garantiza compatibilidad total con la especificacion Reactive Streams. Las operaciones sobre `Mono` y `Flux` se componen de forma declarativa mediante una amplia catalogo de operadores: `map`, `flatMap`, `filter`, `zip`, `merge`, `switchIfEmpty`, `onErrorResume`, entre otros.

La composabilidad de estos operadores permite construir pipelines complejos de procesamiento de datos de forma legible y mantenible. Cada operacion retorna un nuevo `Mono` o `Flux`, lo que favorece un estilo de programacion funcional libre de efectos colaterales en la cadena de transformacion.

> La ejecucion diferida es un principio clave: el pipeline no se ejecuta hasta que alguien lo suscriba. Esto permite construir flujos complejos de forma declarativa y optimizar la composicion antes de que se produzca cualquier efecto real.

Spring WebFlux ofrece dos modelos de programacion para definir endpoints reactivos: el modelo basado en anotaciones similar a Spring MVC (utilizando `@RestController`), y el modelo funcional basado en `RouterFunction` y `HandlerFunction`. Ambos modelos comparten la misma infraestructura subyacente y operan sobre el mismo motor de ejecucion reactivo.

El servidor subyacente puede ser **Netty** (el predeterminado) o cualquier otro servidor compatible con Reactive Streams, como Servlet 3.1+ con soporte para non-blocking I/O. Netty gestiona el ciclo de vida de las conexiones y la multiplexacion de canales sin requerir un hilo dedicado por conexion, lo que permite escalar horizontalmente con un consumo de memoria significativamente menor que el modelo thread-per-request.

