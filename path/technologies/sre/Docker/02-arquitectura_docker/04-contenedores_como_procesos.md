# Contenedores como procesos

La comprension mas profunda de los contenedores surge cuando se les reconoce por lo que realmente son: procesos del sistema anfitrion, limitados y aislados, pero procesos al fin y al cabo. No hay magia ni virtualizacion; un contenedor en ejecucion es simplemente un proceso con un conjunto de propiedades especiales configuradas por el kernel. Esta perspectiva tiene implicaciones practicas para la gestion de recursos, la senalizacion, la terminacion de procesos y la supervision de contenedores.

## Los contenedores son procesos del anfitrion

Cuando se ejecuta `docker run`, el proceso resultante es visible en el sistema anfitrion como cualquier otro proceso. Se puede ver con `ps`, `top` o `htop`, y se puede inspeccionar con las herramientas estandar de Linux.

```bash
# Ejecutar un contenedor simple
docker run -d --name sleep alpine sleep 300

# Ver el proceso en el anfitrion
ps aux | grep sleep
# root     12345  0.0  0.0   1552   4 ?        Ss   10:00   0:00 sleep 300

# Ver el PID del proceso del contenedor
docker inspect sleep --format '{{.State.Pid}}'
# 12345

# El proceso del contenedor es un proceso real del anfitrion
ls -la /proc/12345/
```

El PID que Docker muestra dentro del contenedor (generalmente 1) es diferente del PID real en el anfitrion. Esto es debido al namespace de PID, que aisla la numeracion de procesos. Dentro del contenedor, el proceso init ve el PID 1; en el anfitrion, tiene un PID normal y corriente.

```bash
# Dentro del contenedor, el proceso ve PID 1
docker exec sleep ps aux
# PID   USER     TIME  COMMAND
# 1     root     0:00 sleep 300

# En el anfitrion, el mismo proceso tiene otro PID
docker inspect sleep --format '{{.State.Pid}}'
# 12345 (por ejemplo)
```

## El proceso init del contenedor

Cada contenedor ejecuta un proceso principal, que es el que se define en el CMD o ENTRYPOINT del Dockerfile. Este proceso recibe el PID 1 dentro del contenedor y tiene responsabilidades especiales, como manejar las senales del sistema y recolectar procesos zombies.

```bash
# El proceso principal recibe senales del sistema
# Docker pasa SIGTERM al proceso PID 1 cuando se ejecuta docker stop
docker stop sleep
# Docker envia SIGTERM, espera 10s, y si no termina, envia SIGKILL
```

El proceso PID 1 en un contenedor debe estar disenado para manejar senales correctamente. Si el proceso no maneja SIGTERM, `docker stop` esperara el timeout y luego enviara SIGKILL, lo que puede provocar una terminacion abrupta con perdida de datos.

```dockerfile
# Buen diseno: el proceso maneja senales
FROM node:20-alpine
CMD ["node", "server.js"]
# Node.js maneja SIGTERM correctamente, permitiendo un apagado graceful

# Mal diseno: el proceso no maneja senales
FROM alpine
CMD ["sh", "-c", "while true; do echo 'hola'; sleep 1; done"]
# sh no propaga senales, docker stop sera forzado
```

## Propagacion de senales

Docker propaga las senales del anfitrion al proceso principal del contenedor. Cuando se ejecuta `docker stop`, Docker envia SIGTERM al PID 1 del contenedor. Si el proceso no termina en 10 segundos (configurable), Docker envia SIGKILL.

```bash
# Personalizar el timeout de parada
docker stop --time=30 sleep

# Enviar senales especificas al contenedor
docker kill --signal=HUP nginx
# Esto envia SIGHUP a nginx, que tipicamente recarga la configuracion

# Enviar cualquier senal
docker kill --signal=USR1 mi-app
```

Es importante entender que `docker kill` no es una terminacion forzada por defecto; simplemente envia la senal especificada (SIGKILL si no se especifica). `docker stop` envia SIGTERM primero, espera, y luego SIGKILL.

## Procesos zombies y su gestion

Un problema comun en contenedores que ejecutan multiples procesos es la acumulacion de procesos zombies. Cuando un proceso hijo termina y su padre no espera por el (no ejecuta `wait()`), el proceso hijo se convierte en zombie. En un contenedor con un solo proceso principal que no gestiona hijos, los zombies pueden acumularse.

```bash
# Ver procesos zombies en un contenedor
docker exec contenedor ps aux | grep Z
# Si aparecen procesos en estado Z, hay zombies
```

Para evitar zombies, se recomienda:
- Usar una imagen base que incluya un init ligero (como `tini` o `dumb-init`)
- Ejecutar un unico proceso por contenedor
- Si es necesario ejecutar multiples procesos, usar un supervisor que gestione hijos

```dockerfile
# Usar tini como init del contenedor
FROM alpine:3.18
RUN apk add --no-cache tini
COPY app /app
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["/app/mi-app"]

# tini gestiona la recoleccion de zombies y la propagacion de senales
```

## Namespace de PID y procesos visibles

El aislamiento del namespace de PID determina que procesos puede ver un contenedor. Por defecto, un contenedor solo ve sus propios procesos. Sin embargo, es posible compartir el namespace de PID con el anfitrion o con otros contenedores.

```bash
# Compartir el namespace de PID con el anfitrion
docker run --pid=host alpine ps aux
# Ve todos los procesos del sistema

# Compartir el namespace de PID con otro contenedor
docker run --pid=container:otro-contenedor alpine ps aux
# Ve los procesos del otro contenedor
```

Compartir el namespace de PID tiene implicaciones de seguridad y funcionalidad. Permitir que un contenedor vea los procesos del anfitrion puede ser util para herramientas de monitoreo, pero tambien expone informacion sensible.

## Recursos y limites a nivel de proceso

Los cgroups imponen limites a nivel de grupo de procesos, no de contenedor individual. Esto significa que se pueden aplicar limites a un grupo de procesos que no necesariamente coinciden con un contenedor.

```bash
# Ver los limites del cgroup del contenedor
docker run -d --memory="256m" --name limited nginx

# El contenedor (y todos sus procesos) estan limitados a 256 MB
# Si el contenedor lanza procesos hijos, todos comparten el mismo limite
```

Cuando un proceso dentro del contenedor excede el limite de memoria, el OOM killer del kernel puede terminarlo. Docker configura el parametro `memory.oom_control` para que el OOM killer termine el proceso que mas memoria consume dentro del cgroup.

## El contenedor como proceso: implicaciones practicas

Entender que los contenedores son procesos tiene implicaciones directas en como se disenan y operan:

1. **Un contenedor, un proceso**: Ejecutar multiples procesos en un contenedor complica la gestion de senales, la recoleccion de zombies y el escalado. La practica recomendada es un proceso por contenedor.

2. **Los logs son stdout/stderr**: Los contenedores deben escribir sus logs en la salida estandar. Docker captura stdout y stderr y los expone a traves de `docker logs`.

```bash
# La aplicacion debe escribir logs en stdout
docker logs -f mi-contenedor
```

3. **La terminacion del proceso principal termina el contenedor**: Si el proceso principal termina (por cualquier razon), el contenedor termina. El estado de salida del proceso se convierte en el estado de salida del contenedor.

```bash
# Ver el codigo de salida de un contenedor terminado
docker ps -a | grep mi-contenedor
docker inspect mi-contenedor --format '{{.State.ExitCode}}'
```

4. **Los recursos se limitan por contenedor, no por proceso**: Todos los procesos dentro del mismo contenedor comparten los mismos limites de cgroups.

> Tratar los contenedores como procesos, no como maquinas virtuales, es el cambio de paradigma mas importante que debe hacer un administrador de sistemas al adoptar contenedores. No se "conecta" a un contenedor para "arreglar las cosas"; se reconstruye y se reemplaza. No se "instalan paquetes" dentro de un contenedor en ejecucion; se actualiza el Dockerfile y se reconstruye la imagen. Esta mentalidad de procesos efimeros es la clave para aprovechar todo el potencial de Docker.

