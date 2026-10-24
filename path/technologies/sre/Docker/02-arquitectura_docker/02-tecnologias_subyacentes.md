# Tecnologias subyacentes de Docker

Docker se apoya en un conjunto de tecnologias del kernel de Linux que, aunque no son exclusivas de Docker, son esenciales para su funcionamiento. Namespaces, cgroups, capacidades del kernel, sistemas de archivos union y calles de red virtual conforman la base sobre la que Docker construye su capa de abstraccion. Comprender estas tecnologias a nivel de sistema operativo es lo que distingue a un usuario avanzado de Docker de alguien que simplemente ejecuta comandos sin entender que ocurre bajo la superficie.

## Namespaces de Linux

Los namespaces son la tecnologia fundamental que proporciona aislamiento a los contenedores. Cada namespace envuelve un recurso global del sistema y lo presenta como privado para los procesos que pertenecen a ese namespace. Docker utiliza seis tipos de namespaces para aislar cada contenedor.

```bash
# Ver los namespaces de un proceso
ls -la /proc/$$/ns/
# total 0
# dr-x--x--x 2 user user 0 ... .
# dr-xr-xr-x 2 user user 0 ... ..
# lrwxrwxrwx 1 user user 0 ... cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx 1 user user 0 ... ipc -> 'ipc:[4026531839]'
# lrwxrwxrwx 1 user user 0 ... mnt -> 'mnt:[4026531841]'
# lrwxrwxrwx 1 user user 0 ... net -> 'net:[4026531993]'
# lrwxrwxrwx 1 user user 0 ... pid -> 'pid:[4026531836]'
# lrwxrwxrwx 1 user user 0 ... user -> 'user:[4026531837]'
# lrwxrwxrwx 1 user user 0 ... uts -> 'uts:[4026531838]'
```

Cada tipo de namespace aisla un recurso especifico:

- **PID namespace**: Los procesos dentro del contenedor ven su propio arbol de procesos, con PID 1 para el proceso inicial. No pueden ver procesos fuera del contenedor.
- **Network namespace**: Cada contenedor tiene sus propias interfaces de red, tablas de rutas y reglas de iptables, independientes del anfitrion y de otros contenedores.
- **Mount namespace**: El contenedor tiene su propio arbol de montajes del sistema de archivos. Los montajes realizados dentro del contenedor no afectan al anfitrion.
- **UTS namespace**: El contenedor puede tener su propio hostname y dominio, independiente del anfitrion.
- **IPC namespace**: Los recursos de comunicacion entre procesos (colas de mensajes, semaforos, memoria compartida) estan aislados.
- **User namespace**: Permite mapear el usuario root dentro del contenedor (UID 0) a un usuario no privilegiado en el anfitrion, mejorando la seguridad.

```bash
# Ver los namespaces de un contenedor especifico
docker inspect --format '{{.State.Pid}}' mi-contenedor
# 12345
ls -la /proc/12345/ns/
```

Cuando dos contenedores comparten el mismo namespace (por ejemplo, usando `--network container:otro`), pueden compartir recursos de red. Docker configura los namespaces durante la creacion del contenedor, y `runc` es el encargado de activarlos antes de ejecutar el proceso.

## Cgroups: control de recursos

Los cgroups (control groups) limitan, contabilizan y aíslan el uso de recursos como CPU, memoria, E/S de disco y red. Linux implementa dos versiones de cgroups: cgroups v1 (controladores individuales para cada recurso) y cgroups v2 (jerarquia unificada). Docker soporta ambas, con preferencia por v2 en sistemas modernos.

```bash
# Ver la version de cgroups en uso
docker info | grep -i cgroup
# Cgroup Driver: systemd
# Cgroup Version: 2

# Ver los cgroups de un contenedor
docker inspect --format '{{.HostConfig.CpuShares}}' mi-contenedor
docker inspect --format '{{.HostConfig.Memory}}' mi-contenedor
```

Los cgroups v2 organizan los limites en una jerarquia unica bajo `/sys/fs/cgroup/`. Docker crea un directorio para cada contenedor dentro de `/sys/fs/cgroup/system.slice/docker-<ID>.scope/`.

```bash
# Ver los limites de cgroups v2 para un contenedor
cat /sys/fs/cgroup/system.slice/docker-<ID>.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-<ID>.scope/cpu.max
```

Los limites se aplican en tiempo real: si un contenedor intenta superar su limite de memoria, el kernel puede terminarlo (OOM kill). Los limites de CPU se implementan como reparto proporcional: el kernel distribuye el tiempo de CPU entre los contenedores segun sus pesos relativos.

```bash
# Ejecutar un contenedor con limites y verificar su aplicacion
docker run -d --memory="512m" --cpus="2" --name test nginx

# Ver los limites en el cgroup
CONTAINER_ID=$(docker inspect --format '{{.Id}}' test)
cat /sys/fs/cgroup/system.slice/docker-$CONTAINER_ID.scope/memory.max
# 536870912
cat /sys/fs/cgroup/system.slice/docker-$CONTAINER_ID.scope/cpu.max
# 200000 100000
```

## Sistemas de archivos union

Los sistemas de archivos union (UnionFS) permiten superponer multiples directorios (capas) en un unico punto de montaje, mostrando una vista unificada del sistema de archivos. Docker utiliza estos sistemas para implementar las capas de las imagenes.

```bash
# Ver el driver de almacenamiento en uso
docker info | grep "Storage Driver"
# Storage Driver: overlay2
```

OverlayFS, el sistema de archivos union que Docker utiliza por defecto, trabaja con dos capas principales: una capa inferior (lowerdir, de solo lectura) y una capa superior (upperdir, de lectura y escritura). La vista unificada se presenta en un directorio llamado merged.

```
Proceso de montaje overlay2:
/var/lib/docker/overlay2/
├── <layer-hash-1>/  # Capa base (lower)
├── <layer-hash-2>/  # Capa intermedia (lower)
├── <layer-hash-3>/  # Capa superior de la imagen (lower)
└── <container-id>/  # Capa del contenedor (upper)
    ├── diff/        # Cambios realizados por el contenedor
    └── merged/      # Vista unificada de todas las capas
```

```bash
# Ver la estructura de capas de un contenedor
docker inspect mi-contenedor --format '{{.GraphDriver.Data}}'
# map[LowerDir:... MergedDir:... UpperDir:... WorkDir:...]
```

Cuando un contenedor escribe en un archivo, OverlayFS aplica la estrategia copy-on-write: el archivo se copia de la capa inferior a la superior y luego se modifica. Esto asegura que las capas inferiores (compartidas entre imagenes) permanezcan inmutables.

## Copys-on-write y el sistema de capas

La estrategia copy-on-write (CoW) es fundamental para la eficiencia de Docker. Cuando se crea un contenedor, no se copia toda la imagen; simplemente se anade una capa de escritura sobre las capas existentes. Las lecturas de archivos que no han sido modificados se sirven desde las capas inferiores; solo las escrituras generan nuevo contenido en la capa superior.

```bash
# Ver las capas de una imagen
docker history nginx:latest
# IMAGE          CREATED       CREATED BY                                      SIZE
# 605c77e624dd   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B
# <missing>      2 weeks ago   STOPSIGNAL SIGQUIT                              0B
# <missing>      2 weeks ago   EXPOSE 80                                       0B
# <missing>      2 weeks ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B
# <missing>      2 weeks ago   COPY 50x.html /usr/share/nginx/html ...         0B
# ... (cada capa listada con su tamano)
```

Cada linea en `docker history` representa una capa. Las capas etiquetadas como `<missing>` son capas de imagenes base o construidas en sistemas intermedios. Las capas de 0B representan cambios de metadatos (CMD, EXPOSE, ENTRYPOINT) que no ocupan espacio adicional.

## iptables y redireccion de puertos

Docker utiliza iptables para gestionar la redireccion de puertos y el aislamiento de red. Cuando se publica un puerto con `-p`, Docker anade reglas en las cadenas PREROUTING y FORWARD de iptables para redirigir el trafico desde el puerto del anfitrion al puerto del contenedor.

```bash
# Ver las reglas de iptables anadidas por Docker
sudo iptables -t nat -L -n
# Chain DOCKER (2 references)
# target     prot opt source         destination
# DNAT       tcp  --  0.0.0.0/0     0.0.0.0/0     tcp dpt:8080 to:172.17.0.2:80

# Ver las reglas de filtrado
sudo iptables -L DOCKER -n
```

Docker crea una cadena DOCKER en las tablas nat y filter, donde anade las reglas correspondientes a cada contenedor. Cuando un contenedor se elimina, Docker limpia las reglas asociadas.

## Seccomp y AppArmor

Seccomp (Secure Computing Mode) y AppArmor son mecanismos de seguridad del kernel de Linux que Docker utiliza para restringir las llamadas al sistema que un contenedor puede realizar.

```bash
# Ver el perfil seccomp por defecto de Docker
docker info | grep seccomp
# seccomp: enabled

# Ejecutar un contenedor con un perfil seccomp personalizado
docker run --security-opt seccomp=/path/to/profile.json nginx

# Ejecutar sin restricciones seccomp (no recomendado)
docker run --security-opt seccomp=unconfined nginx
```

Seccomp filtra las llamadas al sistema que un proceso puede hacer. Docker aplica un perfil por defecto que permite las llamadas esenciales y bloquea las peligrosas (como `mount`, `reboot`, `kexec_load`). AppArmor complementa a Seccomp aplicando politicas de control de acceso obligatorio (MAC) sobre los recursos del sistema.

> Estas tecnologias subyacentes son las que realmente hacen posible la contenerizacion. Docker no implementa namespaces ni cgroups; simplemente los configura y gestiona de forma coordinada. Entender como funcionan estos mecanismos a nivel de kernel permite comprender las limitaciones de los contenedores (por ejemplo, por que no se puede ejecutar un kernel diferente dentro de un contenedor) y tomar decisiones informadas sobre seguridad y rendimiento.

