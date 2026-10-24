# Que es un contenedor

Un contenedor no es una maquina virtual ni un entorno de emulacion; es un proceso aislado que se ejecuta directamente sobre el kernel del sistema operativo anfitrion, compartiendo su nucleo pero operando en un espacio de usuario independiente. Esta distincion es fundamental para entender por que los contenedores han transformado la manera en que desarrollamos, distribuimos y ejecutamos software. Mientras que las maquinas virtuales virtualizan el hardware, los contenedores virtualizan el sistema operativo, lo que los hace drasticamente mas ligeros, rapidos y eficientes en terminos de recursos. La tecnologia de contenedores se apoya en dos mecanismos clave del kernel de Linux: los namespaces, que crean entornos aislados donde un proceso percibe su propio sistema de archivos, red, arbol de procesos y usuarios; y los cgroups, que limitan y monitorizan el uso de recursos como CPU, memoria y disco. Un contenedor no sabe que comparte el kernel con otros procesos y, desde su perspectiva, se comporta como un sistema completo con su propio conjunto de herramientas, bibliotecas y configuraciones.

## Namespaces: el pilar del aislamiento

Los namespaces son la caracteristica del kernel de Linux que permite que cada contenedor tenga una vision privada del sistema. Cuando un contenedor se inicia, Docker crea un conjunto de namespaces para ese contenedor: el namespace de PID aísla los identificadores de proceso, de forma que el proceso init del contenedor ve el PID 1 y no conoce los procesos del anfitrion; el namespace de montaje proporciona un arbol de sistema de archivos independiente; el namespace de red ofrece su propia pila de red con interfaces, rutas y reglas de firewall; el namespace de UTS aisla el hostname y el dominio; el namespace de IPC restringe la comunicacion entre procesos al interior del contenedor; y el namespace de usuario permite mapear usuarios privilegiados dentro del contenedor a usuarios no privilegiados en el anfitrion.

```bash
# Ver los namespaces de un contenedor en ejecucion
docker inspect --format '{{.State.Pid}}' <contenedor>
# Con el PID, se pueden ver los namespaces en /proc/<pid>/ns
ls -la /proc/<pid>/ns/
```

Cada namespace aparece como un enlace simbolico que apunta a un identificador unico. Si dos procesos comparten el mismo identificador para un namespace determinado, significa que comparten ese recurso. Docker utiliza esta mecanica para aislar contenedores entre si y respecto al anfitrion.

## Cgroups: control de recursos

Los cgroups (control groups) complementan a los namespaces proporcionando la capacidad de limitar, contabilizar y priorizar el uso de recursos. Sin cgroups, un contenedor podria consumir toda la memoria del sistema o saturar la CPU, afectando a otros contenedores y al propio anfitrion. Docker asigna por defecto un cgroup a cada contenedor, pero el usuario puede personalizar los limites en el momento de la ejecucion.

```bash
# Ejecutar un contenedor con limites de memoria y CPU
docker run --memory="256m" --cpus="1.5" nginx

# Ver el estado de los cgroups para un contenedor
docker stats <contenedor>
```

Los cgroups se organizan jerarquicamente en el sistema de archivos virtual `/sys/fs/cgroup/`, donde cada subsistema (memoria, cpu, blkio, etc.) tiene su propia jerarquia. Docker escribe en estos archivos virtuales para establecer los limites, y el kernel se encarga de hacerlos cumplir.

## Sistemas de archivos en capas

Una de las innovaciones mas importantes de Docker es el uso de sistemas de archivos en capas (layered filesystems) para construir imagenes. Cada instruccion en un Dockerfile crea una nueva capa que contiene los cambios respecto a la capa anterior. Estas capas son de solo lectura y se comparten entre imagenes y contenedores, lo que ahorra espacio en disco y acelera las descargas.

```bash
# Ver las capas de una imagen
docker history nginx:latest
```

Cuando un contenedor se inicia, Docker anade una capa de escritura (container layer) sobre las capas de solo lectura de la imagen. Todos los cambios que el contenedor realiza durante su ejecucion se almacenan en esta capa, que se descarta cuando el contenedor se elimina a menos que se use un volumen.

> El sistema de capas es la razon por la que construir una imagen eficiente requiere ordenar las instrucciones de menor a mayor frecuencia de cambio: las capas que cambian con menos frecuencia deben ir al principio del Dockerfile para maximizar el uso de la cache.

## El contenedor como unidad de distribucion

Un contenedor no es solo una forma de ejecutar software; es una unidad de distribucion estandarizada. Al empaquetar la aplicacion con todas sus dependencias, bibliotecas, configuraciones y variables de entorno en una imagen, se elimina el problema de "en mi maquina funciona". La misma imagen que se prueba en desarrollo es la que se despliega en produccion, sin necesidad de instalar dependencias adicionales ni configurar el entorno manualmente.

```bash
# Extraer una imagen y ejecutarla en cualquier sistema con Docker
docker pull nginx:1.25
docker run -d -p 8080:80 nginx:1.25
```

Esta portabilidad se extiende a cualquier sistema operativo que pueda ejecutar Docker, ya sea Linux, Windows o macOS, y a cualquier proveedor de nube, desde AWS hasta un servidor on-premise.

## Contenedores y microservicios

La adopcion de contenedores esta intimamente ligada a la arquitectura de microservicios. Un contenedor encapsula idealmente un solo proceso o servicio, lo que permite descomponer aplicaciones monoliticas en componentes independientes que pueden desarrollarse, desplegarse y escalarse por separado. Cada microservicio en su contenedor tiene su propio ciclo de vida, sus propias dependencias y su propia configuracion, lo que facilita la mantenibilidad y la evolucion del sistema.

> Un contenedor debe ejecutar un solo proceso. Ejecutar multiples procesos dentro del mismo contenedor complica la gestion de senales, la recoleccion de procesos zombies y el escalado individual de cada servicio.

