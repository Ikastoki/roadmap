# Instalacion de MongoDB

Un administrador de bases de datos se enfrenta a la decision de instalar MongoDB en entornos que van desde el laptop de desarrollo hasta clusters de produccion distribuidos en varias regiones. Cada plataforma impone su propio mecanismo de instalacion, gestion de dependencias y modelo de ejecucion, y conocer las particularidades de cada una permite elegir el enfoque correcto segun el contexto operativo.

## Linux (Ubuntu/Debian)

La distribucion oficial de MongoDB para sistemas basados en Debian utiliza el gestor de paquetes `apt` con verificacion de firma mediante GPG. Este metodo garantiza que los paquetes instalados provienen directamente de MongoDB Inc. y no han sido alterados.

```bash
# Importar la clave GPG oficial
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor

# Agregar el repositorio de MongoDB
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
http://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" | \
   sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Actualizar e instalar
sudo apt-get update
sudo apt-get install -y mongodb-org
```

La clave GPG se importa en formato `--dearmor` para que apt pueda verificarla en formato binario. El repositorio se configura con la opcion `signed-by` que vincula explicitamente la clave al repositorio, evitando ataques de suplantacion.

Una vez instalado, MongoDB se gestiona como un servicio systemd:

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod
```

El binario `mongod` se ejecuta bajo el usuario `mongodb` y almacena los datos por defecto en `/var/lib/mongodb`. La configuracion reside en `/etc/mongod.conf`.

> En sistemas Debian 12 (Bookworm) o superior, puede ser necesario instalar `gnupg` y `curl` previamente si no estan disponibles: `sudo apt-get install -y gnupg curl`.

## RHEL/CentOS (dnf)

En la familia Red Hat, el gestor `dnf` reemplaza a `yum` como herramienta de gestion de paquetes. MongoDB se distribuye en formato RPM con su propio repositorio.

```bash
# Crear el archivo de repositorio
sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo <<EOF
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/\$releasever/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF

# Instalar MongoDB
sudo dnf install -y mongodb-org
```

La variable `$releasever` se expande automaticamente al numero de version de RHEL, Rocky Linux o AlmaLinux. La directiva `gpgcheck=1` fuerza la verificacion de la firma del paquete antes de instalarlo.

El servicio se gestiona igualmente con systemd:

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

## macOS (brew)

En macOS, Homebrew es el gestor de paquetes de facto. MongoDB ofrece un tap oficial que simplifica la instalacion y la gestion de versiones.

```bash
# Agregar el tap oficial de MongoDB
brew tap mongodb/brew

# Instalar la version社区
brew install mongodb-community@7.0

# Iniciar el servicio
brew services start mongodb-community@7.0
```

El tap `mongodb/brew` contiene las formulas mantenidas por MongoDB Inc. La instalacion incluye `mongod`, `mongos` y `mongosh`. Los archivos de configuracion se localizan en `/usr/local/etc/mongod.conf` (Apple Silicon: `/opt/homebrew/etc/mongod.conf`).

> En macOS, los logs se escriben por defecto en `/usr/local/var/log/mongodb/` (o la ruta equivalente segun la arquitectura) y la base de datos en `/usr/local/var/mongodb/`.

## Windows (MSI)

Windows utiliza un instalador MSI que ofrece una interfaz grafica para configurar los parametros iniciales del servidor. La instalacion puede realizarse tanto mediante el asistente grafico como desde linea de comandos para entornos automatizados.

```bash
# Instalacion silenciosa desde PowerShell (modo administrador)
msiexec.exe /q /i mongodb-windows-x86_64-7.0.msi ^
    INSTALLLOCATION="C:\Program Files\MongoDB\Server\7.0\" ^
    ADDLOCAL="all"
```

El instalador MSI ofrece las siguientes opciones:
- **Complete**: instala MongoDB, mongosh y las Database Tools
- **Custom**: permite seleccionar componentes individuales
- **Service**: configura MongoDB como servicio de Windows

Una vez instalado, es necesario crear los directorios de datos y logs:

```cmd
md C:\data\db
md C:\data\log
```

MongoDB se ejecuta como servicio de Windows:

```cmd
net start MongoDB
```

## Docker

La imagen oficial de MongoDB en Docker Hub proporciona el entorno mas aislado y reproducible. Cada version de MongoDB tiene su propio tag, y la configuracion se inyecta mediante variables de entorno o volumenes.

```bash
# Ejecutar MongoDB 7.0 en un contenedor
docker run --name mongodb \
    -p 27017:27017 \
    -v mongodb_data:/data/db \
    -d mongo:7
```

El volumen `mongodb_data` persiste los datos incluso despues de eliminar el contenedor. El mapeo de puertos expone el puerto 27017 hacia el host. La imagen `mongo:7` ejecuta `mongod` por defecto.

Para configuraciones mas avanzadas se puede montar un archivo de configuracion personalizado:

```bash
docker run --name mongodb \
    -p 27017:27017 \
    -v mongodb_data:/data/db \
    -v ./mongod.conf:/etc/mongod.conf \
    -d mongo:7 --config /etc/mongod.conf
```

> Las imagenes oficiales de MongoDB para Docker estan basadas en Ubuntu y mantienen la misma estructura de directorios que una instalacion nativa. La imagen `mongo:7` corresponde a la version 7.0.x mas reciente.

La autenticacion se habilita pasando variables de entorno para el usuario inicial:

```bash
docker run --name mongodb \
    -e MONGO_INITDB_ROOT_USERNAME=admin \
    -e MONGO_INITDB_ROOT_PASSWORD=secret \
    -d mongo:7
```

## MongoDB Atlas

Atlas es el servicio de base de datos como servicio (DBaaS) de MongoDB. Elimina la necesidad de gestionar la instalacion, el aprovisionamiento y la operacion del servidor, ofreciendo un cluster gestionado en la nube.

El nivel gratuito **M0** (free tier) proporciona:
- 512 MB de almacenamiento
- Replica set de 3 nodos compartidos
- Sin necesidad de tarjeta de credito para empezar
- Cifrado TLS en todas las conexiones
- Backup automatizado (snapshots cada 6 horas)

La cadena de conexion se obtiene desde la interfaz de Atlas y sigue el formato:

```
mongodb+srv://<usuario>:<contraseña>@<cluster>.mongodb.net/<base_datos>?retryWrites=true&w=majority
```

El prefijo `+srv` permite la resolucion DNS del cluster sin necesidad de especificar direcciones IP, facilitando la reconexion ante cambios en la topologia.

> El free tier M0 no soporta escalado vertical ni horizontal. Si la aplicacion crece mas alla de 512 MB, es necesario migrar a un cluster dedicado M2 o superior.

