<h1 align="center">Operaciones Básicas con Commits</h1>

<h2>📑 Contenido</h2>

- [Crear un Commit](#crear-un-commit)
- [Ver Historial de Commits](#ver-historial-de-commits)
- [Cambiar el Mensaje de un Commit (Amend)](#cambiar-el-mensaje-de-un-commit-amend)
- [Revertir un Commit](#revertir-un-commit)
- [Eliminar un Commit (Reset)](#eliminar-un-commit-reset)
- [Fusionar Commits](#fusionar-commits)
- [Eliminar un Commit (Rebase Interactivo)](#eliminar-un-commit-rebase-interactivo)

## Crear un Commit

Para crear un commit, primero debes realizar cambios en tus archivos. Luego, puedes usar el comando git commit seguido de la opción -m y un mensaje descriptivo para registrar los cambios. Por ejemplo:

```bash
git commit -m "Agregado nuevo módulo de autenticación"
```

## Ver Historial de Commits

Puedes ver el historial de commits de tu repositorio utilizando el comando git log. Esto te mostrará una lista de commits en orden cronológico inverso, con detalles como el autor, la fecha, el mensaje y el identificador único (hash).

```bash
git log
```

## Cambiar el Mensaje de un Commit (Amend)

Si necesitas cambiar el mensaje de tu commit más reciente o agregar cambios adicionales a ese commit, puedes usar el comando git commit --amend. Esto es útil para hacer correcciones rápidas o mejorar la claridad del mensaje de un commit.

```bash
git commit --amend -m "Mensaje revisado"
```

## Revertir un Commit

Si deseas deshacer los cambios realizados en un commit específico, puedes utilizar el comando git revert. Esto creará un nuevo commit que deshace los cambios del commit seleccionado.

```bash
git revert <hash-del-commit>
```

## Eliminar un Commit (Reset)

Puedes eliminar commits de una rama utilizando el comando git reset. Hay tres tipos de reset: soft, mixed y hard. Soft mantiene los cambios en tu área de preparación, mixed los elimina de tu área de preparación y hard los elimina por completo. Ten cuidado al usar reset, ya que puede reescribir la historia del proyecto.

```bash
### Ejemplo de reset suave (soft reset)
git reset --soft HEAD~1
```

## Fusionar Commits

Si tienes varios commits pequeños que deseas agrupar en uno solo para mantener una historia de versiones más limpia, puedes utilizar la opción --squash durante la fusión de ramas o commits con git merge o git pull. Esto te permite combinar varios commits en uno solo.

```bash
git merge --squash <rama-o-commit-a-fusionar>
```

## Eliminar un Commit (Rebase Interactivo)

El rebase interactivo te permite eliminar, reorganizar o editar commits en la historia de versiones. Esto se hace con el comando git rebase -i.

```bash
git rebase -i HEAD~3
```


