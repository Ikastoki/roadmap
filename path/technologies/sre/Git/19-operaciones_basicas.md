<h1 align="center">Operaciones Básicas</h1>

<h2>📑 Contenido</h2>

- [Operaciones básicas](#operaciones-básicas)
  - [Iniciar repositorio](#iniciar-repositorio)
  - [Clonar repositorio](#clonar-repositorio)
  - [Agregar archivos al área de preparación](#agregar-archivos-al-área-de-preparación)
  - [Hacer commit](#hacer-commit)
  - [Estado del repositorio](#estado-del-repositorio)
  - [Historial de commits](#historial-de-commits)
  - [Nueva rama](#nueva-rama)
  - [Cambiar de rama](#cambiar-de-rama)
  - [Fusionar ramas](#fusionar-ramas)
  - [Descargar cambios remotos](#descargar-cambios-remotos)
  - [Subir cambios remotos](#subir-cambios-remotos)
  - [Crear etiquetas](#crear-etiquetas)
- [Subir un repositorio local a Github](#subir-un-repositorio-local-a-github)

## Operaciones básicas

### Iniciar repositorio

```bash
git init
```

### Clonar repositorio

```bash
git clone <URL_del_repositorio_remoto>
```

### Agregar archivos al área de preparación

```bash
### Agregar un archivo específico
git add <nombre_archivo>

### Agregar todos los archivos modificados
git add .
```

### Hacer commit

```bash
git commit -m "Mensaje del commit"
```

### Estado del repositorio

```bash
git status
```

### Historial de commits

```bash
git log
```

### Nueva rama

```bash
git branch <nombre_de_rama>
```

### Cambiar de rama

```bash
git checkout <nombre_de_rama>
```

### Fusionar ramas

```bash
git merge <nombre_de_rama>
```

### Descargar cambios remotos

```bash
git pull origin <nombre_de_rama>
```

### Subir cambios remotos

```bash
git push origin <nombre_de_rama>
```

### Crear etiquetas

```bash
### Crear una etiqueta ligera
git tag <nombre_etiqueta>

### Crear una etiqueta anotada
git tag -a <nombre_etiqueta> -m "Mensaje"
```

## Subir un repositorio local a Github

```bash
### Inicializar repositorio
git init

### Agregar todos los archivos
git add .

### Insertar commit
git commit -m "first commit"

### Conectar con el repositorio remoto
git remote add origin https://github.com/NOMBRE_USUARIO/NOMBRE_PROYECTO.git

### Subir repositorio
git push -u origin master
```


