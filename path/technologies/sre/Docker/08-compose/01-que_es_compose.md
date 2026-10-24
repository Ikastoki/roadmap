# Docker Compose: definicion y estructura

Ejecutar aplicaciones con Docker mediante `docker run` funciona para contenedores aislados, pero tan pronto como una aplicacion requiere multiples servicios (una base de datos, un servidor web, una cola de mensajes, un cache), la gestion manual se vuelve insostenible. Es necesario orquestar la creacion de redes, la asignacion de volumenes, el orden de inicio y la configuracion de cada contenedor, todo ello de forma coordinada y repetible. Docker Compose resuelve exactamente ese problema: es una herramienta que permite definir y ejecutar aplicaciones multi-servicio a partir de un archivo declarativo YAML, eliminando la necesidad de escribir largos scripts o recordar decenas de parametros de linea de comandos.

## Que es Docker Compose

Docker Compose es una herramienta de orquestacion ligera que forma parte del ecosistema Docker. Su proposito es tomar un archivo de configuracion --por defecto `docker-compose.yml`-- y traducirlo en uno o varios contenedores, redes y volumenes, gestionando su ciclo de vida como una sola unidad logica llamada "proyecto" o "stack". Cada servicio definido en el archivo se corresponde con un contenedor, pero Compose abstrae los detalles de conexion: los servicios se descubren entre si por su nombre, se conectan a las redes declaradas y montan los volumenes especificados sin intervencion manual.

Antes de Compose, el flujo tipico para levantar una aplicacion con varios servicios era algo como:

```bash
docker network create mi-red
docker volume create pgdata
docker run -d --name db --network mi-red -v pgdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:15
docker run -d --name app --network mi-red -p 5000:5000 mi-app:latest
```

Con Compose, todo eso se reduce a un archivo y un solo comando.

## Estructura del archivo docker-compose.yml

El archivo `docker-compose.yml` sigue el formato YAML y se organiza en secciones bien definidas. La estructura basica consta de cuatro bloques principales: `version`, `services`, `volumes` y `networks`.

```yaml
version: "3.9"

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - api

  api:
    image: mi-api:latest
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db
    depends_on:
      - db

  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  pgdata:

networks:
  default:
    driver: bridge
```

### La clave `version`

Indica la version del formato del archivo Compose que se esta utilizando. Aunque las versiones recientes de Docker Compose (v2) ya no exigen esta clave y tratan el archivo como si fuera de la version mas reciente, incluirla de forma explicita garantiza compatibilidad con versiones anteriores. Los valores mas comunes son `"3.8"` o `"3.9"`.

> A partir de Docker Compose v2 integrado en el CLI de Docker (`docker compose` en lugar de `docker-compose`), el campo `version` es opcional. Se recomienda omitirlo en proyectos nuevos.

### La clave `services`

Es el corazon del archivo. Cada servicio define un contenedor y su configuracion. Las opciones mas importantes dentro de cada servicio son:

- **`image`**: imagen base desde la que se crea el contenedor. Puede ser una imagen publica (`postgres:15`), privada (`registry.example.com/mi-app:1.0`) o construida localmente mediante `build`.
- **`ports`**: mapeo de puertos entre el host y el contenedor en formato `"HOST:CONTAINER"`.
- **`volumes`**: montaje de volumenes o bind mounts dentro del contenedor.
- **`environment`**: variables de entorno definidas directamente en el archivo.
- **`env_file`**: ruta a un archivo que contiene variables de entorno.
- **`depends_on`**: declaracion de dependencias entre servicios.
- **`networks`**: redes a las que se conecta el servicio.

### La clave `volumes`

Declara los volumenes nombrados que usaran los servicios. Un volumen nombrado es gestionado por Docker y persiste independientemente del ciclo de vida del contenedor. Simplemente listar el nombre del volumen aqui asegura que Docker lo cree si no existe.

```yaml
volumes:
  pgdata:
  redis_data:
```

### La clave `networks`

Define redes personalizadas. Por defecto Compose crea una red por proyecto, pero cuando se necesita control sobre el driver, el direccionamiento IP o el aislamiento entre servicios, se declaran explicitamente:

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

> Una red marcada como `internal: true` no tiene acceso a Internet. Es util para servicios que solo deben comunicarse entre si, como la capa de base de datos.

## Compose como herramienta declarativa

La naturaleza declarativa de Compose es su principal ventaja: el archivo describe el estado deseado del sistema, y Compose se encarga de alcanzarlo. No es necesario indicar el orden de creacion de los recursos ni gestionar limpiezas. Cuando se ejecuta `docker compose up`, la herramienta calcula las diferencias entre el estado actual y el deseado, y aplica solo los cambios necesarios. Este enfoque es el mismo que utilizan orquestadores mas complejos como Kubernetes, lo que hace de Compose una puerta de entrada natural a la orquestacion profesional.

