# Tecnologias subyacentes en Linux

Los contenedores en Linux no son una tecnologia magica: se construyen sobre tres pilares fundamentales del kernel de Linux que existian mucho antes de que Docker los pusiera de moda. Namespaces, cgroups y union file systems son las piezas que hacen posible el aislamiento, el control de recursos y la eficiencia de almacenamiento de los contenedores.

## Namespaces: aislamiento de la vista del sistema

Los namespaces aíslan lo que un proceso puede ver del sistema. Cada namespace crea una "vista" independiente de un recurso del sistema.

- **PID namespace:** cada contenedor tiene su propio arbol de procesos. El PID 1 dentro del contenedor no es el PID 1 del host.
- **NET namespace:** cada contenedor tiene su propia pila de red, su propia IP, sus propias interfaces y tablas de enrutamiento.
- **MNT namespace:** cada contenedor tiene su propio sistema de archivos montado, aislado del host.
- **UTS namespace:** cada contenedor tiene su propio hostname y dominio.
- **USER namespace:** cada contenedor puede tener sus propios usuarios y grupos, mapeados a usuarios no privilegiados en el host.
- **IPC namespace:** aislamiento de comunicaciones interprocesos (semáforos, colas de mensajes, memoria compartida).

```bash
# Ver namespaces de un contenedor
docker run -d --name mi-web nginx
# Dentro del contenedor:
docker exec mi-web ps aux
# PID 1 es nginx, no el PID real del host

# En el host:
ps aux | grep nginx
# El proceso tiene un PID diferente (ej. 54321)
# El contenedor ve PID 1; el host ve PID 54321
```

## Cgroups: limites de recursos

Los cgroups (control groups) permiten limitar, priorizar y monitorean el uso de recursos de un proceso o grupo de procesos.

- **CPU:** limitar cuantos nucleos de CPU puede usar un contenedor.
- **Memoria:** limitar cuanta RAM puede consumir.
- **I/O de disco:** limitar la tasa de lectura/escritura.
- **Red:** limitar el ancho de banda de red.
- **PID:** limitar el numero de procesos que un contenedor puede crear.

```bash
# Docker: limitar recursos con cgroups
docker run -d \
  --name mi-app \
  --memory 512m \           # Maximo 512 MB de RAM
  --cpus 1.5 \              # Maximo 1.5 nucleos de CPU
  --pids-limit 100 \        # Maximo 100 procesos
  --cpu-shares 512 \        # Prioridad relativa de CPU
  mi-app

# Verificar limites dentro del contenedor
docker exec mi-app cat /sys/fs/cgroup/memory/memory.limit_in_bytes
# 536870912 (512 MB)
```

## Union File Systems: capas de almacenamiento

Los union file systems permiten crear un sistema de archivos virtual que apila multiples directorios (capas) como si fueran uno solo. Es lo que hace que las imagenes de contenedores sean eficientes en espacio.

**OverlayFS (el mas usado):**
```
Capa de escritura (contenedor)
    │
    ▼
Capas de solo lectura (imagen)
    │
    ├── Capa 4: configuracion nginx
    ├── Capa 3: nginx instalado
    ├── Capa 2: dependencias de sistema
    └── Capa 1: Ubuntu base (alpine)
```

Cuando un contenedor escribe un archivo, se crea una copia en la capa de escritura (copy-on-write). Las capas de solo lectura no se modifican.

```bash
# Ver capas de una imagen
docker history nginx:alpine
# IMAGE          CREATED      CREATED BY                      SIZE
# a1b2c3d4e5f6   2 weeks ago  /bin/sh -c #(nop) CMD ["nginx…  0B
# f6e5d4c3b2a1   2 weeks ago  /bin/sh -c #(nop) COPY file:…   2.58kB
# ...
# 567890abcdef   3 months ago /bin/sh -c #(nop) ADD file:…    7.74MB
```

**Eficiencia de espacio:**
Si 10 contenedores usan la misma imagen base Alpine (7.74 MB), solo se almacena una copia de la capa base. Las 10 capas de escritura son las unicas que ocupan espacio adicional.

> Namespaces, cgroups y union file systems son las tres patas sobre las que se sostiene la contenerizacion. Sin namespaces no hay aislamiento; sin cgroups no hay control de recursos; sin union file systems no hay eficiencia de almacenamiento. El kernel de Linux ya tenia estas capacidades; Docker las combino en un paquete facil de usar.

