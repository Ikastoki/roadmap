# Flujo colaborativo basico

El flujo colaborativo en Git define como los miembros de un equipo trabajan juntos, sincronizan cambios y evitan interferencias. Un flujo bien definido reduce conflictos, mejora la calidad del codigo y hace que la协作 sea predecible y eficiente.

El flujo basico sigue un patron simple: trabajar en rama local, enviar cambios y que los colaboradores extraigan esos cambios.

## Flujo tipico paso a paso

**Paso 1: Obtener los ultimos cambios**

Antes de iniciar cualquier tarea, sincroniza tu repositorio local con el remoto para tener los cambios mas recientes.

```bash
git pull origin main
```

**Paso 2: Crear una rama para la tarea**

Nunca trabajes directamente en `main`. Crea una rama descriptiva para cada tarea.

```bash
git checkout -b feature/busqueda-usuarios
```

**Paso 3: Trabajar y hacer commits**

Implementa los cambios, haciendo commits periodicos con mensajes descriptivos.

```bash
git add .
git commit -m "Agregar endpoint de busqueda"
git commit -m "Implementar filtro por nombre"
```

**Paso 4: Enviar la rama al remoto**

```bash
git push -u origin feature/busqueda-usuarios
```

**Paso 5: Abrir una solicitud de fusion (PR/MR)**

En la plataforma (GitHub, GitLab), abre una Pull Request o Merge Request que compare tu rama con `main`. Solicita revision a un colega.

**Paso 6: Revision y fusion**

Despues de la aprobacion, la rama se fusiona con `main` desde la interfaz de la plataforma.

**Paso 7: Limpiar**

```bash
git checkout main
git pull origin main
git branch -d feature/busqueda-usuarios
git push origin --delete feature/busqueda-usuarios
```

## Sincronizar antes de cada tarea

La causa mas comun de conflictos de fusion es no sincronizar antes de trabajar. Si dos personas trabajan sobre la misma base obsoleta, al fusionar ambos tendran cambios que chocan.

```bash
# Siempre antes de una nueva tarea:
git checkout main
git pull origin main
git checkout -b mi-nueva-tarea
```

## Evitar el force push en ramas compartidas

`git push --force` sobrescribe el historial remoto con tu historial local. Si otra persona trabajo sobre la misma rama, sus commits se pierden silenciosamente.

```bash
# PELIGROSO: sobrescribe historial compartido
git push --force origin main

# Mas seguro: solo avanza, nunca retrocede
git push --force-with-lease origin main
```

**Regla de oro:** nunca uses `--force` en ramas compartidas. Si necesitas reescribir historial, hazlo en ramas locales antes de compartir.

## Flujo con forks (proyectos de terceros)

Para contribuir a proyectos donde no tienes acceso de escritura:

1. Hacer fork del repositorio en tu cuenta.
2. Clonar tu fork.
3. Agregar el repositorio original como `upstream`.
4. Crear una rama para tu contribucion.
5. Hacer commits y push a tu fork.
6. Abrir PR desde tu fork al repositorio original.

```bash
git clone git@github.com:tuproject/proyecto.git
cd proyecto
git remote add upstream git@github.com:original/proyecto.git

git checkout -b fix/bug-login
# ... trabajar ...
git push origin fix/bug-login
# Abrir PR desde GitHub
```

> El flujo colaborativo es mas importante que la herramienta. Un equipo que sigue un flujo claro (pull antes de push, ramas por tarea, PR para revision) trabaja mas rapido y con menos errores que un equipo con herramientas potentes pero sin proceso.

