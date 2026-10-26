# Personalizacion del entorno de Git

Git permite configurar multiples aspectos de su comportamiento: desde el editor de texto que usa para mensajes de commit hasta aliases para comandos frecuentes, colores de salida y estrategias de fusion por defecto. Una buena personalizacion reduce tiempo, evita errores y hace que Git se adapte a tu flujo de trabajo en lugar de tener que adaptarte tu a el.

## Configuracion global

La configuracion global se almacena en `~/.gitconfig` y aplica a todos los repositorios del usuario.

```bash
# Identidad del autor
git config --global user.name "Tu Nombre"
git config --global user.email "tu@email.com"

# Editor de texto predeterminado
git config --global core.editor "code --wait"    # VS Code
git config --global core.editor "vim"             # Vim
git config --global core.editor "nano"            # Nano

# Estrategia de fusion por defecto
git config --global pull.rebase true              # Siempre rebase al pull
git config --global merge.tool "vscode"           # Herramienta de conflictos

# Colores en la salida
git config --global color.ui auto
```

## Configuracion por repositorio

La configuracion local sobreescribe la global y se almacena en `.git/config` del repositorio.

```bash
# Configurar autor diferente para un proyecto empresarial
git config user.email "nombre@empresa.com"

# Estrategia de fusion especifica para este repositorio
git config merge.conflictstyle diff3
```

## Aliases para comandos frecuentes

Los aliases son atajos que reducen comandos largos a abreviaturas.

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"
git config --global alias.amend "commit --amend --no-edit"
git config --global alias.wip "!git add . && git commit -m 'WIP'"
```

```bash
# Ahora puedes usar:
git st          # en lugar de git status
git lg          # en lugar de git log --oneline --graph --all --decorate
git last        # en lugar de git log -1 HEAD
git amend       # en lugar de git commit --amend --no-edit
```

## Configuracion de merge con diff3

Cuando ocurre un conflicto, la configuracion `diff3` muestra tres versiones: la original, la tuya y la de la otra persona. Esto facilita enormemente la resolucion.

```bash
git config --global merge.conflictstyle diff3
```

**Sin diff3:**
```
<<<<<<< HEAD
tu cambio
=======
cambio de la otra persona
>>>>>>> feature
```

**Con diff3:**
```
<<<<<<< HEAD
tu cambio
|||||||.merge-base
version original
=======
cambio de la otra persona
>>>>>>> feature
```

## Configuracion recomendada para equipos

```bash
# ~/.gitconfig
[user]
    name = Tu Nombre
    email = tu@email.com
[core]
    editor = code --wait
    autocrlf = input          # Linux/macOS
    whitespace = fix
[pull]
    rebase = true
[merge]
    conflictstyle = diff3
    tool = vscode
[diff]
    tool = vscode
[color]
    ui = auto
[alias]
    st = status
    lg = log --oneline --graph --all --decorate
    last = log -1 HEAD
    amend = commit --amend --no-edit
    undo = reset --soft HEAD~1
```

> Personalizar Git no es un lujo: es una inversion en productividad. Un alias que ahorra 3 segundos por uso se traduce en minutos al dia y horas al mes. La configuracion correcta elimina friccion del flujo de trabajo y reduce la probabilidad de errores por comandos largos o mal recordados.

