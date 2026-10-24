# Comandos esenciales de Docker Compose

Docker Compose expone un conjunto de comandos que permiten controlar el ciclo de vida completo de una aplicacion multi-servicio. A diferencia de los comandos de Docker que operan sobre contenedores individuales, los comandos de Compose actuan sobre el proyecto en su conjunto, interpretando el archivo `docker-compose.yml` para determinar que contenedores, redes y volumenes deben crearse, modificarse o eliminarse.

La sintaxis general desde Docker Compose v2 es:

```bash
docker compose <comando> [opciones]
```

> En versiones anteriores era necesario usar `docker-compose` (con guion) como binario independiente. Docker Compose v2 esta integrado en el CLI de Docker y se invoca como subcomando: `docker compose`. Si el sistema aun usa la version legacy, el comportamiento es identico pero el binario cambia.

## docker compose up

Es el comando principal. Crea e inicia todos los servicios definidos en el archivo Compose. Si las imagenes no existen localmente, las descarga automaticamente. Si un servicio tiene una seccion `build`, Compose construye la imagen antes de iniciar el contenedor.

```bash
# Crear e iniciar todos los servicios en primer plano
docker compose up

# Ejecutar en segundo plano (detached)
docker compose up -d

# Construir las imagenes antes de iniciar (ignora cache si existe)
docker compose up -d --build

# Iniciar solo servicios especificos
docker compose up -d web db
```

Cuando se ejecuta `docker compose up`, Compose crea una red por defecto para el proyecto, monta los volumenes declarados y conecta cada servicio a las redes correspondientes. Los logs de todos los servicios se multiplexan en la salida estandar.

> La opcion `-d` (detached) libera la terminal y deja los contenedores ejecutandose en segundo plano. Es la forma habitual en entornos de produccion o desarrollo prolongado.

## docker compose down

Detiene y elimina los recursos creados por `up`. Por defecto elimina los contenedores y la red por defecto, pero preserva los volumenes nombrados para no perder datos persistentes.

```bash
# Detener y eliminar contenedores y redes
docker compose down

# Eliminar tambien los volumenes nombrados
docker compose down -v

# Eliminar tambien las imagenes construidas
docker compose down --rmi all
```

Es importante entender que `down` no es lo mismo que `stop`. Mientras que `stop` solo detiene los contenedores (pueden reanudarse con `start`), `down` los elimina junto con las redes. Los volumenes se conservan a menos que se use `-v`.

```bash
# Detener servicios sin eliminar recursos
docker compose stop

# Reanudar servicios detenidos
docker compose start

# Reiniciar servicios
docker compose restart
```

## docker compose logs

Accede a la salida estandar y de error de los servicios. Es la herramienta principal para depurar problemas en aplicaciones multi-servicio.

```bash
# Ver logs de todos los servicios
docker compose logs

# Seguir los logs en tiempo real (tail -f)
docker compose logs -f

# Ver logs de un servicio concreto
docker compose logs web

# Mostrar solo las ultimas N lineas
docker compose logs --tail=50

# Incluir timestamps
docker compose logs -t
```

> Por defecto `logs` muestra las lineas de todos los servicios mezcladas. Para seguir unicamente un servicio en tiempo real sin la mezcla del resto, se combina `-f` con el nombre del servicio: `docker compose logs -f api`.

## docker compose exec

Ejecuta un comando dentro de un contenedor que ya esta en ejecucion. Es equivalente a `docker exec` pero referenciando al servicio por su nombre en lugar del ID del contenedor.

```bash
# Abrir una shell interactiva en el contenedor del servicio web
docker compose exec web /bin/bash

# Ejecutar un comando puntual
docker compose exec db pg_dump -U postgres mi_db > backup.sql

# Ejecutar como un usuario especifico
docker compose exec --user=1000 web npm test
```

A diferencia de `docker exec`, Compose resuelve automaticamente el nombre del servicio al contenedor correspondiente, incluso si hay multiples replicas escaladas.

## Otros comandos utiles

```bash
# Listar servicios y su estado
docker compose ps

# Ver la configuracion efectiva (tras resolver variables)
docker compose config

# Construir imagenes sin iniciar servicios
docker compose build

# Descargar imagenes sin iniciar servicios
docker compose pull

# Eliminar contenedores, redes y volumenes no utilizados
docker compose down -v
```

El comando `config` es especialmente util para verificar que el archivo YAML es correcto y visualizar como quedan las variables de entorno y los valores por defecto tras la resolucion:

```bash
docker compose config
```

Si el archivo tiene errores de sintaxis, `config` los reporta sin necesidad de intentar levantar los servicios.

