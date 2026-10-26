# Visualizacion y comparacion de cambios

La capacidad de ver exactamente que ha cambiado, cuando y donde es una de las mayores ventajas de Git. Las herramientas de visualizacion y comparacion permiten entender la evolucion del proyecto, identificar que cambios pertenecen a que commit y diagnosticar problemas cuando algo no funciona como se esperaba.

Git proporciona multiples formas de comparar el estado del proyecto: entre el directorio de trabajo y el area de preparacion, entre el area de preparacion y el ultimo commit, o entre dos commits arbitrarios.

## Comparar cambios no preparados

`git diff` sin argumentos muestra las diferencias entre el directorio de trabajo y el area de preparacion (o el ultimo commit si no hay nada preparado).

```bash
# Diferencias en el working tree vs staging
git diff

# Salida tipica
# diff --git a/archivo.py b/archivo.py
# index abc1234..def5678 100644
# --- a/archivo.py
# +++ b/archivo.py
# @@ -10,6 +10,8 @@ def funcion():
#      pass
#
# +    # Nuevo codigo agregado
# +    resultado = calcular()
#
#      return resultado
```

Los archivos modificados se muestran con `+` (lineas agregadas) y `-` (lineas eliminadas).

## Comparar cambios preparados

`git diff --staged` (o `--cached`) muestra las diferencias entre el area de preparacion y el ultimo commit. Estos son los cambios que se incluiran en el proximo commit.

```bash
git diff --staged
```

## Comparar con un commit especifico

```bash
# Diferencia con el ultimo commit
git diff HEAD

# Diferencia con un commit especifico
git diff abc1234

# Diferencia entre dos commits
git diff abc1234 def5678

# Diferencia de un archivo especifico
git diff HEAD -- src/main.py

# Resumen estadistico
git diff --stat
# src/main.py | 12 ++++++------
# src/utils.py | 3 +++
# 2 files changed, 9 insertions(+), 6 deletions(-)

# Diferencia con la rama principal
git diff main..mi-rama
```

## Herramientas graficas de comparacion

Para una visualizacion mas clara, se pueden utilizar herramientas graficas externas configuradas como dif de Git:

```bash
# Configurar vimdiff
git config --global diff.tool vimdiff

# Configurar VS Code
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'

# Abrir herramienta grafica
git difftool
```

## Visualizar el historial

```bash
# Historial compacto
git log --oneline --graph --all
# * a1b2c3d (HEAD -> main) Agregar login
# | * e4f5g6h (feature/auth) Implementar OAuth
# |/
# * i7j8k9l Initial commit

# Historial con archivos modificados
git log --stat

# Historial de un archivo
git log --oneline -- src/main.py

# Buscar commits por autor
git log --author="Juan"

# Buscar commits por mensaje
git log --grep="bug"

# Historial con diferencias completas
git log -p
```

## Visualizar un commit

```bash
# Ver detalles de un commit
git show abc1234

# Ver solo los archivos modificados
git show --stat abc1234

# Ver un archivo en un commit especifico
git show abc1234:src/main.py
```

> Las herramientas de comparacion de Git son tu lente de aumento para entender que ha cambiado en el proyecto. Usar `git diff` antes de cada commit te permite revisar tus cambios, detectar errores y garantizar que el historial refleje exactamente lo que intendias hacer.

