# Docker en diferentes sistemas operativos

Docker se ejecuta en Linux, macOS y Windows, pero lo hace de formas fundamentalmente diferentes en cada plataforma. La razon es simple: los contenedores de Docker estan construidos sobre tecnologias del kernel de Linux (namespaces, cgroups, sistemas de archivos union), y ni macOS ni Windows tienen estas caracteristicas en su kernel nativo. Cada sistema operativo resuelve esta limitacion de una manera particular, con implicaciones en rendimiento, compatibilidad y experiencia de uso.

## Docker en Linux: ejecucion nativa

En Linux, Docker se ejecuta de forma nativa. El demonio dockerd se comunica directamente con el kernel del sistema, creando namespaces, configurando cgroups y montando sistemas de archivos union sin capas de virtualizacion intermedias. Esto proporciona el mejor rendimiento posible.

```bash
# En Linux, Docker Engine se instala directamente sobre el SO
# No se requiere virtualizacion adicional
docker run -d --name web -p 80:80 nginx

# El contenedor se ejecuta directamente sobre el kernel del anfitrion
# Rendimiento practicamente identico al nativo
```

El rendimiento en Linux es virtualmente identico al de procesos nativos. La sobrecarga de los namespaces y cgroups es minima, tipicamente inferior al 1 % en CPU y memoria. El acceso a dispositivos y la E/S de red son casi nativos.

```bash
# Verificar que Docker usa el kernel nativo
docker info | grep -E "Kernel Version|Operating System"
# Kernel Version: 6.5.0-21-generic
# Operating System: Ubuntu 22.04.3 LTS
```

## Docker en macOS: la capa de virtualizacion

macOS no tiene soporte nativo para contenedores Linux, ya que su kernel (XNU) no implementa namespaces ni cgroups al estilo Linux. Para resolver esto, Docker Desktop para macOS ejecuta una maquina virtual ligera de Linux utilizando el framework de virtualizacion nativo de Apple (Hypervisor.framework) o VirtualBox en versiones antiguas.

```bash
# En macOS, Docker Desktop crea una VM Linux oculta
# Los contenedores se ejecutan realmente dentro de esa VM
docker run hello-world
# El contenedor se ejecuta en la VM, no directamente en macOS
```

La VM de Linux que Docker Desktop utiliza se llama "Docker Desktop VM" y ejecuta una distribucion Alpine Linux personalizada. Esta VM incluye el demonio Docker, containerd y runc. El usuario no interactua directamente con la VM, pero puede acceder a ella en caso necesario.

```bash
# Acceder a la VM de Docker Desktop en macOS
docker run -it --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh
# Dentro de la VM Docker, se puede ver el proceso dockerd y los contenedores reales
```

El rendimiento en macOS es ligeramente inferior al de Linux debido a la capa de virtualizacion. La traduccion de llamadas al sistema entre macOS y la VM Linux introduce latencia, especialmente en operaciones de E/S de archivos y red.

```bash
# En macOS, montar codigo fuente local en el contenedor
# La VM debe acceder a los archivos en el sistema de archivos de macOS
docker run -d -v $(pwd):/app node:20 npm run dev
# La sincronizacion de archivos entre macOS y la VM puede ser lenta
```

> En macOS con chip Apple Silicon (M1, M2, M3), Docker Desktop ofrece imagenes nativas para ARM, que se ejecutan sin emulacion. Si una imagen no tiene variante ARM, Docker puede emular AMD64 mediante Rosetta 2, pero con penalizacion de rendimiento. Se recomienda usar imagenes con soporte multiarquitectura (como las oficiales de Docker Hub) para evitar la emulacion.

## Docker en Windows: WSL 2 y Hyper-V

En Windows, Docker Desktop ofrece dos opciones de backend: WSL 2 (recomendado) y Hyper-V. El backend de WSL 2 ejecuta una distribucion completa de Linux dentro de una maquina virtual ligera gestionada por Windows.

```powershell
# En Windows con WSL 2, los contenedores Linux se ejecutan en WSL
docker run hello-world
# El contenedor se ejecuta en la distribucion WSL de Docker

# Verificar que WSL 2 esta en uso
wsl -l -v
#  NAME                   STATE           VERSION
#  docker-desktop         Running         2
#  docker-desktop-data    Running         2
```

Con WSL 2, el rendimiento es significativamente mejor que con Hyper-V, acercandose al rendimiento nativo de Linux. La integracion entre Windows y el subsistema Linux es fluida, permitiendo acceder a archivos de Windows desde los contenedores y viceversa.

```powershell
# Los contenedores Windows se ejecutan directamente en el kernel de Windows
# Esto requiere imagenes base de Windows, que son significativamente mas grandes
docker pull mcr.microsoft.com/windows/servercore:ltsc2022
# ~5 GB para la imagen base
```

Los contenedores Windows nativos (no Linux) se ejecutan directamente sobre el kernel de Windows sin virtualizacion. Sin embargo, las imagenes base de Windows son mucho mas grandes (varios gigabytes) que sus equivalentes Linux (megabytes), y el ecosistema de imagenes Windows es mucho menos extenso.

## Contenedores Windows frente a contenedores Linux

Existen dos tipos de contenedores en Windows: contenedores Linux (ejecutados via WSL 2 o Hyper-V) y contenedores Windows (nativos del kernel de Windows). Los contenedores Windows pueden ser de dos modos:

```powershell
# Modo Windows Server Core: imagen completa, compatible con aplicaciones .NET
docker pull mcr.microsoft.com/windows/servercore:ltsc2022

# Modo Nano Server: imagen minima, para aplicaciones .NET Core ligeras
docker pull mcr.microsoft.com/windows/nanoserver:ltsc2022
```

```powershell
# Cambiar entre modos de contenedor en Windows
# En PowerShell como administrador:
& 'C:\Program Files\Docker\Docker\DockerCli.exe' -SwitchLinuxEngine
& 'C:\Program Files\Docker\Docker\DockerCli.exe' -SwitchWindowsEngine
```

Los contenedores Windows requieren que la version del kernel del anfitrion coincida exactamente con la version de la imagen base. Esto significa que una imagen construida para Windows Server 2022 no funcionara en Windows Server 2019, lo que limita la portabilidad.

## Diferencias de rendimiento entre plataformas

Las diferencias de rendimiento entre plataformas son notables y deben tenerse en cuenta al desarrollar y desplegar aplicaciones con Docker.

```bash
# Prueba de rendimiento simple: I/O de disco
# En Linux (nativo):
time docker run --rm alpine dd if=/dev/zero of=/tmp/test bs=1M count=1000
# ~0.5s

# En macOS (con VM):
time docker run --rm alpine dd if=/dev/zero of=/tmp/test bs=1M count=1000
# ~2-3s (penalizacion por virtualizacion)

# En Windows (WSL 2):
time docker run --rm alpine dd if=/dev/zero of=/tmp/test bs=1M count=1000
# ~1-2s (mejor que macOS, peor que Linux)
```

Las operaciones de E/S de archivos son las mas afectadas por la virtualizacion en macOS y Windows. Las operaciones de CPU y red tienen penalizaciones menores.

## Compatibilidad de imagenes entre plataformas

Una de las ventajas de Docker es que las imagenes son portables entre plataformas, pero con matices. Una imagen construida para Linux se ejecuta en cualquier plataforma (Linux directamente, macOS via VM, Windows via WSL 2). Una imagen de Windows solo se ejecuta en Windows.

```bash
# Ver las plataformas para las que esta disponible una imagen
docker buildx imagetools inspect node:20
# Name:      docker.io/library/node:20
# MediaType: application/vnd.docker.distribution.manifest.list.v2+json
# Platform:  linux/amd64, linux/arm/v6, linux/arm/v7, linux/arm64, linux/ppc64le, linux/s390x, linux/386
# La imagen node:20 esta disponible para multiples arquitecturas
```

Docker Hub y otros registros modernos soportan manifiestos multiarquitectura (manifest lists). Una misma etiqueta como `node:20` puede referirse a imagenes diferentes para cada arquitectura. Docker descarga automaticamente la variante correcta para la plataforma en la que se ejecuta.

```bash
# Construir imagenes multiarquitectura con buildx
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t usuario/mi-app:1.0 --push .
```

> La eleccion del sistema operativo para Docker depende del caso de uso. Linux ofrece el mejor rendimiento y es la opcion natural para produccion. macOS y Windows son excelentes para desarrollo, pero es importante ser consciente de las diferencias de rendimiento, especialmente en E/S de archivos. Para equipos de desarrollo mixtos, el uso de imagenes multiarquitectura garantiza que todos puedan trabajar con las mismas imagenes independientemente de su plataforma.

