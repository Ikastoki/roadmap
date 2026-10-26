# Ciclo basico de trabajo en Git

El ciclo basico de trabajo en Git sigue un patron simple pero poderoso: preparar cambios, confirmarlos y consultar el historial. Comprender este ciclo es la base para todas las operaciones mas avanzadas de Git, desde la协作 con ramas hasta la resolucion de conflictos.

Cada paso del ciclo corresponde a una operacion especifica de Git que mueve datos entre las tres areas: directorio de trabajo, area de preparacion y repositorio.

## Paso 1: Preparar cambios

El primer paso es seleccionar que cambios quieres incluir en el proximo commit. Puedes agregar archivos completos o incluso partes especificas de un archivo.

```bash
# Agregar un archivo completo
git add archivo.py

# Agregar un directorio
git add src/

# Agregar todos los cambios modificados y nuevos
git add .

# Seleccion interactiva: elegir que partes de un archivo agregar
git add -p
```

`git add -p` (*patch*) es especialmente util: te permite revisar cada cambio (hunk) individualmente y decidir si incluirlo o no en el area de preparacion. Esto permite crear commits atomicos y limpios, separando distintos tipos de cambios en commits diferentes.

```bash
git add -p
# diff --git a/archivo.py b/archivo.py
# @@ -10,6 +10,8 @@ def funcion():
#      pass
#
# +    # Nuevo codigo
# +    resultado = calcular()
#
# Stage this hunk [y,n,q,a,d,/,e,?]? y
```

## Paso 2: Confirmar cambios (commit)

Un commit es una instantanea inmutable del estado del proyecto en un momento dado. Cada commit tiene un identificador unico (hash SHA-1/SHA-256), un autor, una marca temporal y un mensaje que describe los cambios.

```bash
# Commit con mensaje
git commit -m "Agregar funcion de calculo"

# Commit con mensaje multilinea
git commit -m "Agregar validacion de email

Implementar validacion basica de formato de email
en el formulario de registro. Usa expresion regular
para verificar la estructura del email."

# Commit que incluye archivos modificados (no nuevos)
git commit -am "Corregir bug en login"
```

**Buenas practicas en mensajes de commit:**

- El asunto debe ser corto (50 caracteres maximo) y estar en imperativo ("Agregar", no "Agregado").
- El cuerpo (opcional) explica el porque y el como, no el que.
- Separar cambios logicos en commits diferentes.

```bash
# Mal
git commit -m "fix"

# Bien
git commit -m "Corregir validacion de email en formulario de registro

La regex anterior no aceptaba subdominios. Se actualiza
para soportar emails como usuario@sub.dominio.com"
```

## Paso 3: Consultar el historial

Despues de crear commits, puedes revisar el historial para verificar que los cambios se registraron correctamente.

```bash
# Historial basico
git log --oneline
# a1b2c3d (HEAD -> main) Corregir validacion de email
# e4f5g6h Agregar funcion de calculo
# i7j8k9l Initial commit

# Historial con diferencias
git log -p

# Historial de un archivo especifico
git log --follow -- archivo.py

# Estadisticas por autor
git shortlog -sn
```

## Consulta de diferencias

`git diff` muestra las diferencias entre el estado actual y el ultimo commit, o entre diferentes areas de Git.

```bash
# Diferencias no preparadas (working tree vs staging)
git diff

# Diferencias preparadas (staging vs last commit)
git diff --staged

# Diferencias entre dos commits
git diff abc1234 def5678

# Diferencias de un archivo especifico
git diff HEAD -- archivo.py

# Resumen de cambios
git diff --stat
```

## Deshacer cambios

```bash
# Deshacer cambios en el directorio de trabajo (sin afectar staging)
git checkout -- archivo.py
git restore archivo.py

# Quitar un archivo del area de preparacion
git reset HEAD archivo.py
git restore --staged archivo.py

# Modificar el ultimo commit (mensaje o contenido)
git commit --amend -nuevo mensaje
```

> El ciclo basico de trabajo (add, commit, log) es la operacion que mas veces realizas en un dia de desarrollo con Git. Practicar este ciclo hasta que se vuelva automatico es la mejor inversion de tiempo para cualquier desarrollador.

