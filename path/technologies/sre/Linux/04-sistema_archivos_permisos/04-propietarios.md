# Propiedad de archivos y directorios

Cada archivo y directorio en Linux pertenece a un usuario y a un grupo. Esta doble vinculacion es la base del modelo de permisos discrecional: el propietario puede decidir que nivel de acceso concede al grupo y al resto de usuarios. La administracion de la propiedad es una tarea habitual en sistemas multiusuario, especialmente cuando se transfieren archivos entre cuentas o se configuran directorios compartidos.

## El comando chown

`chown` (change owner) modifica el usuario propietario de un archivo o directorio. Solo el superusuario puede cambiar el propietario de un archivo.

```bash
chown usuario archivo.txt
chown usuario:grupo archivo.txt   # Cambia usuario y grupo simultaneamente
chown :grupo archivo.txt          # Cambia solo el grupo (equivalente a chgrp)
chown usuario: archivo.txt        # Cambia usuario y grupo al grupo primario del usuario
```

Opciones habituales:

```bash
chown -R usuario:grupo /directorio   # Recursivo: aplica a todo el arbol
chown --reference=modelo.txt destino.txt  # Copia la propiedad de otro archivo
chown -R --from=antiguo_usuario nuevo_usuario /home  # Cambia solo si el propietario actual coincide
```

La opcion recursiva `-R` es indispensable para directorios: cambia la propiedad del directorio y de todos los archivos y subdirectorios que contiene.

```bash
chown -R unai:desarrolladores /var/www/proyecto
```

> Un usuario normal no puede transferir la propiedad de un archivo a otro usuario, incluso si es el propietario del archivo. Solo root puede ejecutar `chown` para cambiar el propietario. Esta restriccion evita que un usuario pueda eludir cuotas de disco o realizar ataques de suplantacion.

### Casos de uso de chown

- Transferencia de archivos entre usuarios:

```bash
sudo chown -R nuevo_usuario /home/antiguo_usuario/documentos
```

- Correccion de propiedad tras restaurar backups (los archivos restaurados suelen pertenecer a root):

```bash
sudo chown -R unai:unai /home/unai/restauracion
```

- Configuracion de servidores web (el usuario `www-data` debe ser propietario de los archivos servidos):

```bash
sudo chown -R www-data:www-data /var/www/html
```

## El comando chgrp

`chgrp` (change group) modifica exclusivamente el grupo propietario de un archivo o directorio. A diferencia de `chown`, cualquier usuario puede cambiar el grupo de sus propios archivos, pero solo a grupos de los que sea miembro.

```bash
chgrp desarrolladores proyecto/    # Cambia el grupo a desarrolladores
chgrp -R desarrolladores /compartido/  # Recursivo
chgrp --reference=modelo.txt destino.txt  # Copia el grupo de otro archivo
```

### Diferencia entre chown :grupo y chgrp

Ambos comandos logran el mismo resultado para el grupo:

```bash
chown :desarrolladores archivo.txt
chgrp desarrolladores archivo.txt
```

`chgrp` es semanticamente mas claro cuando solo se desea modificar el grupo, y ademas permite que usuarios sin permisos de superusuario cambien el grupo de sus archivos a otros grupos de los que formen parte.

## Visualizacion de propietarios

El comando `ls -l` muestra el usuario y grupo propietario en las columnas tercera y cuarta:

```bash
ls -l
-rw-r--r-- 1 unai desarrolladores 2048 jul  1 12:00 proyecto.py
drwxr-xr-x 2 root root            4096 jul  1 12:00 scripts
```

En el ejemplo, `proyecto.py` pertenece al usuario `unai` y al grupo `desarrolladores`. El directorio `scripts` pertenece a `root:root`.

Para ver solo la informacion de propiedad de un archivo:

```bash
stat -c "%U %G" archivo.txt
stat archivo.txt   # Informacion detallada
```

## Propiedad efectiva vs. propiedad real

Un proceso en Linux tiene asociados varios identificadores:
- **UID real**: el usuario que ejecuto el proceso.
- **UID efectivo**: el usuario cuyos permisos se utilizan para las comprobaciones de acceso (puede diferir si el binario tiene SUID).
- **UID guardado**: permite intercambiar entre UID real y efectivo sin perder privilegios.

Cuando un binario con SUID se ejecuta, el UID efectivo pasa a ser el del propietario del archivo, aunque el UID real siga siendo el del usuario que lo ejecuto. Esto es relevante al cambiar la propiedad de archivos mediante programas SUID: el programa actua con los permisos del propietario del binario, no del usuario que lo invoca.

```bash
# Ejemplo de un script SUID mal disenado (no hacer en produccion)
sudo chown root:root script.sh
sudo chmod u+s script.sh
# Cualquier usuario que ejecute script.sh lo hara como root
```

## Practicas recomendadas

- Asignar la propiedad al usuario minimo necesario: un servidor web no necesita ser root para servir contenido; `www-data` es suficiente.
- Usar grupos para delegar administracion: si varios usuarios administran un directorio, crear un grupo (ej. `admins_web`) y asignarselo como grupo propietario evita tener que usar root para cada modificacion.
- Verificar la propiedad tras operaciones de copia o restauracion: `cp -p` preserva la propiedad original, pero `cp` sin opciones la asigna al usuario que ejecuta la copia.
- En entornos con `sudo`, restringir el uso de `chown` a comandos especificos en lugar de permitir `sudo chown` generico:

```bash
usuario ALL=(root) /usr/bin/chown
```

