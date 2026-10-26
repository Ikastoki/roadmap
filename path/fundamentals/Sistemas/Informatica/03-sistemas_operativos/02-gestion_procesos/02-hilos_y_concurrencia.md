# Hilos y concurrencia

Los hilos (*threads*) son la unidad más pequeña de ejecución que puede ser planificada por el sistema operativo. Mientras que un proceso representa una instancia completa de un programa con su propio espacio de direcciones, un hilo es un flujo de ejecución dentro de un proceso que comparte el espacio de memoria con otros hilos del mismo proceso. Esta compartición de memoria hace que la comunicación entre hilos sea extremadamente rápida, pero también introduce complejidad en la gestión de la concurrencia.

La programación concurrente con hilos es una de las áreas más desafiantes de la informática, ya que los bugs relacionados con la concurrencia (condiciones de carrera, interbloqueos) son extremadamente difíciles de reproducir y diagnosticar. Comprender los fundamentos de los hilos y la concurrencia es esencial para diseñar aplicaciones que aprovechen eficientemente los procesadores multinúcleo modernos.

## Qué es un hilo

Un hilo es un flujo secuencial de instrucciones que se ejecuta dentro de un proceso. Cada proceso tiene al menos un hilo (el hilo principal), pero puede crear múltiples hilos que ejecutan simultáneamente不同的 partes del programa.

Los hilos comparten con el proceso padre:

- Espacio de direcciones de memoria (código, datos, heap).
- Archivos abiertos.
- Señales y manejadores de señales.
- ID de usuario y grupo.

Cada hilo tiene su propio:

- Conjunto de registros de la CPU (contador de programa, puntero de pila).
- Pila de ejecución (*stack*) para variables locales y llamadas a función.
- Estado (listo, ejecutando, esperando).
- ID de hilo.

## Proceso vs hilo: memoria compartida

La compartición de memoria entre hilos es tanto una ventaja como un desafío. La ventaja es la velocidad: dos hilos del mismo proceso pueden intercambiar datos simplemente escribiendo y leyendo de la misma ubicación de memoria, sin necesidad de mecanismos de comunicación complejos como tuberías o sockets.

El desafío es la sincronización: cuando dos hilos acceden simultáneamente a la misma ubicación de memoria y al menos uno de ellos realiza una escritura, el resultado puede ser impredecible. Este problema se conoce como *condición de carrera* (*race condition*).

```python
# Ejemplo de condición de carrera
counter = 0

# Hilo 1
def incrementar():
    global counter
    for _ in range(1000000):
        counter += 1  # Lectura + incremento + escritura (no atómico)

# Hilo 2
def incrementar():
    global counter
    for _ in range(1000000):
        counter += 1

# El resultado final puede ser cualquier valor entre 2 y 2,000,000
# porque las operaciones de counter += 1 se intercalan de forma aleatoria
```

Para evitar condiciones de carrera, se utilizan mecanismos de sincronización como mutex, semáforos y bloqueos (*locks*). Un mutex garantiza que solo un hilo puede acceder a una sección crítica de código a la vez, eliminando la posibilidad de intercalación de operaciones.

## Hilos a nivel de kernel vs nivel de usuario

**Hilos a nivel de kernel (*kernel threads*):** son gestionados directamente por el sistema operativo. Cada hilo kernel se mantiene en una estructura de datos del kernel y es planificado por el planificador de procesos del SO. Los hilos kernel pueden ejecutarse en paralelo en múltiples núcleos de CPU y son despertados por interrupciones del sistema.

**Hilos a nivel de usuario (*user threads*):** son gestionados por una biblioteca en el espacio de usuario, sin conocimiento del kernel. La biblioteca de hilos planifica la ejecución de los hilos de usuario en uno o más hilos kernel. Ventaja: la creación y conmutación de hilos de usuario es mucho más rápida. Desventaja: si un hilo de usuario realiza una operación de bloqueo (como una llamada al sistema de E/S), todos los hilos del proceso se bloquean.

**Modelos de implementación:**

- **1:1 (un hilo kernel por hilo de usuario):** Linux con pthreads y Windows NT. Cada hilo de usuario tiene un hilo kernel correspondiente. Máxima parallelism pero overhead de kernel por cada hilo.
- N:1 (múltiples hilos de usuario en un solo hilo kernel): hilos de usuario gestionados por una biblioteca. Mínimo overhead pero sin parallelism real.
- M:N (hilos de usuario mapeados a hilos kernel): combina ventajas de ambos modelos pero es complejo de implementar. Go con goroutines utiliza un modelo similar.

## Problemas de concurrencia

**Condición de carrera (*race condition*):** ocurre cuando el resultado de una operación depende del orden no determinista en que se ejecutan los hilos. Se resuelve con mecanismos de sincronización (mutex, semáforos, bloqueos).

**Interbloqueo (*deadlock*):** ocurre cuando dos o más hilos están bloqueados esperando recursos que están retenidos por otros hilos del mismo grupo, creando un ciclo de dependencias del que ninguno puede progresar.

```
Hilo A: tiene Recurso 1, espera Recurso 2
Hilo B: tiene Recurso 2, espera Recurso 1
→ Ninguno puede avanzar (interbloqueo)
```

Se resuelve evitando las cuatro condiciones necesarias para el interbloqueo: mutualidad, retención y espera, no preemptión y espera circular.

**Inanición (*starvation*):** un hilo nunca recibe tiempo de ejecución porque otros hilos de mayor prioridad lo superan continuamente.

> Los bugs de concurrencia son notoriamente difíciles de depurar porque dependen del timing exacto de la ejecución de los hilos, que varía entre ejecuciones. Herramientas como ThreadSanitizer (TSan) y helgrind pueden detectar condiciones de carrera y problemas de sincronización durante la ejecución de pruebas.

