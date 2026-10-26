# Recursos y limites

Sin limites de recursos, un contenedor puede consumir toda la CPU, memoria y disco del host, afectando a los demas contenedores y al propio sistema operativo. Establecer limites claros es fundamental para mantener la estabilidad del host y garantizar un uso justo de los recursos entre multiples contenedores.

## Limitar memoria

```bash
# Limitar memoria maxima
docker run -d --memory 512m --name mi-app mi-app:1.0
# El contenedor no puede usar mas de 512 MB de RAM
# Si supera el limite, el kernel lo termina (OOM killed)

# Limitar memoria y swap
docker run -d --memory 512m --memory-swap 1g mi-app:1.0
# RAM maxima: 512 MB
# RAM + Swap maximo: 1 GB

# Sin swap
docker run -d --memory 512m --memory-swap 512m mi-app:1.0
# Swap deshabilitado (memory-swap = memory)

# Memoria reservada (garantizada)
docker run -d --memory 1g --memory-reservation 512m mi-app:1.0
# Garantiza al menos 512 MB; puede usar hasta 1 GB si hay disponibilidad
```

## Limitar CPU

```bash
# Limitar a 1.5 nucleos de CPU
docker run -d --cpus 1.5 mi-app:1.0

# Limitar a 50% de un nucleo
docker run -d --cpus 0.5 mi-app:1.0

# Asignar nucleos especificos (CPU affinity)
docker run -d --cpuset-cpus="0,1" mi-app:1.0
# Solo puede usar los nucleos 0 y 1

# Peso relativo de CPU (default: 1024)
docker run -d --cpu-shares 512 mi-app:1.0   # Menos prioridad
docker run -d --cpu-shares 2048 mi-app:1.0  # Mas prioridad
```

## Limitar I/O de disco

```bash
# Limitar velocidad de lectura/escritura
docker run -d --device-read-bps /dev/sda:10mb mi-app:1.0
docker run -d --device-write-bps /dev/sda:5mb mi-app:1.0

# Limitar IOPS
docker run -d --device-read-iops /dev/sda:1000 mi-app:1.0
docker run -d --device-write-iops /dev/sda:500 mi-app:1.0
```

## Limitar procesos (PIDs)

```bash
# Limitar numero maximo de procesos
docker run -d --pids-limit 100 mi-app:1.0
# Evita forks de bombas de procesos (fork bombs)
```

## Ejemplo completo

```bash
# Contenedor con todos los limites definidos
docker run -d \
  --name mi-app-produccion \
  --memory 1g \
  --memory-swap 1g \
  --cpus 2 \
  --pids-limit 200 \
  --device-read-bps /dev/sda:50mb \
  --device-write-bps /dev/sda:20mb \
  --restart unless-stopped \
  mi-app:1.0
```

## Verificar limites

```bash
# Ver limites de memoria
docker exec mi-app cat /sys/fs/cgroup/memory/memory.limit_in_bytes
# 1073741824 (1 GB)

# Ver uso de memoria
docker exec mi-app cat /sys/fs/cgroup/memory/memory.usage_in_bytes

# Ver limites de CPU
docker exec mi-app cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
docker exec mi-app cat /sys/fs/cgroup/cpu/cpu.cfs_period_us

# Ver stats en tiempo real
docker stats mi-app
```

## El problema del noisy neighbor

Sin limites, un contenedor que consume todos los recursos afecta a los demas. Los limites previenen este escenario:

```bash
# Contenedor sin limites (RIESGO)
docker run -d --name greedy-app mi-app:1.0
# Si este contenedor consume toda la RAM, otros contenedores seran OOM killed

# Contenedor con limites (SEGURO)
docker run -d --name safe-app --memory 512m --cpus 1 mi-app:1.0
# Este contenedor no puede afectar a los demas
```

> Establecer limites de recursos no es opcional en produccion: es obligatorio. Un contenedor sin limites es como un inquilino sin contrato que puede ocupar todo el edificio. Los limites garantizan que cada contenedor recibe los recursos que necesita sin afectar a los demas.

