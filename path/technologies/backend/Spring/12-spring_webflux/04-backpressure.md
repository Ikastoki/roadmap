# Backpressure

La backpressure es el mecanismo de control de flujo que permite al consumidor comunicar al productor la velocidad a la que puede procesar datos. En un sistema reactivo, cuando un productor emite elementos a una velocidad mayor de la que el consumidor puede procesar, se produce un desbalance que puede provocar consumo excesivo de memoria, perdida de datos o colapso del sistema. La backpressure resuelve este problema permitiendo que el consumidor regule la tasa de emision del productor.

Dentro de la especificacion Reactive Streams, la backpressure se gestiona a traves de la `Subscription`. Cuando un `Subscriber` se suscribe a un `Publisher`, recibe una `Subscription` con la que puede solicitar un numero especifico de elementos mediante `request(n)`. El publisher solo emite la cantidad de elementos solicitados, esperando a recibir una nueva solicitud antes de continuar.

```java
Flux.range(1, 1000)
    .subscribe(
        valor -> procesar(valor),
        error -> log.error("Error", error),
        () -> log.info("Completado")
    );
```

En Project Reactor, la backpressure se gestiona automaticamente. Los flujos con fuente de velocidad limitada (como `Flux.range` o `Flux.interval`) emiten elementos solo cuando el subscriber los solicita. Sin embargo, cuando la fuente es inherentemente rapida (como un `FluxSink` o un flux basado en eventos), es necesario definir una estrategia de backpressure.

## Estrategias de backpressure

Reactor proporciona varios operadores para gestionar el comportamiento cuando la velocidad de emision supera la de consumo:

**`onBackpressureBuffer(maxSize)`** almacena los elementos pendientes de procesar en un buffer interno. Cuando el buffer alcanza su capacidad maxima, comportamiento configurable: puede descartar el elemento mas antiguo, lanzar un error o invocar un callback.

```java
Flux.create(sink -> {
    for (int i = 0; i < 10000; i++) {
        sink.next(i);
    }
    sink.complete();
}, FluxSink.OverflowStrategy.LATEST)
    .onBackpressureBuffer(256, dropped ->
        log.warn("Elemento descartado por buffer lleno: {}", dropped),
        BufferOverflowStrategy.DROP_OLDEST)
    .delayElements(Duration.ofMillis(1))
    .subscribe(valor -> procesar(valor));
```

**`onBackpressureDrop()`** descarta los elementos que el consumidor no puede procesar en el momento de la emision. Es util cuando la informacion mas reciente es la relevante y los datos intermedios pueden perderse sin impacto.

```java
Flux.create(sink -> {
    for (int i = 0; i < 10000; i++) {
        sink.next(i);
    }
    sink.complete();
})
    .onBackpressureDrop(dropped ->
        log.debug("Elemento descartado: {}", dropped))
    .delayElements(Duration.ofMillis(1))
    .subscribe(valor -> procesar(valor));
```

**`onBackpressureLatest()`** conserva solo el ultimo elemento emitido cuando el consumidor no esta listo. Los elementos intermedios se descartan. Es la estrategia mas eficiente en terminos de memoria cuando solo importa el estado mas reciente.

```java
Flux.create(sink -> {
    for (int i = 0; i < 10000; i++) {
        sink.next(i);
    }
    sink.complete();
})
    .onBackpressureLatest()
    .delayElements(Duration.ofMillis(1))
    .subscribe(valor -> procesar(valor));
```

> La eleccion de la estrategia de backpressure depende del dominio: `buffer` cuando todos los datos son importantes, `latest` cuando solo importa el estado actual (como sensores IoT), y `drop` cuando el dato mas reciente reemplaza completamente al anterior.

## Gestion de requests

El operador `request()` permite un control manual de la suscripcion. En lugar de delegar la gestion de backpressure al framework, se puede solicitar explicitamente un numero de elementos al publisher.

```java
Flux.range(1, 100)
    .subscribe(new Subscriber<>() {
        private Subscription subscription;
        private int procesados = 0;

        @Override
        public void onSubscribe(Subscription s) {
            this.subscription = s;
            subscription.request(10); // Solicitar los primeros 10
        }

        @Override
        public void onNext(Integer valor) {
            procesar(valor);
            procesados++;
            if (procesados % 10 == 0) {
                subscription.request(10); // Solicitar 10 mas cada 10 procesados
            }
        }

        @Override
        public void onError(Throwable t) { log.error("Error", t); }

        @Override
        public void onComplete() { log.info("Completado"); }
    });
```

Este patron es util cuando se necesita adaptar la velocidad de procesamiento a un recurso limitado, como una base de datos que solo puede procesar un numero finito de operaciones concurrentes.

## Configuracion del servidor

En el contexto de un servidor Netty que sirve WebFlux, la configuracion de backpressure afecta a todo el pipeline de E/S. Netty gestiona internamente un buffer de recepcion que puede configurarse para adaptarse a las caracteristicas del trafico esperado.

```yaml
spring:
  reactor:
    netty:
      io-worker-count: 8
  main:
    lazy-initialization: true
```

El operador `limitRate(n)` controla la velocidad a la que un flux upstream emite elementos hacia downstream. Internamente, `limitRate` solicita el triple de elementos al upstream (`n * 3`) para amortiguar latencias de red, pero solo entrega `n` elementos al subscriber. Esto crea un efecto de buffering automatico que suaviza las fluctuaciones de velocidad.

```java
Flux.interval(Duration.ofMillis(1))
    .limitRate(100) // Emitir como maximo 100 elementos por ciclo
    .flatMap(this::procesarElemento)
    .subscribe();
```

La backpressure es una pieza fundamental de la arquitectura reactiva. Sin ella, un sistema reactivo no es mas que un sistema asincrono convencional con riesgo de desbordamiento de buffers. Su correcta implementacion garantiza estabilidad bajo carga variable y protege los recursos del sistema de picos inesperados de trafico.

