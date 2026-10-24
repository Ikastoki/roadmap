# Tu primer script

Un script de shell no es mas que un archivo de texto que contiene una secuencia de comandos que el shell ejecuta secuencialmente. La potencia del shell scripting reside en que cualquier comando que se pueda escribir en la terminal se puede incluir en un script, y cualquier construccion de programacion (variables, condicionales, bucles) se puede combinar con esos comandos para automatizar tareas de cualquier complejidad.

## Creacion del archivo

El primer paso es crear un archivo de texto con un nombre significativo. No se requiere una extension especifica, aunque por convencion los scripts Bash usan `.sh`.

```bash
touch backup.sh
```

## El shebang

La primera linea de un script contiene el **shebang** (`#!`), una secuencia magica de dos caracteres que indica al sistema que interprete debe utilizar para ejecutar el archivo. Cuando el kernel detecta que un archivo ejecutable comienza con `#!`, lee el resto de la linea como la ruta al interprete y ejecuta el script pasandoselo a ese interprete como argumento.

```bash
#!/bin/bash
```

Sin el shebang, el sistema utilizara el shell por defecto del usuario, que podria ser Zsh, Fish u otro, y el script podria fallar si usa sintaxis especifica de Bash. Siempre se debe especificar el interprete, incluso en scripts simples.

> `#!/bin/bash` especifica Bash en `/bin/bash`. Si se quiere un script POSIX portable se usa `#!/bin/sh`. En sistemas modernos `/bin/sh` suele ser un enlace a Dash, que es mas rapido pero carece de caracteristicas de Bash como arrays, `(( ))` o `[[ ]]`.

## Permisos de ejecucion

Un script no puede ejecutarse hasta que tenga permiso de ejecucion. `chmod` concede este permiso:

```bash
chmod +x backup.sh
```

A partir de ese momento se puede ejecutar con `./backup.sh` (ruta relativa) o con la ruta absoluta. Si el directorio del script esta en el `PATH`, bastaria con escribir `backup.sh`.

## Estructura basica

Un script bien estructurado incluye, ademas del shebang, una documentacion minima (usando comentarios con `#`), variables al inicio, funciones si es necesario y un flujo claro:

```bash
#!/bin/bash

# Script: backup.sh
# Descripcion: Realiza una copia de seguridad del directorio home
# Uso: ./backup.sh

ORIGEN="$HOME/documentos"
DESTINO="$HOME/backups"
FECHA=$(date +%Y%m%d_%H%M%S)
ARCHIVO="backup_$FECHA.tar.gz"

mkdir -p "$DESTINO"
tar -czf "$DESTINO/$ARCHIVO" "$ORIGEN"
echo "Backup creado: $DESTINO/$ARCHIVO"
```

Para depurar un script se puede ejecutar con `bash -x script.sh`, que muestra cada comando antes de ejecutarlo. Es la herramienta mas util para entender que esta haciendo realmente un script cuando algo no funciona como se espera.

```bash
bash -x backup.sh
+ ORIGEN=/home/usuario/documentos
+ DESTINO=/home/usuario/backups
+ FECHA=20260728_120000
+ ARCHIVO=backup_20260728_120000.tar.gz
+ mkdir -p /home/usuario/backups
+ tar -czf /home/usuario/backups/backup_20260728_120000.tar.gz /home/usuario/documentos
```

> Ejecutar un script con `bash script.sh` no requiere permiso de ejecucion, solo lectura. El permiso de ejecucion es necesario unicamente cuando se invoca el script directamente mediante su ruta (`./script.sh`).

