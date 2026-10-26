# Contenedores como procesos

Un contenedor no es una maquina virtual, ni un sandbox, ni un entorno aislado misterioso. Un contenedor es, en su esencia, un proceso del sistema operativo anfitrion que tiene aislamiento adicional mediante namespaces y limites de recursos mediante cgroups. Comprender este concepto es fundamental para diagnosticar problemas, entender el rendimiento y trabajar correctamente con contenedores.

## Un contenedor es un proceso

```bash
# Ejecutar un contenedor
docker run -d --name mi-nginx nginx:alpine

# En el host, ver el proceso del contenedor
ps aux | grep nginx
# root  12345  nginx: master process nginx -g daemon off;
# root  12346  nginx: worker process

# El PID 12345 es un proceso normal del kernel de Linux
# Tiene su propio PID namespace (contenedor ve PID 1; host ve PID 12345)
# Tiene su propio network namespace (IP separada)
# Tiene su propio mount namespace (sistema de archivos aislado)
```

## PID 1 dentro del contenedor

Dentro del contenedor, el proceso principal tiene PID 1. Este proceso es el que mantiene vivo al contenedor: si PID 1 termina, el contenedor se detiene.

```bash
# Dentro del contenedor:
docker exec mi-nginx ps aux
# PID   USER     COMMAND
#   1   root     nginx: master process nginx -g daemon off;
#   7   root     nginx: worker process

# En el host:
ps aux | grep "nginx: master"
# root  12345  nginx: master process nginx -g daemon off;
# El host ve el PID real (12345), no el PID 1 del contenedor
```

## Procesos huérfanos

Si el proceso principal (PID 1) del contenedor termina, el contenedor se detiene, incluso si hay otros procesos corriendo dentro.

```bash
# Ejecutar contenedor con proceso interactivo
docker run -it ubuntu bash

# Dentro del contenedor, salir con exit
exit
# El contenedor se detiene porque PID 1 (bash) termino
```

## Multiples procesos en un contenedor

Aunque un contenedor puede ejecutar multiples procesos, la recomendacion es un proceso por contenedor. Esto facilita el escalado, los logs y el monitoreo.

```bash
# Contenedor con multiples procesos (no recomendado)
docker run -d ubuntu sh -c "while true; do echo hello; sleep 10; done & nginx -g 'daemon off;'"

# Contenedor con un solo proceso (recomendado)
docker run -d nginx
# Un solo proceso: nginx
```

## Procesos del host vs. procesos del contenedor

```bash
# En el host, ver todos los procesos de contenedores
ps aux | grep -E "runc|containerd-shim"

# Cada contenedor tiene un containerd-shim que lo gestiona
# El shim es el padre del proceso del contenedor
# Si el daemon de Docker muere, los shim siguen ejecutandose
# Los contenedores no se detienen

# Ver la jerarquia de procesos
pstree -p $(docker inspect --format '{{.State.Pid}}' mi-nginx)
# systemd(1)───dockerd(1234)───containerd(1235)───containerd-shim(1236)───nginx(1237)
```

## Implicaciones de seguridad

Como un contenedor es un proceso del host, las vulnerabilidades del kernel pueden afectar a todos los contenedores. Un contenedor que escapa del aislamiento de namespaces puede acceder al host.

```bash
# Un contenedor con privilegios excesivos puede comprometer el host
docker run --privileged -it ubuntu bash
# Este contenedor tiene acceso casi total al hardware del host
# NUNCA usar --privileged en produccion sin justificacion
```

> Entender que un contenedor es un proceso del host con aislamiento adicional elimina la magia y permite tomar decisiones informadas de seguridad, rendimiento y arquitectura. No es un mundo aislado; es un proceso con mejores candados que un proceso normal.

