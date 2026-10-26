# Tipos de almacenamiento

Docker y Podman ofrecen tres tipos de almacenamiento para resolver el problema de la efimeridad: volumenes gestionados, bind mounts y tmpfs. Cada tipo tiene un caso de uso especifico y diferentes caracteristicas de persistencia, rendimiento y portabilidad.

## Volumes (volúmenes gestionados)

Los volumenes son directorios gestionados por el motor de contenedores. Se almacenan en `/var/lib/docker/volumes/` (Docker) y persisten independientemente del ciclo de vida del contenedor.

```bash
# Crear volumen nombrado
docker volume create mi-datos

# Ejecutar contenedor con volumen
docker run -d \
  --name mi-db \
  -v mi-datos:/var/lib/postgresql/data \
  postgres:16-alpine

# Los datos sobreviven a:
docker stop mi-db
docker rm mi-db
# El volumen mi-datos sigue existiendo

# Ver volumenes
docker volume ls
# DRIVER    VOLUME NAME
# local     mi-datos
# local     postgres-data
```

**Caracteristicas:**
- Gestionados por Docker/Podman.
- Persisten al eliminar el contenedor.
- Portables entre hosts (si se copian).
- Mejor rendimiento que bind mounts en macOS/Windows.
- Ideal para produccion.

## Bind Mounts

Vinculan un directorio especifico del sistema anfitrion al contenedor. El contenedor lee y escribe directamente en el filesystem del host.

```bash
# Montar directorio del host
docker run -d \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx:alpine
# Los archivos de $(pwd)/html se reflejan en el contenedor

# Con opciones
docker run -d \
  -v $(pwd)/src:/app/src:ro \     # Solo lectura
  -v $(pwd)/data:/app/data \      # Lectura/escritura
  nginx:alpine
```

**Caracteristicas:**
- Acceso directo al filesystem del host.
- Cambios en el host son inmediatos en el contenedor (y viceversa).
- Ideal para desarrollo (hot reload).
- Dependen del path del host (no portables).
- Rendimiento variable en macOS/Windows (la VM intermedia).

**Usos tipicos:**
- Desarrollo: montar el codigo fuente para hot reload.
- Configuracion: montar archivos de configuracion.
- Herramientas de diagnostico.

## tmpfs

Almacenamiento temporal en memoria RAM. Los datos no se escriben en disco y se pierden al detener el contenedor.

```bash
# Montar tmpfs
docker run -d \
  --tmpfs /tmp:rw,size=100m,noexec,nosuid \
  nginx:alpine
# /tmp es writable pero en memoria RAM
# Los datos se pierden al detener el contenedor
```

**Caracteristicas:**
- Solo en Linux (no funciona en macOS/Windows via Docker Desktop).
- Los datos estan en memoria RAM (maximo rapido).
- No persisten al detener el contenedor.
- Ideal para datos sensibles (no tocan disco) y temporales de alto rendimiento.

## Comparacion

| Aspecto | Volumen | Bind Mount | tmpfs |
|---|---|---|---|
| Persistencia | Si | Si | No |
| Gestionado por | Docker | Usuario | SO |
| Portabilidad | Alta | Baja | Ninguna |
| Rendimiento en Linux | Alto | Alto | Muy alto |
| Rendimiento en macOS | Alto | Bajo (VM) | N/A |
| Ideal para | Produccion | Desarrollo | Temporal/sensible |

```bash
# Volumen: datos persistentes
docker run -d -v postgres-data:/var/lib/postgresql/data postgres:16

# Bind mount: codigo en desarrollo
docker run -d -v $(pwd)/src:/app/src node:20 npm run dev

# tmpfs: datos temporales sensibles
docker run -d --tmpfs /tmp:rw,size=50m my-app
```

> La eleccion del tipo de almacenamiento determina la durabilidad, el rendimiento y la portabilidad de los datos. En produccion, usa volumenes. En desarrollo, usa bind mounts. Para datos temporales y sensibles, usa tmpfs.

