# Comunicación entre procesos

La comunicación entre procesos (*Inter-Process Communication*, IPC) es el conjunto de mecanismos que permiten a los procesos intercambiar datos y coordinarse entre sí. En un sistema multitarea, los procesos a menudo necesitan cooperar para completar una tarea: un proceso puede generar datos que otro proceso consume, o múltiples procesos pueden necesitar sincronizarse para acceder a un recurso compartido.

Los mecanismos de IPC varían en complejidad, rendimiento y escalabilidad. La elección del mecanismo adecuado depende de los requisitos de la aplicación: si los procesos están en la misma máquina o en diferentes, si se necesita transferencia de datos de alta velocidad o comunicación estructurada, y si se requiere sincronización o solo transferencia.

## Pipes y named pipes

**Pipe (tubería):** es el mecanismo de IPC más simple y antiguo de Unix. Un pipe es un canal unidireccional que conecta la salida de un proceso con la entrada de otro. Los datos fluyen en una sola dirección, del proceso escritor al proceso lector.

```bash
# En shell: el pipe conecta la salida de ls con la entrada de grep
ls -la | grep ".txt"
```

Los pipes anónimos (*anonymous pipes*) solo existen durante la vida útil de los procesos conectados y solo funcionan entre procesos padre-hijo o entre procesos hermanos creados con `fork()`.

**Named pipe (FIFO):** es un pipe que tiene un nombre en el sistema de archivos, lo que permite que procesos no relacionados se comuniquen a través de él. Los named pipes persisten en el sistema de archivos y se comportan como archivos especiales que permiten E/S bidireccional.

## Mensajes y colas de mensajes

Las colas de mensajes permiten a los procesos enviar y recibir mensajes con estructura, cada uno con un tipo y prioridad. A diferencia de los pipes, las colas de mensajes no requieren que el emisor y el receptor estén activos al mismo tiempo: los mensajes se almacenan en la cola hasta que el receptor los lea.

Linux implementa colas de mensajes POSIX (`mq_open`, `mq_send`, `mq_receive`) que proporcionan una interfaz estandarizada para la comunicación por mensajes. Cada mensaje tiene una prioridad, y los procesos pueden seleccionar mensajes por tipo o prioridad en lugar de leerlos secuencialmente.

## Memoria compartida

La memoria compartida es el mecanismo de IPC más rápido, ya que permite que múltiples procesos accedan a la misma región de memoria sin necesidad de copiar datos entre espacios de direcciones. Los procesos mapean una región de memoria compartida en su propio espacio de direcciones y leen/escriben directamente en ella.

```c
// Crear memoria compartida
int shmid = shmget(IPC_PRIVATE, 4096, IPC_CREAT | 0666);
void *ptr = shmat(shmid, NULL, 0);

// Proceso 1: escribir
strcpy(ptr, "Hola desde proceso 1");

// Proceso 2: leer
printf("%s\n", (char *)ptr);
```

La memoria compartida es extremadamente rápida pero requiere mecanismos de sincronización (semáforos, mutex) para evitar condiciones de carrera. Un solo buffer compartido sin sincronización puede producir datos inconsistentes si ambos procesos escriben simultáneamente.

## Sockets de dominio Unix

Los sockets de dominio Unix (*Unix domain sockets*) permiten la comunicación bidireccional entre procesos en la misma máquina, utilizando un archivo especial del sistema de archivos como punto de conexión. Ofrecen las mismas semánticas que los sockets de red pero con menor overhead, ya que los datos no necesitan traversar la pila de protocolos de red.

```
Servidor:                          Cliente:
socket(AF_UNIX, SOCK_STREAM, 0)    socket(AF_UNIX, SOCK_STREAM, 0)
bind(fd, &addr, ...)               connect(fd, &addr, ...)
listen(fd, ...)                    write(fd, "hola", 4)
accept(fd, ...)                    read(fd, buf, ...)
read(fd, buf, ...)
write(fd, "adiós", 5)
```

Los sockets de dominio Unix son ampliamente utilizados por servicios del sistema (D-Bus, systemd) y por bases de datos locales (MySQL, PostgreSQL) donde la comunicación por red es innecesaria.

## Señales

Las señales son el mecanismo de IPC más antiguo y simple de Unix. Una señal es un mensaje de un bit que notifica a un proceso de que ha ocurrido un evento. Las señales son asincrónicas: pueden llegar en cualquier momento durante la ejecución del proceso.

Las señales más comunes incluyen:

- **SIGINT (2):** interrupción (Ctrl+C).
- **SIGKILL (9):** terminación forzada (no puede ser ignorada).
- **SIGTERM (15):** terminación amigable (puede ser manejada por el proceso).
- **SIGSEGV (11):** violación de segmento (acceso a memoria inválida).
- **SIGCHLD (17):** un proceso hijo ha terminado.
- **SIGALRM (14):** expiración de un temporizador.

Las señales son útiles para notificar eventos simples pero no son adecuadas para transferir datos. Para comunicación estructurada, los pipes, colas de mensajes o sockets son preferibles.

> Las señales en Linux son notoriamente difíciles de manejar correctamente porque pueden interrumpir cualquier punto de la ejecución del proceso. Los manejadores de señales deben ser.async-signal-safe: no pueden llamar a la mayoría de las funciones de la biblioteca C, lo que limita severamente lo que pueden hacer.

