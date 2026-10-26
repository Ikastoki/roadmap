# Contenedores en diferentes sistemas operativos

Los contenedores nacieron en Linux y estan intimamente ligados al kernel de Linux. Ejecutar contenedores nativos en Windows o macOS requiere una capa adicional que proporcione el kernel Linux necesario. Esta realidad tiene implicaciones importantes en rendimiento, configuracion y comportamiento.

## Linux: ejecucion nativa

En Linux, los contenedores se ejecutan directamente sobre el kernel del host. No hay capas intermedias, no hay emulacion, no hay sobrecarga adicional. Namespaces, cgroups y OverlayFS son funcionalidades nativas del kernel.

```bash
# En Linux, Docker ejecuta contenedores nativamente
docker run -d nginx
# El proceso nginx se ejecuta directamente sobre el kernel de Linux
# Sin capas intermedias
```

## Windows: WSL2

Docker Desktop en Windows utiliza WSL2 (Windows Subsystem for Linux) para ejecutar un kernel de Linux real dentro de una VM ligera. Los contenedores se ejecutan sobre este kernel de Linux, no sobre el kernel de Windows.

```
Windows 11
├── Docker Desktop
│   ├── WSL2 (VM ligera con kernel Linux)
│   │   ├── Contenedor 1: nginx (Linux)
│   │   ├── Contenedor 2: node.js (Linux)
│   │   └── Contenedor 3: postgres (Linux)
│   └── API de Docker
```

**Contenedores Windows nativos:**
Windows tambien puede ejecutar contenedores Windows, que usan el kernel de Windows y contienen el SO Windows completo. Son mucho mas pesados que los contenedores Linux.

```bash
# Contenedor Windows nativo
docker run -d mcr.microsoft.com/windows/servercore:ltsc2022 powershell
# Este contenedor ocupa ~5 GB (contiene Windows Server Core)
```

```bash
# Contenedor Linux en Windows (a traves de WSL2)
docker run -d nginx:alpine
# Este contenedor ocupa ~40 MB
# Se ejecuta sobre el kernel Linux dentro de WSL2
```

## macOS: Virtualization.framework

En macOS, Docker Desktop usa el framework de virtualizacion de Apple (o HyperKit en versiones anteriores) para ejecutar una VM Linux con el kernel necesario.

```
macOS
├── Docker Desktop
│   ├── VM Linux (Virtualization.framework)
│   │   ├── Contenedor 1: nginx
│   │   └── Contenedor 2: redis
│   └── socket: docker.sock
```

## Diferencias de rendimiento

| SO | Metodo | Sobrecarga |
|---|---|---|
| Linux | Nativo | ~0% |
| Windows (WSL2) | VM ligera | ~5-10% |
| macOS (Virtualization.framework) | VM ligera | ~5-15% |
| Windows (contenedor Windows) | Kernel Windows | ~10-20% |

## Implicaciones para desarrollo

```bash
# Desarrollar en macOS con Docker
# Los archivos del host (macOS) se montan en la VM Linux
# El rendimiento de archivos montados es menor que en Linux nativo

# Solucion: usar un volume en lugar de bind mount para E/S intensiva
docker run -v mi-datos:/app/data postgres

# O usar Docker Desktop con virtiofs (mejor rendimiento)
```

## Podman en diferentes SO

```bash
# Podman en Linux: ejecucion nativa
podman run -d nginx

# Podman en macOS: a traves de Podman Machine (VM)
podman machine init
podman machine start
podman run -d nginx

# Podman en Windows: a traves de Podman Machine (WSL2)
podman machine init
podman machine start
podman run -d nginx
```

> Los contenedores son nativos de Linux. En Windows y macOS, se ejecutan a traves de una VM que proporciona el kernel Linux necesario. Esta realidad no es un problema en la practica: la VM es transparente y el rendimiento es aceptable para desarrollo y pruebas. Para produccion, Linux sigue siendo la plataforma preferida para contenedores.

