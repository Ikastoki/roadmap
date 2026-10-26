# Trabajo con ramas

Las ramas son la herramienta que permite el desarrollo paralelo en Git. Crear, cambiar y fusionar ramas son operaciones basicas que debes dominar para trabajar de forma efectiva. Las ramas ligeras de Git hacen que estas operaciones sean instantaneas, fomentando un flujo de trabajo donde cada cambio se aislana en su propia rama.

## Listar, crear y cambiar ramas

```bash
# Listar ramas locales
git branch
# * main
#   feature/login
#   bugfix/header

# Listar ramas remotas
git branch -r
# origin/main
# origin/develop

# Listar todas las ramas (locales y remotas)
git branch -a

# Crear nueva rama (sin cambiar a ella)
git branch feature/dashboard

# Cambiar a una rama
git checkout feature/dashboard
git switch feature/dashboard

# Crear y cambiar a la vez
git checkout -b feature/api
git switch -c feature/api

# Renombrar rama
git branch -m nombre-viejo nombre-nuevo
```

## Fusion (merge)

La fusion integra los cambios de una rama en otra. Git ofrece dos tipos principales de fusion:

### Fast-forward (avance rapido)

Cuando la rama destino no ha avanzado desde que se creo la rama origen, Git simplemente mueve el puntero de la rama destino al commit mas reciente de la rama origen. No se crea un commit de fusion.

```
Antes:
main:    A ← B ← C
                  ↑
feature:         D ← E

Fusion fast-forward:
main:    A ← B ← C ← D ← E  (HEAD -> main, feature)
```

```bash
# Fusion con fast-forward
git checkout main
git merge feature
```

### 3-way merge (fusion de tres vías)

Cuando ambas ramas han avanzado desde el punto de divergencia, Git crea un commit de fusion que combina los cambios de ambas ramas.

```
Antes:
main:    A ← B ← C ← F
                  ↑
feature:         D ← E

Fusion de tres vias:
main:    A ← B ← C ← F ← M  (commit de fusion)
                  ↑         ↑
feature:         D ← E ────┘
```

```bash
# Fusion de tres vias
git checkout main
git merge feature
# Merge made by the 'ort' strategy.
#  src/main.py | 10 +++++++---
#  1 file changed, 7 insertions(+), 3 deletions(-)
```

**Fast-forward vs 3-way merge:**

| Aspecto | Fast-forward | 3-way merge |
|---|---|---|
| Historial | Lineal, limpio | Grafo con commit de fusion |
| Cuandos | La rama destino no ha avanzado | Ambas ramas han avanzado |
| Commit de fusion | No crea uno | Crea un commit de fusion (M) |
| Visualizacion | Historial lineal | Historial ramificado |

```bash
# Forzar fast-forward (rebase en lugar de merge)
git merge --ff-only feature

# Forzar 3-way merge (aunque sea posible fast-forward)
git merge --no-ff feature -m "Fusionar feature/login"
```

## Eliminar ramas

```bash
# Eliminar rama ya fusionada (seguro)
git branch -d feature/terminada

# Eliminar rama sin verificar (peligroso)
git branch -D feature/temporal

# Eliminar rama remota
git push origin --delete feature/antigua
```

## Resolucion de conflictos de fusion

Cuando dos ramas modifican la misma linea de un archivo, Git no puede decidir automaticamente que cambio conservar. El conflicto se marca en el archivo con indicadores especiales:

```
<<<<<<< HEAD
cambios de la rama actual
=======
cambios de la rama que se esta fusionando
>>>>>>> feature/nueva
```

Para resolver un conflicto:

1. Abrir el archivo y buscar los marcadores de conflicto.
2. Editar el archivo para conservar la logica correcta.
3. Eliminar los marcadores de conflicto.
4. Agregar el archivo y confirmar la fusion.

```bash
# Despues de fusionar y encontrar conflictos
git status   # Muestra archivos en conflicto

# Editar los archivos manualmente
vim archivo.py

# Marcar como resuelto
git add archivo.py

# Finalizar la fusion
git commit
```

> El merge es una de las operaciones mas criticas de Git. Un merge limpio mantiene el historial comprensible; un merge descuidado puede introducir bugs y crear un historial confuso. Siempre revisa los cambios antes de fusionar y resuelve los conflictos cuidadosamente.

