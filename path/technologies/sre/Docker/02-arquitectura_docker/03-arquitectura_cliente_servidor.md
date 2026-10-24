# Arquitectura cliente-servidor de Docker

Docker adopta una arquitectura cliente-servidor clasica, donde el cliente (la CLI de Docker) se comunica con un servidor (el demonio dockerd) a traves de una API REST. Esta separacion no es un detalle de implementacion menor: define como se interactua con Docker, como se asegura la comunicacion y como se integran herramientas externas. La API REST de Docker es, de hecho, uno de sus componentes mas importantes, pues permite que cualquier lenguaje o herramienta pueda gestionar contenedores sin depender de la CLI.

## La API REST de Docker

El demonio dockerd expone una API REST en el socket Unix `/var/run/docker.sock` (por defecto) y opcionalmente en un puerto TCP. Esta API sigue el estilo RESTful con recursos como `/containers`, `/images`, `/networks` y `/volumes`, cada uno con operaciones CRUD estandar.

```bash
# Consultar la version de la API directamente
curl --unix-socket /var/run/docker.sock http://localhost/version
# {
#   "Platform": {"Name": "Docker Engine - Community"},
#   "Components": [
#     {"Name": "Engine", "Version": "24.0.7"},
#     {"Name": "containerd", "Version": "1.6.28"},
#     {"Name": "runc", "Version": "1.1.12"}
#   ],
#   "ApiVersion": "1.43",
#   "Arch": "amd64",
#   "Os": "linux",
#   "BuildTime": "2024-01-31T20:10:57.000000000+00:00",
#   "GitCommit": "afdd53b",
#   "GoVersion": "go1.21.6",
#   "MinAPIVersion": "1.12"
# }
```

La API REST permite realizar cualquier operacion que la CLI soporte. De hecho, la CLI no es mas que un cliente que traduce comandos a llamadas HTTP.

```bash
# Listar contenedores via API
curl --unix-socket /var/run/docker.sock http://localhost/containers/json?all=true

# Crear un contenedor via API
curl -X POST --unix-socket /var/run/docker.sock \
  -H "Content-Type: application/json" \
  -d '{"Image":"nginx:latest","Cmd":["nginx","-g","daemon off;"]}' \
  http://localhost/containers/create

# Iniciar el contenedor creado
curl -X POST --unix-socket /var/run/docker.sock \
  http://localhost/containers/<ID>/start
```

## Comunicacion local: socket Unix

La comunicacion local entre el cliente y el demonio se realiza a traves del socket Unix `/var/run/docker.sock`. Este metodo es seguro porque solo los procesos con acceso al sistema de archivos pueden utilizarlo. El socket pertenece al grupo docker, lo que permite controlar quien puede ejecutar comandos Docker.

```bash
# Ver el socket de Docker
ls -la /var/run/docker.sock
# srw-rw---- 1 root docker 0 ... /var/run/docker.sock

# Verificar que el usuario pertenece al grupo docker
groups $USER
# usuario docker
```

El socket Unix es el mecanismo de comunicacion por defecto y el mas seguro para entornos locales. La pertenencia al grupo docker otorga efectivamente privilegios de root, ya que el socket permite ejecutar cualquier operacion sin restricciones adicionales.

> Exponer /var/run/docker.sock dentro de un contenedor es una practica de seguridad extremadamente peligrosa. El contenedor obtiene control total sobre el demonio Docker del anfitrion, pudiendo ejecutar contenedores privilegiados, montar el sistema de archivos del anfitrion y escapar del aislamiento del contenedor. Solo debe hacerse en escenarios muy controlados, como herramientas de CI/CD que necesitan gestionar contenedores desde dentro de otros contenedores.

## Comunicacion remota: TCP con TLS

Para gestionar hosts Docker remotos, el demonio puede configurarse para escuchar en un puerto TCP. Sin embargo, exponer la API sin cifrar y sin autenticacion (puerto 2375) es extremadamente peligroso, ya que cualquiera con acceso a la red podria tomar control del demonio. La configuracion segura requiere TLS (puerto 2376).

```bash
# Configuracion insegura (SOLO para pruebas locales)
# /etc/docker/daemon.json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"]
}

# Configuracion segura con TLS
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
  "tlsverify": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem"
}
```

```bash
# Conectarse a un host remoto con TLS
docker -H tcp://servidor:2376 --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=cert.pem \
  --tlskey=key.pem \
  ps

# Usando variables de entorno para simplificar
export DOCKER_HOST=tcp://servidor:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=/home/usuario/.docker
docker ps
```

## Comunicacion remota por SSH

Una alternativa mas conveniente que TCP/TLS es utilizar SSH como transporte. Docker puede conectarse a un host remoto a traves de SSH, aprovechando la autenticacion y el cifrado que SSH ya proporciona.

```bash
# Conectarse por SSH
docker -H ssh://usuario@servidor ps

# Usar una clave SSH especifica
docker -H ssh://usuario@servidor --ssh ~/.ssh/id_rsa ps

# Variable de entorno
export DOCKER_HOST=ssh://usuario@servidor
docker ps
```

La comunicacion por SSH es la opcion mas recomendada para gestionar hosts remotos, ya que no requiere configurar TLS en el servidor y aprovecha la infraestructura de claves SSH existente.

## Variables de entorno del cliente

El cliente Docker puede configurarse mediante variables de entorno, lo que facilita la integracion con scripts y herramientas de CI/CD.

```bash
# Variables de entorno principales
export DOCKER_HOST=tcp://192.168.1.100:2376     # Host y puerto del demonio
export DOCKER_TLS_VERIFY=1                       # Forzar verificacion TLS
export DOCKER_CERT_PATH=/home/user/.docker       # Directorio de certificados
export DOCKER_CONFIG=/home/user/.docker          # Directorio de configuracion
export DOCKER_CONTEXT=mi-contexto                # Contexto de Docker
export DOCKER_API_VERSION=1.43                   # Version de la API

# La CLI usa estas variables automaticamente
docker ps  # Se conecta al host configurado
```

## Contextos de Docker

Los contextos de Docker permiten definir y conmutar facilmente entre diferentes configuraciones de conexion. Un contexto incluye el host, los certificados TLS y el nombre del contexto.

```bash
# Crear un contexto para un servidor remoto
docker context create remoto \
  --docker "host=ssh://usuario@servidor"

# Crear un contexto para AWS ECS
docker context create aws-ecs \
  --ecs "cluster=mi-cluster"

# Listar contextos
docker context ls

# Usar un contexto
docker context use remoto
docker ps  # Ahora se conecta al servidor remoto

# Ejecutar un comando con un contexto especifico
docker --context remoto ps
```

Los contextos son especialmente utiles en entornos donde se trabaja con multiples clusters (desarrollo, staging, produccion) y se necesita cambiar rapidamente entre ellos sin recordar URLs y rutas de certificados.

## Versionado de la API

La API de Docker esta versionada (actualmente v1.43). El cliente negocia automaticamente la version de la API con el servidor, pero se puede fijar una version especifica para compatibilidad.

```bash
# Ver la version de la API del servidor
docker version --format '{{.Server.APIVersion}}'
# 1.43

# Fijar una version de API especifica
export DOCKER_API_VERSION=1.41
docker ps  # Usa la API v1.41
```

El versionado garantiza que los scripts y herramientas que dependen de la API sigan funcionando aunque el servidor se actualice. Las versiones antiguas de la API se mantienen por compatibilidad retrospectiva.

> La arquitectura cliente-servidor de Docker no es simplemente una decision de diseno, sino el fundamento que permite la integracion con herramientas externas, la gestion remota y la automatizacion. Cualquier operacion que se pueda realizar con la CLI se puede realizar con una llamada HTTP directa a la API, lo que abre la puerta a aplicaciones y scripts en cualquier lenguaje de programacion.

