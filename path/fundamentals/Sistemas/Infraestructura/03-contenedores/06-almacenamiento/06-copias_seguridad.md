# Copias de seguridad de volumenes

Los volumenes de Docker contienen datos criticos que deben ser respaldados periodicamente. Dado que los volumenes son directorios en el host, las copias de seguridad se realizan ejecutando un contenedor temporal que tiene acceso al volumen y empaqueta los datos en un archivo comprimido.

## Backup basico

```bash
# Crear backup de un volumen
docker run --rm \
  -v postgres-data:/datos:ro \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/postgres-backup-$(date +%Y%m%d).tar.gz -C /datos .

# Resultado: backups/postgres-backup-20240115.tar.gz
```

## Backup de multiples volumenes

```bash
# Script de backup completo
#!/bin/bash
BACKUP_DIR="/backups/$(date +%Y%m%d-%H%M)"
mkdir -p $BACKUP_DIR

# Backup de PostgreSQL
docker run --rm \
  -v postgres-data:/datos:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/postgres.tar.gz -C /datos .

# Backup de Redis
docker run --rm \
  -v redis-data:/datos:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/redis.tar.gz -C /datos .

# Backup de archivos de la app
docker run --rm \
  -v app-uploads:/datos:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/uploads.tar.gz -C /datos .

echo "Backups completos en: $BACKUP_DIR"
ls -lh $BACKUP_DIR
```

## Restaurar volumen

```bash
# Crear volumen vacio
docker volume create postgres-data-restaurado

# Restaurar desde backup
docker run --rm \
  -v postgres-data-restaurado:/datos \
  -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/postgres-backup-20240115.tar.gz -C /datos

# Usar el volumen restaurado
docker run -d \
  --name db-restaurada \
  -v postgres-data-restaurado:/var/lib/postgresql/data \
  postgres:16-alpine
```

## Backup con pg_dump ( PostgreSQL)

```bash
# Backup logico de PostgreSQL (mas flexible que backup de volumen)
docker exec mi-db pg_dump -U admin -Fc miapp > backup.dump

# Backup con pg_dumpall (incluye usuarios y roles)
docker exec mi-db pg_dumpall -U admin > backup-all.sql

# Restaurar backup logico
docker exec -i mi-db pg_restore -U admin -d miapp < backup.dump
```

## Backup automatizado con cron

```bash
# /etc/cron.d/docker-backup
# Backup diario a las 2:00 AM
0 2 * * * root /usr/local/bin/docker-backup.sh

# Script: /usr/local/bin/docker-backup.sh
#!/bin/bash
BACKUP_DIR="/backups/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

docker run --rm \
  -v postgres-data:/datos:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/postgres.tar.gz -C /datos .

# Mantener solo los ultimos 7 dias de backups
find /backups -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \;
```

> Las copias de seguridad de volumenes son la ultima linea de defensa contra la perdida de datos. Un volumen sin backup es un volador de datos esperando a que algo salga mal. La automatizacion de backups con scripts y cron garantiza que los datos esten protegidos sin intervencion manual.

