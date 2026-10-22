<h1 align="center">GIT</h1>

<h2>📑 Contenido</h2>

- [¿Que es GIT ?](#que-es-git-)
- [Qué hace Git](#qué-hace-git)
- [Como trabaja Git](#como-trabaja-git)
- [Instalar](#instalar)
- [Version](#version)
- [Configurar Git // Mirar configuración en local(solo para un repo)](#configurar-git--mirar-configuración-en-localsolo-para-un-repo)

## ¿Que es GIT ?

Git es un software de control de versiones diseñado por Linus Torvalds, pensando en la eficiencia, la confiabilidad y compatibilidad del mantenimiento de versiones de aplicaciones cuando estas tienen un gran número de archivos de código fuente.

Se utiliza para:

- Seguimiento de cambios en el código
- Seguimiento de quién realizó cambios
- Colaboración de codificación

## Qué hace Git

- Administrar proyectos con repositorios(**Repositories**).
- Clonar(**Clone**) un proyecto para trabajar en una copia local.
- Controlar y realizar un seguimiento de los cambios con, en modo provisional.(**Staging**) y Confirmación(**Committing**).
- Bifurcar(**Branch**) y combinar(**Merge**) para permitir el trabajo en Diferentes partes y versiones de un proyecto.
- Extraer(**Pull**) la versión más reciente del proyecto en una copia local.
- Insertar(**Push**) actualizaciones locales en el proyecto principal.

## Como trabaja Git

- Primero inicializa Git en una carpeta, convirtiéndolo en un repositorio git.
- Git ahora crea una carpeta oculta para realizar un seguimiento de los cambios en esa carpeta.
- Cuando un archivo se cambia, agrega o elimina, se considera modificado.
- Seleccione los archivos modificados que desea organizar.
- Los archivos preconfigurados están confirmados(committing), que solicita a Git que almacene una instantánea permanente de los archivos.
- Git le permite ver el historial completo de cada confirmación.
- Puede volver a cualquier confirmación anterior.
- Git no almacena una copia separada de cada archivo en cada confirmación, pero ¡Realiza un seguimiento de los cambios realizados en cada confirmación!

## Instalar

Se puede instalar Git de la web oficial: [Git](https://www.git-scm.com/)

## Version

Revisar la version de Git que tenemos instalada:

```bash
git --version
git -v
```

## Configurar Git // Mirar configuración en local(solo para un repo)

Para empezar a usar GIT es necesario establecer una configuración previa.
Se necesita configurar el e-mail y el nombre de usuario para poder utilizar GIT.

```bash
### Asociar un nombre de Usuario y un Correo
git config --global user.name "MiNombre"
git config --global user.email "correo@dominio.com"
```

También se puede cambiar el editor por defecto:

```bash
### Elegir editor
git config --global core.editor tuEditor

### Para VSCode
git config --global core.editor "code --wait"
```

Comprobar las configuraciones:

```bash
### Comprobando tu Configuración
git config --list

### Comprobar usuario, correo y editor
git config user.name
git config user.email
git config code.editor
```


