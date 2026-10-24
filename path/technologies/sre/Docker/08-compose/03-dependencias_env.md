# Dependencias, healthchecks y variables de entorno

Cuando una aplicacion se compone de varios servicios, el orden de inicio y la configuracion sensible se convierten en problemas practicos. La base de datos debe estar disponible antes de que la aplicacion intente conectarse, y las credenciales no deben aparecer en el codigo fuente ni en el repositorio. Docker Compose proporciona mecanismos para abordar ambos aspectos: `depends_on` para expresar dependencias entre servicios, `healthcheck` para determinar si un servicio esta realmente operativo, y sistema de variables de entorno con archivos `.env` para separar la configuracion del codigo.

## depends_on: dependencias entre servicios

La directiva `depends_on` establece el orden de creacion e inicio de los servicios. Cuando un servicio A declara `depends_on: B`, Compose asegura que B se inicie antes que A y que A se detenga antes que B.

```yaml
services:
  app:
    image: mi-app:latest
    depends_on:
      - db
      - redis

  db:
    image: postgres:15

  redis:
    image: redis:7-alpine
```

Sin embargo, `depends_on` solo garantiza que el contenedor dependiente haya sido creado e iniciado, no que el proceso dentro de el este listo para aceptar conexiones. Una base de datos PostgreSQL puede tardar varios segundos en inicializarse incluso despues de que el contenedor este en estado "running". Esto significa que, con `depends_on` solo, la aplicacion podria intentar conectarse a la base de datos antes de que esta haya completado su arranque.

```yaml
services:
  app:
    image: mi-app:latest
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

La forma extendida de `depends_on` permite especificar `condition: service_healthy`, que hace que Compose espere hasta que el servicio dependiente pase su healthcheck antes de iniciar el servicio que depende de el.

> `depends_on` en su forma simple no espera a que el servicio este listo. Es un error comun asumir que si, y la consecuencia son errores de conexion intermitentes al arrancar el stack. Siempre que sea posible, usar la forma extendida con `condition: service_healthy` combinada con un healthcheck adecuado.

## Healthchecks

Un healthcheck es un comando que Docker ejecuta periodicamente dentro del contenedor para determinar si el proceso esta funcionando correctamente. Si el comando devuelve un codigo de salida 0, el contenedor se considera saludable; cualquier otro codigo indica fallo.

```yaml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
```

Los parametros del healthcheck son:

- **`test`**: comando a ejecutar. Puede ser una cadena, una lista estilo `CMD` o `CMD-SHELL`.
- **`interval`**: tiempo entre ejecuciones del healthcheck.
- **`timeout`**: tiempo maximo para que el comando responda.
- **`retries`**: numero de fallos consecutivos para considerar el contenedor como no saludable.
- **`start_period`**: tiempo de gracia inicial durante el cual los fallos no cuentan.

Para inspeccionar el estado de salud de un contenedor:

```bash
docker compose ps
docker inspect --format='{{.State.Health.Status}}' <container>
```

Un contenedor puede estar en los estados `starting`, `healthy` o `unhealthy`. Compose puede usar estos estados con `condition: service_healthy` en `depends_on` para sincronizar el arranque.

## Variables de entorno en Compose

Las variables de entorno son el mecanismo estandar para parametrizar la configuracion de los contenedores: credenciales de bases de datos, URLs de servicios externos, niveles de log, etc. Compose ofrece dos formas de definirlas.

### environment

Define variables directamente en el archivo Compose:

```yaml
services:
  app:
    image: mi-app:latest
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - DB_NAME=mi_app
      - DB_USER=admin
      - DB_PASSWORD=secret
```

Tambien en formato de mapa:

```yaml
environment:
  DB_HOST: db
  DB_PORT: 5432
```

El problema de esta aproximacion es que las credenciales quedan escritas en el archivo Compose, que tipicamente se versiona en Git. Para separar la configuracion sensible del codigo se utiliza `env_file`.

### env_file

Carga variables de entorno desde un archivo externo:

```yaml
services:
  app:
    image: mi-app:latest
    env_file:
      - ./app.env
```

El archivo `.env` (o `app.env`) tiene un formato simple de pares `CLAVE=valor`:

```
DB_HOST=db
DB_PORT=5432
DB_NAME=mi_app
DB_USER=admin
DB_PASSWORD=secret
LOG_LEVEL=debug
```

> El archivo `.env` no debe incluirse en el repositorio cuando contiene credenciales reales. Se recomienda crear un `.env.example` con valores ficticios como plantilla y anadir `.env` al `.gitignore`.

### Interpolacion de variables

Compose permite usar variables de entorno del shell en el propio archivo YAML mediante la sintaxis `${VARIABLE}`:

```yaml
services:
  app:
    image: mi-app:${VERSION:-latest}
    ports:
      - "${PORT:-5000}:5000"
```

El valor tras el guion (`:-latest`) es el valor por defecto si la variable no esta definida. Esta tecnica es util para parametrizar entornos (desarrollo, staging, produccion) sin duplicar archivos Compose.

```bash
# Ejecutar con variable del shell
VERSION=1.2.3 PORT=8080 docker compose up -d
```

### Archivo .env del proyecto

Ademas de `env_file` por servicio, Compose lee automaticamente un archivo `.env` en el directorio del proyecto y lo usa para interpolar variables en el propio `docker-compose.yml`. Este comportamiento es automatico y no requiere declaracion explicita.

```
# .env (en el directorio del proyecto)
POSTGRES_VERSION=15
APP_PORT=5000
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:${POSTGRES_VERSION}
  app:
    ports:
      - "${APP_PORT}:5000"
```

> El orden de precedencia para la resolucion de variables es: 1) entorno del shell, 2) archivo `.env` del proyecto, 3) valor por defecto en la interpolacion.

