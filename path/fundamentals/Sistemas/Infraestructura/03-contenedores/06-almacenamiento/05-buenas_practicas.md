# Buenas practicas de almacenamiento

Las buenas practicas de almacenamiento garantizan que los datos persistentes esten seguros, respaldados y correctamente gestionados. La combinacion de volumenes nombrados, backups regulares y separacion de codigo y datos crea una base solida para la operacion de contenedores.

## Separar codigo de datos

```dockerfile
# MAL: los datos estan dentro del contenedor
# Si el contenedor se elimina, los datos se pierden
VOLUME /app/data
# Los datos se pierden porque el volumen es anonimo

# BIEN: usar volumen nombrado
docker run -d -v mis-datos:/app/data mi-app
# El volumen nombrado persiste independientemente del contenedor
```

**Regla:** el codigo va en la imagen; los datos van en volumenes.

## Nunca almacenar datos criticos en el filesystem efímero

```bash
# PELIGROSO: base de datos sin volumen nombrado
docker run -d --name db postgres:16-alpine
# Si eliminas el contenedor, pierdes TODOS los datos

# SEGURO: base de datos con volumen nombrado
docker run -d --name db \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:16-alpine
# Los datos sobreviven a la eliminacion del contenedor
```

## Backups periodicos

```bash
# Script de backup diario (cron)
#!/bin/bash
BACKUP_DIR="/backups/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup de PostgreSQL
docker exec mi-db pg_dump -U admin miapp | gzip > $BACKUP_DIR/db-$(date +%H%M).sql.gz

# Backup de volumenes
docker run --rm \
  -v postgres-data:/datos:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/volumen-$(date +%H%M).tar.gz -C /datos .
```

## Monitoreo de espacio

```bash
# Ver espacio usado por volumenes
docker system df -v

# Alerta si un volumen supera un limite
VOLUME_SIZE=$(docker exec mi-db du -sh /var/lib/postgresql/data | awk '{print $1}')
echo "Tamano del volumen: $VOLUME_SIZE"
```

## Limpieza de volumenes huérfanos

```bash
# Identificar volumenes sin usar
docker volume ls -f dangling=true

# Eliminar volumenes huérfanos
docker volume prune

# En Docker Compose:
docker volume prune -f
```

## Permisos y propiedad

```dockerfile
# Asegurar permisos correctos en el contenedor
RUN mkdir -p /app/data && \
    chown -R appuser:appgroup /app/data
USER appuser
# El usuario del contenedor debe poder leer/escribir en el volumen
```

```bash
# Si hay problemas de permisos
docker run -v mi-datos:/app/data -u root mi-app chown -R appuser:appgroup /app/data
```

> Las buenas practicas de almacenamiento son inversiones en la integridad de los datos. Un volumen nombrado con backups periodicos y monitoreo de espacio es la base de la confiabilidad de cualquier aplicacion que depende de datos persistentes en contenedores.

