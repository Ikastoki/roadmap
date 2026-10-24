# Buenas practicas y copias de seguridad

La gestion del almacenamiento persistente en Docker no se limita a elegir el mecanismo adecuado; implica adoptar una serie de practicas que garanticen la integridad, disponibilidad y portabilidad de los datos. El objetivo es tratar los contenedores como unidades de computo desechables mientras los datos se mantienen en capas de almacenamiento independientes, respaldadas y auditables.

## Separacion de responsabilidades

La regla fundamental del almacenamiento en contenedores es que el codigo y las dependencias pertenecen a la imagen, mientras que los datos pertenecen al volumen. Esta separacion permite actualizar la aplicacion simplemente sustituyendo la imagen, sin temor a perder informacion. Tambien facilita el escalado horizontal: multiples instancias del mismo contenedor pueden montar el mismo volumen y acceder a los datos compartidos.

```bash
# Correcto: datos fuera del contenedor
docker run -d --name postgres -v pgdata:/var/lib/postgresql/data postgres:16

# Incorrecto: datos dentro de la capa de escritura
docker run -d --name postgres postgres:16  # los datos desaparecen al hacer rm
```

En terminos practicos, esto significa que ninguna base de datos, sistema de colas o almacenamiento de archivos deberia ejecutarse sin un volumen montado. El directorio de datos de la aplicacion (definido en la documentacion de la imagen) debe montarse siempre como un volumen nombrado. Las excepciones son contenedores estrictamente temporales para pruebas o procesos batch que no requieren persistencia.

## Bases de datos en contenedores

Ejecutar bases de datos en Docker es una practica extendida en desarrollo y entornos de preproduccion, aunque en produccion critica suele preferirse una instancia gestionada fuera del ecosistema de contenedores. Cuando se opta por contenedores con base de datos, el volumen debe configurarse con el driver adecuado y, en entornos distribuidos, considerar drivers de volumen que replican los datos a traves de varios nodos.

```yaml
# docker-compose.yml para PostgreSQL en preproduccion
services:
  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: miapp
      POSTGRES_USER: app
      POSTGRES_PASSWORD: "${DB_PASSWORD}"
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

> No almacenar contraseñas ni datos sensibles en el archivo `docker-compose.yml`. Usar variables de entorno referenciadas desde un archivo `.env` o un gestor de secretos como HashiCorp Vault o Docker secrets.

## Logs y archivos generados

Las aplicaciones que generan logs, archivos subidos por usuarios o artefactos de compilacion deben escribirlos en un volumen, no en el sistema de archivos del contenedor. La rotacion de logs y la gestion del tamano del volumen deben planificarse desde el inicio, ya que un volumen lleno puede provocar la caida del contenedor y, en el caso de bases de datos, la corrupcion de los datos.

```bash
# Ver el espacio ocupado por un volumen
docker system df --volumes

# Limitar el tamano de un volumen con tmpfs (solo datos volatiles)
docker run --tmpfs /var/log:size=100m nginx
```

## Copias de seguridad con contenedor temporal

Docker no incluye una herramienta nativa de backup para volumenes, pero la arquitectura de montaje permite una estrategia elegante: ejecutar un contenedor temporal que monte el volumen a respaldar y genere un archivo comprimido en un directorio del host. Este contenedor efimero utiliza la imagen mas ligera disponible (`busybox`, `alpine`), ejecuta el comando de compresion y se elimina automaticamente al finalizar.

```bash
# Backup de un volumen a un archivo .tar en el directorio actual
docker run --rm \
  -v datos-postgres:/source:ro \
  -v "$(pwd)":/backup \
  alpine \
  tar czf /backup/postgres-backup-$(date +%Y%m%d).tar.gz -C /source .

# Restauracion de un backup a un volumen
docker run --rm \
  -v datos-postgres:/target \
  -v "$(pwd)":/backup \
  alpine \
  tar xzf /backup/postgres-backup-20260728.tar.gz -C /target
```

El parametro `--rm` garantiza que el contenedor se elimine al terminar, evitando acumular contenedores detenidos. La opcion `:ro` en el volumen de origen protege los datos contra modificaciones accidentales durante la copia. La restauracion sigue el mismo patron pero invirtiendo el sentido: el backup se lee del host y se extrae en el volumen montado como destino.

Para bases de datos, es preferible realizar el backup a traves de la herramienta nativa del motor en lugar de copiar el directorio de datos directamente, ya que una copia del sistema de archivos puede ser inconsistente si la base de datos esta en ejecucion y no se ha forzado un flush de las cache internas.

```bash
# Backup consistente de PostgreSQL usando pg_dump dentro del contenedor
docker exec -t postgres pg_dumpall -U app > backup.sql

# Restauracion
cat backup.sql | docker exec -i postgres psql -U app
```

> Programar los backups con un scheduler externo (cron, systemd timer, o un orquestador como Kubernetes con CronJobs). No ejecutar backups manualmente en produccion.

## Limpieza de volumenes huerfanos

Con el tiempo, los volumenes que ningun contenedor referencia se acumulan en el host ocupando espacio en disco. La limpieza periodica mediante `docker volume prune` debe formar parte del mantenimiento rutinario. En entornos de integracion continua, donde se crean y destruyen contenedores constantemente, es recomendable ejecutar este comando al final de cada pipeline.

```bash
# Programar limpieza semanal via cron (ejecutar como root)
0 3 * * 0 /usr/bin/docker volume prune -f >> /var/log/docker-prune.log 2>&1
```

La eliminacion de volumenes es irreversible. Antes de ejecutar `docker volume prune` en produccion, verificar que los volumenes marcados como dangling no contienen datos necesarios. Una practica defensiva consiste en etiquetar los volumenes importantes y filtrar los prunes para que solo actuen sobre los no etiquetados.

