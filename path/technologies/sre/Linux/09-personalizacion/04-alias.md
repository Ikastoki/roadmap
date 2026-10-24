# Alias

Un alias es un nombre corto que la shell expande por un comando mas largo antes de ejecutarlo. Es un mecanismo de abreviacion puramente textual: la shell reemplaza el alias por su definicion en el historial de comandos antes de que el interprete analice la linea. No es una variable ni una funcion; es una sustitucion directa de cadena.

Los alias eliminan la friccion de escribir comandos largos o con opciones repetitivas. Un administrador que ejecuta `docker ps` constantemente puede aliasarlo a `dps`, y cada vez que escriba `dps` la shell ejecutara `docker ps`. La ganancia es aparentemente menor, pero multiplicada por decenas de ejecuciones diarias el ahorro es significativo.

## Definicion basica

Se usa el comando interno `alias` de la shell:

```bash
# Sintaxis: alias nombre='comando'
alias ll='ls -la'
alias la='ls -A'
alias grep='grep --color=auto'
alias rm='rm -i'
```

La definicion usa comillas simples para evitar que la shell expanda variables o globs en el momento de definir el alias. Si se usan comillas dobles, el shell expande `$` y `` ` `` al definir, no al ejecutar:

```bash
# Definir alias con comillas simples (correcto)
alias dir='echo $PWD'
# Cada vez que se ejecute, mostrara el directorio actual

# Definir alias con comillas dobles (incorrecto)
alias dir="echo $PWD"
# Se expande $PWD al definir, queda fijo al directorio actual
```

## Listar alias activos

El comando `alias` sin argumentos muestra todos los alias definidos en la sesion actual:

```bash
alias
# alias ll='ls -la'
# alias la='ls -A'
# alias grep='grep --color=auto'
# alias rm='rm -i'
```

Para conocer la definicion de un alias concreto:

```bash
alias ll
# alias ll='ls -la'
```

## Eliminar alias

`unalias` elimina un alias de la sesion actual:

```bash
unalias ll        # eliminar un alias
unalias -a        # eliminar todos los alias
```

## Alias con argumentos

Los alias puros no aceptan argumentos de forma nativa. Cuando se escribe `ll /etc`, la shell expande el alias a `ls -la /etc` -- los argumentos se anaden al final del comando expandido. Esto funciona para comandos que aceptan argumentos al final, pero falla si se necesita insertar argumentos en medio.

Para alias que requieren argumentos en posiciones intermedias se necesitan funciones de shell:

```bash
# Funcion que busca en el historial (no se puede con alias)
histgrep() {
    history | grep "$1"
}

# Funcion que crea y entra en un directorio
mkcd() {
    mkdir -p "$1" && cd "$1"
}
```

Las funciones son mas potentes que los alias porque admiten logica condicional, bucles y parametros posicionales. Se definen en los mismos archivos de configuracion que los alias.

## Persistencia en .bashrc / .zshrc

Los alias definidos con `alias` en el terminal solo viven durante la sesion actual. Al cerrar la terminal, desaparecen. Para hacerlos permanentes se anaden al archivo de configuracion de la shell correspondiente.

### Bash: ~/.bashrc

```bash
# Editar el archivo de configuracion de bash
nano ~/.bashrc

# Anadir los alias al final del archivo
alias ll='ls -la'
alias la='ls -A'
alias grep='grep --color=auto'
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
alias df='df -h'
alias du='du -h'
alias free='free -h'
alias psg='ps aux | grep'

# Recargar la configuracion sin cerrar la sesion
source ~/.bashrc
```

### Zsh: ~/.zshrc

```zsh
# En ~/.zshrc (Z shell)
alias ll='ls -la'
alias la='ls -A'
alias grep='grep --color=auto'
alias rm='rm -i'
alias -g G='| grep'        # alias global (zsh)
```

> Zsh permite alias globales con `-g`, que se expanden en cualquier posicion de la linea de comandos, no solo al inicio. `echo hola G error` se expandiria a `echo hola | grep error`. Esto no existe en bash.

### Diferencia entre ~/.bashrc, ~/.bash_profile y ~/.profile

La confusion sobre que archivo editar es habitual. La regla general:

- **`~/.bashrc`**: se ejecuta en cada nueva terminal interactiva (no login). Es donde van alias, funciones y configuracion del prompt.
- **`~/.bash_profile`** o **`~/.profile`**: se ejecuta solo al iniciar sesion (login shell). Normalmente contiene variables de entorno como `PATH`, `EDITOR`.
- **`~/.bash_logout`**: se ejecuta al cerrar la sesion.

La mayoria de distribuciones configuran `~/.bash_profile` para que ejecute `~/.bashrc` automaticamente. Los alias deben ir en `~/.bashrc` para que esten disponibles en todas las terminales.

```bash
# Verificar que ~/.bashrc se esta cargando
echo "alias test='echo funcionando'" >> ~/.bashrc
source ~/.bashrc
test
# funcionando
```

## Alias por sistema: /etc/bash.bashrc

Los alias globales para todos los usuarios del sistema se definen en archivos como `/etc/bash.bashrc` (Debian/Ubuntu) o `/etc/bashrc` (Red Hat). Requieren privilegios de superusuario para editarse:

```bash
sudo nano /etc/bash.bashrc
```

Un alias definido en el archivo del sistema puede sobrescribirse en `~/.bashrc` del usuario porque los archivos de usuario se procesan despues.

## Ejemplos de alias utiles

```bash
# Navegacion
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'
alias home='cd ~'
alias docs='cd ~/Documentos'

# Listados
alias l='ls -CF'
alias la='ls -A'
alias ll='ls -la'
alias lr='ls -laRt'           # recursivo por fecha
alias lt='ls -lart'           # por fecha (mas reciente al final)

# Seguridad (evitar sobrescritura accidental)
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
alias chmod='chmod --preserve-root'
alias chown='chown --preserve-root'

# Legibilidad
alias df='df -h'
alias du='du -h'
alias free='free -h'
alias ip='ip -c'
alias ss='ss -tuln'

# Busqueda
alias psg='ps aux | grep'
alias ports='ss -tulanp'
alias tree='tree -C'

# Git
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
alias gl='git log --oneline --graph'
alias gd='git diff'
alias gco='git checkout'
alias gb='git branch'
alias gst='git stash'
```

> Los alias que anaden opciones a comandos basicos (`rm -i`, `cp -i`) son una red de seguridad, pero crean dependencia: si se ejecuta el comando en un sistema sin esos alias, el comportamiento cambia. Es recomendable usarlos en entornos personales pero no abusar en scripts ni en sistemas compartidos.

## Alias temporales y por proyecto

Los alias pueden definirse para una sola sesion o activarse condicionalmente segun el directorio de trabajo:

```bash
# Alias temporal (solo dura lo que la sesion)
alias trabajo='cd /ruta/del/proyecto && source venv/bin/activate'

# Alias condicional (solo si existe cierto directorio)
if [ -d "$HOME/proyecto" ]; then
    alias p='cd $HOME/proyecto && npm run dev'
fi
```

Para alias especificos de un proyecto, se puede usar `direnv` (herramienta que carga configuracion al entrar en un directorio) o un archivo local que se ejecute manualmente:

```bash
# .aliases_locales (en el directorio del proyecto)
alias start='docker-compose up -d && npm run dev'
alias logs='docker-compose logs -f'

# Cargar manualmente
source .aliases_locales
```

