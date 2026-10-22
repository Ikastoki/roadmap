<h1 align="center">Repositorios Remotos</h1>

<h2>📑 Contenido</h2>

- [¿Que son los repositorios remotos?](#que-son-los-repositorios-remotos)
- [Operaciones básicas mas comunes](#operaciones-básicas-mas-comunes)
  - [Clonar](#clonar)
  - [Agregar](#agregar)
  - [Listar](#listar)
  - [Descargar cambios](#descargar-cambios)
  - [Enviar cambios](#enviar-cambios)
  - [Eliminar](#eliminar)
  - [Cambiar URL](#cambiar-url)

## ¿Que son los repositorios remotos?

Un repositorio remoto en Git es una versión de tu repositorio Git que se encuentra alojada en un servidor en línea o en otro lugar fuera de tu sistema local. Los repositorios remotos son esenciales para colaborar en proyectos con otras personas y para realizar copias de seguridad de tu código.

## Operaciones básicas mas comunes

Si utilizas un alojamiento de control de versiones como GitHub, te proporcionara una guía con los comandos que debes seguir al alojar un repositorio local.

### Clonar

Para comenzar a trabajar en un proyecto, puedes clonar un repositorio remoto en tu máquina local usando el comando git clone. Esto crea una copia local completa del repositorio remoto, que puedes modificar y sincronizar con el repositorio remoto cuando sea necesario.

```bash
git clone <URL_del_repositorio_remoto>
```

### Agregar

Puedes agregar un repositorio remoto a tu repositorio local utilizando el comando git remote add. Esto es útil cuando quieres colaborar con otros o seguir un repositorio en línea.

```bash
git remote add <nombre_remoto> <URL_del_repositorio_remoto>
```

### Listar

Puedes ver una lista de los repositorios remotos asociados con tu repositorio local usando el comando git remote.

```bash
git remote -v
```

### Descargar cambios

Puedes obtener los cambios más recientes desde un repositorio remoto usando el comando git pull. Esto sincroniza tus cambios locales con los cambios realizados en el repositorio remoto.

```bash
git pull <nombre_remoto> <rama_remota>
```

### Enviar cambios

Para enviar tus cambios locales al repositorio remoto, puedes usar el comando git push.

```bash
git push <nombre_remoto> <rama_local>
```

```bash
git push -u origin master
```

El uso de -u establece una relación de seguimiento, lo que significa que en futuros comandos git push y git pull, Git sabrá a qué rama remota hacer referencia de forma predeterminada.

### Eliminar

Si ya no deseas mantener una conexión con un repositorio remoto, puedes eliminarlo usando el comando git remote remove.

```bash
git remote remove <nombre_remoto>
```

### Cambiar URL

Si necesitas cambiar la URL de un repositorio remoto existente, puedes usar el comando git remote set-url.

```bash
git remote set-url <nombre_remoto> <nueva_URL>
```


