# Instalacion y configuracion inicial

Antes de comenzar a utilizar un sistema de control de versiones, es necesario instalar el cliente en el sistema operativo y configurar la identidad del autor. Esta configuracion basal es compartida por todos los repositorios del sistema y permite la trazabilidad de los cambios.

Git es el VCS predominante en la industria; la mayoria de las instrucciones de esta seccion se centran en el y su configuracion.

## Instalacion de Git

### Linux

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install git

# RHEL/CentOS/Fedora
sudo dnf install git

# Arch Linux
sudo pacman -S git
```

### macOS

```bash
# Via Homebrew (recomendado)
brew install git

# Via Xcode Command Line Tools
xcode-select --install
```

### Windows

```bash
# Installer oficial desde git-scm.com
# Incluye Git Bash (terminal Unix-like) y Git GUI

# Via Chocolatey
choco install git

# Via winget
winget install Git.Git
```

### Verificar instalacion

```bash
git --version
# git version 2.43.0
```

## Configuracion de identidad

La identidad del autor se almacena en la configuracion global de Git y se anade automaticamente a cada commit que realices. Es fundamental para la trazabilidad: cada cambio en el historial lleva asociado un nombre y un correo electronico.

```bash
# Configuracion global (se aplica a todos los repositorios)
git config --global user.name "Tu Nombre"
git config --global user.email "tu@email.com"

# Verificar configuracion
git config --global --list

# Configuracion por repositorio (override de la global)
git config user.name "Nombre del Proyecto"
git config user.email "proyecto@empresa.com"
```

## Configuracion recomendada

```bash
# Editor de texto predeterminado para commits
git config --global core.editor "vim"
# Alternativas: nano, code --wait (VS Code), subl -n -w (Sublime)

# Configurar el nombre de la rama por defecto
git config --global init.defaultBranch main

# Habilitar colores en la salida
git config --global color.ui auto

# Configurar el programa de merge
git config --global merge.tool vimdiff

# Habilitar rebase automatico al hacer pull
git config --global pull.rebase true
```

## Estructura de archivos de configuracion

Git almacena la configuracion en tres niveles:

- **`~/.gitconfig`** (global): configuracion del usuario para todos los repositorios.
- **`.git/config`** (local): configuracion especifica del repositorio.
- **`/etc/gitconfig`** (sistema): configuracion por defecto del sistema.

```bash
# Ver configuracion efectiva (combina los tres niveles)
git config --list --show-origin

# Editar configuracion global manualmente
git config --global -e
```

> La configuracion de Git es el primer paso para una协作 efectiva. Un nombre y correo electronico correctos garantizan que cada commit este debidamente atribuido. Invertir unos minutos en configurar Git correctamente al inicio evita problemas de trazabilidad mas adelante.

