# Eliminacion automatica

La eliminacion automatica de contenedores con `--rm` es una practica esencial para mantener el sistema limpio. Los contenedores efimeros (que ejecutan una tarea y terminan) no deben quedarse acumulados en el sistema. `--rm` elimina el contenedor automaticamente al detenerse, evitando la acumulacion de contenedores obsoletos.

## Cuando usar --rm

- **Tareas de un solo uso:** scripts, migraciones de base de datos, procesamiento de archivos.
- **Pruebas:** ejecutar una imagen para verificar algo y que se elimine despues.
- **CI/CD:** pipelines que necesitan contenedores limpios para cada ejecucion.
- **Herramientas de diagnostico:** contenedores temporales para inspeccionar algo.

```bash
# Ejecutar script Python y eliminar
docker run --rm -v $(pwd):/app -w /app python:3.12 python script.py
# El contenedor se ejecuta, procesa el script y se elimina

# Ejecutar migracion de base de datos y eliminar
docker run --rm mi-app:1.0 node migrate.js
# El contenedor ejecuta la migracion y desaparece

# Ejecutar tests y eliminar
docker run --rm -v $(pwd):/app -w /app node:20 npm test
# Los tests corren en un contenedor limpio que se elimina al terminar
```

## Ejemplo: tarea programada

```bash
# Backup diario de base de datos
docker run --rm \
  -e PGPASSWORD=secret \
  -v $(pwd)/backups:/backups \
  postgres:16-alpine \
  pg_dump -h mi-db -U admin mibasededatos | gzip > /backups/backup-$(date +%Y%m%d).sql.gz
# El contenedor se crea, ejecuta el backup y se elimina
```

## Docker system prune: limpieza general

```bash
# Limpiar contenedores detenidos, imagenes colgadas, redes sin usar
docker system prune
# WARNING! This will remove:
#   - all stopped containers
#   - all networks not used by at least one container
#   - all dangling images
#   - all dangling build cache
# Total reclaimed space: 2.1GB

# Limpiar todo (incluyendo imagenes no usadas)
docker system prune -a
# WARNING! This will remove all images without at least one container
# Total reclaimed space: 5.8GB

# Incluir volumenes en la limpieza
docker system prune -a --volumes
# CUIDADO: elimina volumenes con datos
```

```bash
# Limpiar solo contenedores detenidos
docker container prune

# Limpiar solo imagenes colgadas
docker image prune

# Limpiar solo redes sin usar
docker network prune

# Limpiar solo volumenes sin usar
docker volume prune
```

## Eliminacion programada

```bash
# Script para eliminar contenedores detenidos mayores a 7 dias
docker ps -a --filter "status=exited" --format "{{.ID}} {{.CreatedAt}}" | \
while read id created; do
  if [ $(date -d "$created" +%s) -lt $(date -d "7 days ago" +%s) ]; then
    docker rm $id
    echo "Eliminado contenedor $id (creado: $created)"
  fi
done
```

> La eliminacion automatica y la limpieza periodica son esenciales para mantener un sistema de contenedores saludable. Un sistema sin limpieza acumula contenedores, imagenes y volumenes que consumen espacio y dificultan el diagnostico. `docker system prune` es el equivalent a `apt autoremove` pero para contenedores.

