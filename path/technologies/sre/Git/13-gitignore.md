<h1 align="center">Gitignore</h1>

<h2>📑 Contenido</h2>

- [¿Que es gitignore?](#que-es-gitignore)
- [Consideraciones](#consideraciones)
  - [Sintaxis](#sintaxis)
  - [Comentarios](#comentarios)
  - [Agregar al control de versiones](#agregar-al-control-de-versiones)
- [Ejemplos](#ejemplos)

## ¿Que es gitignore?

El archivo .gitignore es un archivo especial utilizado en repositorios Git para **especificar qué archivos y directorios deben ser ignorados y no incluidos en el control de versiones.** Esto es útil para excluir archivos generados automáticamente, archivos binarios, archivos de configuración local y otros tipos de archivos que no deben formar parte del historial de versiones del proyecto.

> [!NOTE]
>
> El archivo .gitignore es una herramienta esencial para mantener el repositorio limpio y evitar problemas con archivos no deseados en el historial de versiones.

Algunos archivos y directorios comunes que suelen ser excluidos en el archivo .gitignore:

- Archivos de configuración local: `.vscode/, .idea/, *.iml, *.vssettings, *.sublime-project`
- Archivos de compilación y ejecución: `*.exe, *.dll, *.so, *.class, *.jar, *.war, *.log`
- Directorios y archivos de dependencias: `node_modules/, vendor/, lib/`
- Archivos de compilación y configuración personalizados: `build/, .env, .DS_Store, .gitignore`
- Archivos de caché y temporales: `*.cache, *.tmp, *.bak.`

## Consideraciones

### Sintaxis

En el archivo .gitignore, puedes especificar patrones de archivos y directorios que deben ser ignorados. Cada patrón va en una línea separada. Puedes usar comodines (_), como _ para coincidir con todos los archivos en un directorio, o \*.log para coincidir con todos los archivos con la extensión .log. También puedes usar barras inclinadas / para especificar rutas relativas desde la raíz del repositorio.

### Comentarios

Puedes agregar comentarios en el archivo .gitignore utilizando el carácter #. Los comentarios se ignorarán y no afectarán las reglas de exclusión.

### Agregar al control de versiones

Aunque el archivo .gitignore no está destinado a ser compartido con otros usuarios del repositorio, es una buena práctica incluirlo en el control de versiones para que otros desarrolladores tengan acceso a las mismas reglas de exclusión. Esto ayuda a mantener una consistencia en el control de versiones.

## Ejemplos

```bash
# Ignorar todos los archivos .log en el directorio raíz:
*.log

# Ignorar un directorio completo llamado logs:
/logs/

# Ignorar todos los archivos .tmp en cualquier directorio:
**/*.tmp

# Ignorar archivos de configuración local, como config.json:
config.json
```


