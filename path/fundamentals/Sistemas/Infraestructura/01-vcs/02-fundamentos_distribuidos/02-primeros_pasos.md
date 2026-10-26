# Primeros pasos en un repositorio

Los primeros pasos para trabajar con Git implican inicializar un repositorio, conocer el estado actual de los archivos y entender los comandos basicos para interactuar con el sistema. Estas operaciones fundamentales son la base sobre la que se construyen todas las demas funcionalidades de Git.

Antes de crear un repositorio, es recomendable crear un directorio para el proyecto y situarse en el antes de ejecutar los comandos de inicializacion.

## Inicializacion de un repositorio

`git init` convierte un directorio existente en un repositorio Git, creando el directorio oculto `.git/` que contiene toda la base de datos del historial.

```bash
# Crear directorio y entrar en el
mkdir mi-proyecto
cd mi-proyecto

# Inicializar repositorio
git init
# Initialized empty Git repository in /home/usuario/mi-proyecto/.git/
```

Despues de `git init`, el directorio es un repositorio Git vacio: no hay commits, no hay ramas (excepto la rama por defecto `main`), y todos los archivos son untracked.

```bash
# Clonar un repositorio existente
git clone https://github.com/usuario/proyecto.git
git clone https://github.com/usuario/proyecto.git mi-carpeta
```

## Consulta del estado

`git status` es el comando mas frecuente en el dia a dia de Git. Muestra el estado actual del directorio de trabajo y del area de preparacion: que archivos estan modificados, que archivos estan preparados y que archivos no estan rastreados.

```bash
git status
# On branch main
#
# No commits yet
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#       archivo.txt
#
nothing added to commit but untracked files present
```

Despues de agregar archivos y hacer commit:

```bash
git status
# On branch main
# Changes to be committed:
#   (use "git restore --staged <file>..." to unstage)
#       new file:   archivo.txt
#
#nothing to commit, working tree clean
```

`git status` en modo corto (`-s`) proporciona una vista mas concisa:

```bash
git status -s
# M  archivo_modificado.txt
# A  archivo_nuevo.txt
# ?? archivo_sin_rastrear.txt
```

Los indicadores de la izquierda representan el area de preparacion; los de la derecha, el directorio de trabajo. `M` = modificado, `A` = agregado, `??` = sin rastrear.

## Consulta del historial

`git log` muestra el historial de commits del repositorio, del mas reciente al mas antiguo.

```bash
# Historial basico
git log

# Formato compacto
git log --oneline
# a1b2c3d Agregar login de usuario
# e4f5g6h Corregir bug en formulario
# i7j8k9l Configurar base de datos

# Historial con grafico de ramas
git log --oneline --graph --all

# Buscar commits por mensaje
git log --grep="login"

# Ver cambios de un commit especifico
git show abc1234
```

## Comandos esenciales

```bash
# Agregar archivos al area de preparacion
git add archivo.txt           # Un archivo
git add directorio/           # Un directorio completo
git add .                     # Todos los archivos modificados/nuevos
git add -p                    # Seleccion interactiva de partes de archivos

# Confirmar cambios
git commit -m "Mensaje descriptivo"
git commit -am "Mensaje"      # Agrega todos los modificados y confirma

# Ver diferencias
git diff                      # Diferencias no preparadas
git diff --staged             # Diferencias preparadas
git diff HEAD                 # Todas las diferencias vs ultimo commit
```

> `git status` y `git log` son tus mejores amigos en Git. Consultar el estado antes de cada operacion y revisar el historial periodicamente te ayudara a evitar errores y a mantener un proyecto ordenado.

