# Instalacion inicial

La instalacion de un motor de contenedores es el primer paso para trabajar con la contenerizacion. Docker es el motor mas utilizado, pero Podman es una alternativa creciente que no requiere daemon ni privilegios de root. Ambos son compatibles con el estandar OCI y ejecutan las mismas imagenes.

## Instalacion de Docker

**Linux (Ubuntu/Debian):**

```bash
# Actualizar repositorios
sudo apt update

# Instalar dependencias
sudo apt install -y ca-certificates curl gnupg

# Agregar clave GPG oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Agregar repositorio
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Agregar usuario al grupo docker (para usar sin sudo)
sudo usermod -aG docker $USER
# Cerrar y abrir sesion nuevamente
```

**macOS:**
Descargar Docker Desktop desde [docker.com](https://www.docker.com/products/docker-desktop/).

**Windows:**
Descargar Docker Desktop. Requiere WSL2 (Windows Subsystem for Linux).

## Instalacion de Podman

```bash
# Linux (Ubuntu/Debian)
sudo apt install -y podman

# macOS
brew install podman
podman machine init
podman machine start
```

## Verificacion

```bash
# Verificar version de Docker
docker --version
# Docker version 24.0.7, build afdd53b

# Verificar que el servicio esta corriendo
sudo systemctl status docker

# Ejecutar imagen de prueba
docker run hello-world
# Hello from Docker!
# This message shows that your installation appears to be working correctly.

# Ejecutar contenedor interactivo
docker run -it alpine sh
# / # whoami
# root
# / # exit

# Ver imagenes descargadas
docker images
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# hello-world  latest    d2c6d6d5a7f4   2 weeks ago    13.3kB
# alpine       latest    0551f7c65dae   3 weeks ago    7.74MB
```

## Verificacion de Podman

```bash
# Verificar version
podman --version

# Ejecutar imagen de prueba
podman run hello-world

# Podman funciona sin daemon y sin privilegios de root
# No necesita servicio systemd
```

## Docker Desktop (alternativa grafica)

Docker Desktop proporciona una interfaz grafica para gestionar contenedores, imagenes, volumenes y redes. Incluye Docker Engine, Docker CLI, Docker Compose y Kubernetes.

```bash
# Verificar Docker Desktop
docker compose version
# Docker Compose version v2.21.0

# Kubernetes viene incluido (opcional)
# Settings > Kubernetes > Enable Kubernetes
```

## Solucion de problemas comun

```bash
# Error: "permission denied" al usar docker sin sudo
sudo usermod -aG docker $USER
# Cerrar sesion y volver a abrir

# Error: "Cannot connect to the Docker daemon"
sudo systemctl start docker
sudo systemctl enable docker

# Verificar logs de Docker
sudo journalctl -u docker.service --no-pager | tail -20
```

> La instalacion de Docker o Podman es el punto de entrada al ecosistema de contenedores. Con un solo comando, puedes tener un servidor web, una base de datos o un entorno de desarrollo completo funcionando en tu maquina. La barrera de entrada es minima; las posibilidades son practicamente infinitas.

