# Gestion de volumenes

La gestion de volumenes incluye crear, inspeccionar, montar y eliminar volumenes. Los volumenes nombrados son la forma recomendada de persistir datos en Docker porque son portables, fiables y fiables de gestionar.

## Comandos basicos

```bash
# Crear volumen
docker volume create mi-datos

# Listar volumenes
docker volume ls
# DRIVER    VOLUME NAME
# local     mi-datos
# local     postgres-data
# local     redis-data

# Inspeccionar volumen
docker volume inspect mi-datos
# [
#   {
#     "CreatedAt": "2024-01-15T10:30:00Z",
#     "Driver": "local",
#     "Labels": {},
#     "Mountpoint": "/var/lib/docker/volumes/mi-datos/_data",
#     "Name": "mi-datos",
#     "Options": {},
#     "Scope": "local"
#   }
# ]

# Eliminar volumen
docker volume rm mi-datos

# Eliminar todos los volumenes sin usar
docker volume prune
# WARNING! This will delete all local volumes not used by at least one container.
# Delete all unused local volumes? [y/N]
```

## Montaje con -v vs. --mount

```bash
# Con -v (sintaxis abreviada)
docker run -d -v mi-datos:/var/lib/postgresql/data postgres:16

# Con --mount (sintaxis explicita, recomendada)
docker run -d --mount type=volume,source=mi-datos,target=/var/lib/postgresql/data postgres:16

# Bind mount con --mount
docker run -d --mount type=bind,source=$(pwd)/html,target=/usr/share/nginx/html,readonly nginx:alpine

# tmpfs con --mount
docker run -d --mount type=tmpfs,target=/tmp,tmpfs-size=100m nginx:alpine
```

## Volumenes en Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: miapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  postgres-data:    # Volumen nombrado declarado
  redis-data:       # Otro volumen nombrado
```

## Inspeccionar uso de volumenes

```bash
# Ver que contenedores usan un volumen
docker ps -q --filter volume=mi-datos
# a1b2c3d4e5f6

# Ver espacio ocupado por volumenes
docker system df -v
# VOLUME NAME     LINKS     SIZE
# mi-datos        1         150MB
# postgres-data   1         2.3GB
```

## Copias de seguridad de volumenes

```bash
# Backup de un volumen
docker run --rm \
  -v mi-datos:/datos:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/mi-datos-backup.tar.gz -C /datos .

# Restaurar volumen desde backup
docker volume create mi-datos-restaurado
docker run --rm \
  -v mi-datos-restaurado:/datos \
  -v $(pwd):/backup \
  alpine tar xzf /backup/mi-datos-backup.tar.gz -C /datos
```

## Eliminar volumenes huérfanos

```bash
# Verificar volumenes sin usar
docker volume ls -f dangling=true

# Eliminar volumenes huérfanos
docker volume prune

# Eliminar todo (contenedores, redes, volumenes, imagenes)
docker system prune -a --volumes
# CUIDADO: elimina TODOS los datos no referenciados
```

> Los volumenes gestionados son la columna vertebral del almacenamiento en contenedores. Mantener un inventario de volumenes, hacer backups periodicos y limpiar volumenes huérfanos son practicas esenciales para la operacion de contenedores en produccion.

