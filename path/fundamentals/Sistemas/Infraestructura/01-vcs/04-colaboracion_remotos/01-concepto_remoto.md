# Concepto de repositorio remoto

Un repositorio remoto es una version del proyecto alojada en un servidor o servicio de nube (GitHub, GitLab, Bitbucket, un servidor privado) que sirve como punto de intercambio entre colaboradores. Cada desarrollador clona el repositorio remoto, trabaja localmente y luego sincroniza sus cambios con el remoto para que otros puedan acceder a ellos.

Los repositorios remotos son el corazon de la协作 en Git: sin ellos, cada trabajador tendria su copia aislada del proyecto sin forma de compartir cambios de forma estructurada.

## Nombre de los remotos

Cuando clonas un repositorio, Git asigna automaticamente el nombre `origin` al remoto del que procede. Puedes tener multiples remotos con nombres diferentes.

```bash
# Ver remotos configurados
git remote -v
# origin  https://github.com/usuario/proyecto.git (fetch)
# origin  https://github.com/usuario/proyecto.git (push)

# Añadir un remoto
git remote add upstream https://github.com/original/proyecto.git

# Renombrar un remoto
git remote rename origin github

# Eliminar un remoto
git remote remove upstream
```

**Nombres convencionales:**

- **origin:** el repositorio principal del proyecto (de donde clonaste).
- **upstream:** el repositorio original del que hiciste fork.
- **backup:** un repositorio de respaldo.

## URLs de los repositorios

Git soporta multiples protocolos para conectarse a repositorios remotos:

```bash
# HTTPS (requiere credenciales por cada operacion de push)
git remote add origin https://github.com/usuario/proyecto.git

# SSH (usa claves publicas, no requiere credenciales interactivas)
git remote add origin git@github.com:usuario/proyecto.git

# Git protocolo nativo (rapido pero sin autenticacion)
git remote add origin git://github.com/usuario/proyecto.git
```

> Los repositorios remotos son la base de la协作 moderna. GitHub, GitLab y Bitbucket no solo almacenan codigo: proporcionan herramientas de revision de codigo, gestion de proyectos, CI/CD y seguridad que van mucho mas alla del VCS basico.

