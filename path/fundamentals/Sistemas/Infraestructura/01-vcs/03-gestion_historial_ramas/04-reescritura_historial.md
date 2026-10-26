# Reescritura de historial

La reescritura de historial es una de las funcionalidades mas poderosas y peligrosas de Git. Permite modificar commits existentes, reorganizar el orden de los cambios, combinar multiples commits en uno solo y corregir errores en commits antiguos. Sin embargo, reescribir historial que ya ha sido compartido puede causar problemas graves a otros colaboradores.

La regla de oro es inquebrantable: **nunca reescribas historial que ya ha sido enviado a un repositorio compartido**. La reescritura de historial es segura solo para commits locales que aun no han sido fusionados con otros.

## Modificar el ultimo commit (amend)

`git commit --amend` reemplaza el ultimo commit con uno nuevo. Puede usarse para corregir el mensaje o para agregar archivos olvidados.

```bash
# Corregir el mensaje del ultimo commit
git commit --amend -m "Nuevo mensaje descriptivo"

# Agregar archivos olvidados al ultimo commit
git add archivo_olvidado.py
git commit --amend --no-edit
```

## Rebase interactivo

`git rebase -i` (*interactive rebase*) es la herramienta mas potente para reescribir historial. Permite reordenar, combinar, editar y eliminar commits de forma interactiva.

```bash
# Reescribir los ultimos 5 commits
git rebase -i HEAD~5
```

El editor muestra una lista de commits con acciones por defecto:

```
pick abc1234 Primer commit
pick def5678 Segundo commit
pick ghi9012 Tercer commit
pick jkl3456 Cuarto commit
pick mno7890 Quinto commit
```

**Acciones disponibles:**

- **pick:** mantener el commit sin cambios.
- **reword:** modificar el mensaje del commit.
- **edit:** pausar el rebase para modificar el contenido del commit.
- **squash:** combinar este commit con el anterior.
- **fixup:** combinar con el anterior descartando este mensaje.
- **drop:** eliminar el commit.

```bash
# Ejemplo: combinar los dos primeros commits
pick abc1234 Agregar estructura basica
squash def5678 Corregir typo en README

# Esto creara un solo commit con ambos cambios
```

## Rebase para mantener historial limpio

`git rebase` sin `-i` actualiza una rama con los cambios mas recientes de otra rama, reescribiendo los commits de la rama actual para que parezca que se crearon sobre la base mas reciente.

```bash
# Actualizar feature con los cambios de main
git checkout feature
git rebase main

# Equivalente a:
# 1. Quitar los commits de feature de donde estan
# 2. Aplicar los commits de main
# 3. Reaplicar los commits de feature encima
```

**Rebase vs merge:**

| Aspecto | Rebase | Merge |
|---|---|---|
| Historial | Lineal, limpio | Grafo con commits de fusion |
| Commits de fusion | No crea | Crea un commit M |
| Seguridad | Reescribe historial (no para ramas compartidas) | No reescribe historial |
| Ideal para | Limpiar historial antes de fusionar | Fusionar ramas en produccion |

## Reglas de seguridad

1. **Nunca reescribas historial compartido:** si otros trabajan sobre la misma rama, reescribir sus commits causara duplicados, conflictos y perdida de trabajo.
2. **Usa rebase para ramas locales:** antes de fusionar una feature con main, haz rebase para limpiar el historial.
3. **Usa merge para integrar ramas publicas:** la rama main nunca debe ser reescrita.
4. **Si reescribes por error, usa reflog:** `git reflog` permite recuperar commits "perdidos" por reescritura.

> La reescritura de historial es una herramienta de mantenimiento, no de diario. Usada correctamente, produce historiales limpios y navegables. Usada incorrectamente, puede causar confusion y perdida de trabajo. La regla de oro (no reescribir historial compartido) debe ser inquebrantable.

