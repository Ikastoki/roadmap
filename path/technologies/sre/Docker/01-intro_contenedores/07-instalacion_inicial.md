# Instalacion inicial de Docker

La instalacion de Docker ha evolucionado significativamente desde sus primeras versiones. En la actualidad, Docker ofrece multiples opciones de instalacion adaptadas a diferentes sistemas operativos y casos de uso. La eleccion del metodo de instalacion depende del sistema operativo anfitrion, de si se necesita Docker Desktop o Docker Engine, y de los requisitos de integracion con el sistema existente. Una instalacion correcta es el primer paso para un entorno de desarrollo robusto y seguro.

## Docker Desktop frente a Docker Engine

Docker Desktop es una aplicacion que incluye Docker Engine, Docker CLI, Docker Compose, y en macOS y Windows, una maquina virtual ligera que ejecuta Linux para albergar los contenedores. Docker Engine es el componente de servidor que se ejecuta directamente sobre Linux, sin necesidad de virtualizacion.

```bash
# Verificar la instalacion
docker --version
# Docker version 24.0.7, build afdd53b
docker compose version
# Docker Compose version v2.24.2
```

En Linux, se recomienda instalar Docker Engine directamente, ya que ofrece mejor rendimiento al no requerir una capa de virtualizacion. En macOS y Windows, Docker Desktop es la opcion recomendada, pues gestiona automaticamente la maquina virtual necesaria para ejecutar contenedores Linux.

## Instalacion en Ubuntu y derivados

La forma recomendada de instalar Docker Engine en Ubuntu es usando el repositorio oficial de Docker. Esto garantiza acceso a las ultimas versiones y actualizaciones de seguridad.

```bash
# 1. Eliminar versiones antiguas
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# 2. Instalar dependencias
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# 3. Anadir la clave GPG oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 4. Anadir el repositorio
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# 5. Instalar Docker Engine
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 6. Verificar la instalacion
sudo docker run hello-world
```

> No ejecutar `docker run` con sudo. Anadir el usuario al grupo docker para evitar usar sudo en cada comando. Esto tiene implicaciones de seguridad: cualquier usuario en el grupo docker tiene permisos equivalentes a root.

```bash
# Anadir el usuario al grupo docker
sudo usermod -aG docker $USER

# Cerrar sesion y volver a iniciarla, o ejecutar:
newgrp docker

# Verificar que funciona sin sudo
docker run hello-world
```

## Instalacion en macOS

En macOS, Docker Desktop es la opcion oficial. Se puede descargar desde Docker Hub o instalar via Homebrew.

```bash
# Instalacion con Homebrew
brew install --cask docker

# Iniciar Docker Desktop
open /Applications/Docker.app

# Verificar la instalacion
docker --version
docker compose version
```

Docker Desktop en macOS utiliza el framework de virtualizacion nativo de Apple (Hypervisor.framework) o VirtualBox, dependiendo de la configuracion. En Mac con chip Apple Silicon (M1, M2, M3), Docker Desktop ofrece imagenes nativas para ARM, ademas de emulacion para imagenes AMD64 mediante Rosetta 2.

## Instalacion en Windows

Docker Desktop para Windows requiere WSL 2 (Windows Subsystem for Linux 2) o Hyper-V habilitado. WSL 2 es la opcion recomendada por su mejor rendimiento y compatibilidad.

```powershell
# En PowerShell como administrador:
# 1. Habilitar WSL 2
wsl --install

# 2. Descargar e instalar Docker Desktop para Windows
# Desde https://desktop.docker.com/win/stable/Docker%20Desktop%20Installer.exe

# 3. Verificar la instalacion
docker --version
docker run hello-world
```

Docker Desktop en Windows ejecuta los contenedores en una maquina virtual de Linux gestionada por WSL 2. Esto proporciona un rendimiento casi nativo y total compatibilidad con imagenes Linux.

## Instalacion headless en servidores

Para servidores sin interfaz grafica, la instalacion es similar a la de Ubuntu, pero con consideraciones adicionales de configuracion para entornos de produccion.

```bash
# En servidores Ubuntu/Debian headless
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Configurar Docker para arranque automatico
sudo systemctl enable docker
sudo systemctl start docker

# Verificar el estado
sudo systemctl status docker
```

En servidores de produccion, es importante configurar Docker para que se ejecute como servicio systemd y configurar los limites de recursos adecuados.

## Verificacion y diagnostico

Despues de la instalacion, es importante verificar que todos los componentes funcionan correctamente.

```bash
# Verificar informacion del sistema Docker
docker info
# Muestra informacion sobre:
# - Numero de contenedores e imagenes
# - Driver de almacenamiento
# - Version del kernel
# - Cgroups habilitados

# Probar ejecucion de un contenedor
docker run --rm hello-world
# Debe mostrar un mensaje de bienvenida y confirmar que Docker funciona

# Verificar Docker Compose
docker compose version

# Verificar BuildKit
docker buildx version
```

## Configuracion post-instalacion

Despues de instalar Docker, hay varias configuraciones recomendadas para un uso optimo y seguro.

```bash
# 1. Configurar el driver de almacenamiento (opcional)
# En produccion, overlay2 es el recomendado
sudo docker info | grep "Storage Driver"

# 2. Configurar el logging driver
# Por defecto es json-file; para produccion se recomienda un driver externo
sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

# 3. Configurar mirror de Docker Hub en China o redes corporativas
# En /etc/docker/daemon.json
{
  "registry-mirrors": ["https://mirror.example.com"]
}

# 4. Reiniciar Docker para aplicar cambios
sudo systemctl restart docker
```

> La configuracion de Docker se realiza principalmente a traves del archivo `/etc/docker/daemon.json`. Las opciones incluyen limites de recursos globales, drivers de almacenamiento, configuracion de red y proxies. Los cambios requieren reiniciar el servicio Docker.

## Solucion de problemas comunes

Algunos problemas frecuentes durante la instalacion y sus soluciones:

```bash
# Error: Cannot connect to the Docker daemon
# Causa: El servicio Docker no esta corriendo
sudo systemctl start docker
sudo systemctl enable docker

# Error: permission denied while trying to connect
# Causa: El usuario no esta en el grupo docker
sudo usermod -aG docker $USER
# Cerrar sesion y volver a entrar

# Error: no matching manifest for linux/arm64 in the manifest list entries
# Causa: La imagen solo esta disponible para AMD64
docker run --platform linux/amd64 nginx

# Error: disk space full
# Docker almacena datos en /var/lib/docker
# Verificar espacio:
sudo du -sh /var/lib/docker/
# Limpiar recursos no usados:
docker system prune -a --volumes
```

> Docker no es simplemente un programa que se instala y se olvida. El motor de Docker requiere mantenimiento: actualizaciones periodicas, limpieza de recursos, monitoreo de logs y, en entornos de produccion, configuracion de backup de `/var/lib/docker`. Una instalacion bien configurada es la base de un ecosistema de contenedores estable y seguro.

