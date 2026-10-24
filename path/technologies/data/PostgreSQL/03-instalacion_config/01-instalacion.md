# Instalacion

PostgreSQL esta disponible en los repositorios oficiales de las principales distribuciones Linux, ademas de contar con instaladores nativos para macOS y Windows, imagenes oficiales para Docker y paquetes compilados para sistemas basados en RPM. La eleccion del metodo de instalacion depende del entorno: en desarrollo local se suele optar por el gestor de paquetes del sistema operativo o Docker, mientras que en produccion predominan las instalaciones desde repositorios oficiales o contenedores orquestados con persistencia separada.

## Linux Debian/Ubuntu

El gestor de paquetes `apt` incluye PostgreSQL en los repositorios oficiales desde versiones recientes de ambas distribuciones. El paquete `postgresql-contrib` agrega extensiones y utilidades adicionales que no forman parte del nucleo.

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

Una vez instalado, el servicio se inicia automaticamente y queda registrado en systemd. El cluster por defecto se crea con el nombre `main` y los archivos de configuracion se almacenan en `/etc/postgresql/16/main/`.

```bash
sudo systemctl status postgresql
```

El binario `psql` y el resto de herramientas de administracion quedan disponibles en el PATH del sistema. El servidor escucha en `localhost:5432` por defecto.

## Linux RHEL/CentOS/Fedora

En las distribuciones basadas en RHEL se utiliza `dnf` como gestor de paquetes. PostgreSQL no suele estar presente en los repositorios base de estas distribuciones, por lo que se recomienda anadir el repositorio oficial de PostgreSQL Global Development Group.

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf install -y postgresql16-server postgresql16-contrib
```

A diferencia de Debian, en RHEL es necesario inicializar manualmente el cluster antes de arrancar el servicio por primera vez:

```bash
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable postgresql-16
sudo systemctl start postgresql-16
```

La inicializacion crea los archivos de configuracion en `/var/lib/pgsql/16/data/`. Este directorio contiene `postgresql.conf`, `pg_hba.conf` y el resto de archivos del cluster.

## macOS

Homebrew es el metodo mas extendido para instalar PostgreSQL en macOS. El formula oficial proporciona la version estable junto con las herramientas de linea de comandos.

```bash
brew install postgresql@16
```

Para iniciar el servicio inmediatamente y configurarlo para que arranque en cada inicio de sesion:

```bash
brew services start postgresql@16
```

Homebrew crea el cluster por defecto automaticamente y lo deja listo para aceptar conexiones locales. Los binarios se instalan en `/opt/homebrew/opt/postgresql@16/bin/` en sistemas Apple Silicon, o en `/usr/local/opt/postgresql@16/bin/` en Intel.

## Windows

El instalador oficial proporciona PostgreSQL junto con pgAdmin, Stack Builder y las herramientas de linea de comandos en un unico paquete.

1. Descargar el instalador desde la pagina oficial de PostgreSQL
2. Ejecutar el asistente y elegir los componentes deseados
3. Durante la instalacion se solicita:
   - Directorio de instalacion
   - Puerto (por defecto 5432)
   - Contrasena del superusuario `postgres`
   - Locale del cluster
4. Al finalizar, pgAdmin se abre automaticamente y el servicio de PostgreSQL queda registrado como un servicio de Windows

Las herramientas de linea de comandos se anaden al PATH del sistema durante la instalacion. pgAdmin permite administrar el servidor de forma grafica, crear bases de datos, ejecutar consultas y gestionar usuarios sin necesidad de usar la terminal.

## Docker

La imagen oficial de PostgreSQL permite levantar una instancia en segundos sin instalar nada en el sistema anfitrion. Es la opcion mas rapida para entornos de desarrollo y pruebas.

```bash
docker run -d \
  --name postgres-server \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  postgres:16
```

Para que los datos persistan entre reinicios del contenedor es necesario montar un volumen:

```bash
docker run -d \
  --name postgres-server \
  -e POSTGRES_PASSWORD=postgres \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

Con Docker Compose se define la configuracion de forma declarativa:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

> La imagen oficial expone el puerto 5432 y usa `postgres` como superusuario por defecto. En entornos de produccion es obligatorio cambiar la contrasena, restringir la exposicion de puertos y considerar el uso de redes separadas para los contenedores.

## Verificacion

Dos comandos basicos confirman que la instalacion fue exitosa y que el servidor esta accesible:

```bash
psql --version
```

Muestra la version del cliente `psql`. La salida indica la version del protocolo y la version del servidor con el que es compatible:

```
psql (PostgreSQL) 16.1
```

```bash
pg_isready
```

Comprueba que el servidor esta aceptando conexiones. Responde con la ruta del socket y el puerto, seguido del estado:

```
/tmp:/var/run/postgresql:5432 - accepting connections
```

Si el servidor no responde, se puede iniciar manualmente segun el sistema operativo:

```bash
sudo systemctl start postgresql        # Linux
brew services start postgresql@16      # macOS
```

Para una verificacion mas completa, conectar con `psql` y ejecutar una consulta que confirme la version del servidor en ejecucion:

```bash
sudo -u postgres psql -c "SELECT version();"
```

