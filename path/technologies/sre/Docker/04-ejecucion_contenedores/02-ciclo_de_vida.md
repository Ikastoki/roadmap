# Ciclo de vida de un contenedor

Un contenedor no es un ente estatico; atraviesa una secuencia de estados desde que se crea hasta que se elimina. Comprender esta maquina de estados es esencial para diagnosticar problemas, gestionar recursos y disenar flujos de trabajo eficientes. Cada transicion entre estados responde a una accion explicita del usuario o a un evento interno del sistema, como la terminacion del proceso principal.

```
Created -> Running -> Paused -> Stopped -> Deleted
                  \-> Stopped (por fallo) -> Deleted
```

## Created

Cuando se ejecuta `docker create`, Docker prepara el sistema de archivos del contenedor a partir de las capas de la imagen, configura los namespaces y los cgroups, pero no inicia ningun proceso. El contenedor existe como una entidad en el sistema Docker pero no esta ejecutandose. Es util para preparar un contenedor y arrancarlo mas tarde con `docker start`.

```bash
# Crear sin ejecutar
docker create --name mi-web -p 8080:80 nginx
docker start mi-web
```

## Running

Al ejecutar `docker run` o `docker start`, Docker lanza el proceso definido en el CMD o ENTRYPOINT de la imagen. El contenedor esta en estado running mientras ese proceso permanezca activo. Docker monitoriza constantemente el PID 1 del contenedor; si este proceso termina, el contenedor pasa automaticamente a estado stopped.

```bash
docker run -d --name mi-app alpine sleep 60
# El contenedor estara running durante 60 segundos
```

## Paused

El estado paused detiene todos los procesos del contenedor mediante la senal SIGSTOP, que el kernel de Linux maneja de forma inmediata. A diferencia de stopped, paused no libera la memoria ni cierra los descriptores de archivo; simplemente congela la ejecucion. Es util para realizar snapshots o liberar CPU temporalmente sin perder el estado del contenedor.

```bash
docker pause mi-app
# El contenedor no consume CPU pero mantiene la memoria asignada

docker unpause mi-app
# Los procesos reanudan su ejecucion
```

## Stopped

El contenedor entra en estado stopped cuando el proceso PID 1 termina, ya sea por completar su trabajo, por recibir una senal de terminacion (`docker stop` envia SIGTERM seguido de SIGKILL) o por un error interno. En este estado, la capa de escritura del contenedor persiste en disco, pero el proceso no se ejecuta y la memoria asignada se libera.

```bash
docker stop mi-app
# Docker envia SIGTERM, espera 10 segundos y envia SIGKILL

docker kill mi-app
# Envia SIGKILL inmediatamente sin tiempo de gracia
```

## Deleted

Al ejecutar `docker rm`, Docker elimina la capa de escritura del contenedor, libera sus recursos asociados y elimina su metadata del sistema. Un contenedor detenido puede eliminarse en cualquier momento, pero un contenedor en ejecucion debe detenerse primero a menos que se use `docker rm -f`.

```bash
docker rm mi-app       # Eliminar un contenedor detenido
docker rm -f mi-app    # Forzar la eliminacion de uno en ejecucion
```

## PID 1 y la muerte del contenedor

Cada contenedor ejecuta un proceso principal que recibe el identificador de proceso 1 dentro del namespace de PID del contenedor. Este proceso tiene responsabilidades especiales: debe manejar las senales del sistema y, cuando termina, el kernel finaliza todos los procesos hijos restantes y detiene el contenedor. Si el PID 1 termina con codigo 0, el contenedor se detiene limpiamente; si termina con codigo distinto de cero, Docker reporta el codigo de salida como un fallo.

```bash
# Ver el codigo de salida de un contenedor
docker inspect mi-app --format '{{.State.ExitCode}}'
```

> Si el proceso principal del contenedor no maneja SIGTERM correctamente, Docker espera 10 segundos y envia SIGKILL. Es buena practica que las aplicaciones dentro del contenedor capturen SIGTERM para realizar una parada ordenada.

## Transiciones entre estados

| Accion                    | Estado origen | Estado destino |
|---------------------------|---------------|----------------|
| `docker create`           | -             | Created        |
| `docker start`            | Created       | Running        |
| `docker run`              | -             | Running        |
| `docker pause`            | Running       | Paused         |
| `docker unpause`          | Paused        | Running        |
| `docker stop`             | Running       | Stopped        |
| `docker kill`             | Running       | Stopped        |
| `docker restart`          | Running       | Running        |
| `docker rm`               | Stopped       | Deleted        |
| `docker rm -f`            | Running       | Deleted        |
| PID 1 termina             | Running       | Stopped        |

Un contenedor stopped no pierde sus datos ni su configuracion; la capa de escritura permanece intacta. Arrancarlo de nuevo con `docker start` reactiva el proceso PID 1 con el mismo sistema de archivos, las mismas variables de entorno y la misma configuracion de red.

```bash
# Ciclo completo
docker run -d --name demo alpine sleep 30
docker pause demo
docker unpause demo
docker stop demo
docker start demo  # reactiva el mismo contenedor
docker rm demo
```

> Los contenedores detenidos ocupan espacio en disco (su capa de escritura) y entradas en la tabla de contenedores de Docker. Es recomendable eliminarlos periodicamente con `docker container prune` cuando ya no sean necesarios.

