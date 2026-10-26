# Permisos y propiedad en el sistema de archivos

Los permisos y la propiedad son los mecanismos de control de acceso que determinan quién puede leer, escribir o ejecutar archivos y directorios en un sistema de archivos Unix/Linux. Este sistema de control de acceso discrecional (*DAC*) es la primera línea de defensa de la seguridad en un sistema multiusuario.

Cada archivo y directorio tiene un propietario, un grupo y una cadena de permisos que define los niveles de acceso para el propietario, los miembros del grupo y el resto de usuarios del sistema. Comprender y gestionar correctamente los permisos es esencial para mantener la seguridad y la funcionalidad del sistema.

## Propietario y grupo

Todo archivo en un sistema Unix/Linux tiene asignado un **propietario** (un usuario) y un **grupo**. El propietario es quien creó el archivo o a quien se le ha asignado con `chown`. El grupo determina qué usuarios no propietarios tienen permisos de acceso al archivo según las reglas de permisos de grupo.

```bash
# Cambiar propietario de un archivo
chown usuario:grupo archivo.txt

# Cambiar propietario recursivamente
chown -r usuario:grupo directorio/

# Cambiar solo el grupo
chgrp grupo archivo.txt
```

La combinación de propietario, grupo y permisos permite crear políticas de acceso flexibles: un archivo puede ser de lectura para todos los usuarios del grupo `desarrollo` pero solo modificable por su propietario.

## Permisos de lectura, escritura y ejecución

Los permisos se representan como una cadena de 10 caracteres que indica:

- **Posición 1:** tipo de archivo (`-` archivo regular, `d` directorio, `l` enlace simbólico, `c` dispositivo de caracteres, `b` dispositivo de bloques).
- **Posiciones 2-4:** permisos del propietario (rwx).
- **Posiciones 5-7:** permisos del grupo (rwx).
- **Posiciones 8-10:** permisos de los demás usuarios (rwx).

Donde **r** = lectura, **w** = escritura, **x** = ejecución, y `-` = permiso no concedido.

```bash
ls -la archivo.txt
# -rw-r--r-- 1 usuario grupo 1024 jul 25 10:00 archivo.txt
#  │││  │││  │
#  │││  │││  └─ otros: solo lectura
#  │││  │└───── grupo: solo lectura
#  │││  └────── propietario: lectura y escritura
#  ││└───────── tipo: archivo regular
```

Para archivos, los permisos significan:

- **r:** permite leer el contenido del archivo.
- **w:** permite modificar o eliminar el archivo.
- **x:** permite ejecutar el archivo (si es un programa o script).

Para directorios, los permisos tienen un significado diferente:

- **r:** permite listar el contenido del directorio (con `ls`).
- **w:** permite crear, eliminar o renombrar archivos dentro del directorio.
- **x:** permite acceder al directorio (con `cd`) y acceder a metadatos de archivos dentro de él.

> Un directorio sin permiso de ejecución (x) no puede ser accedido, aunque tenga permiso de lectura. Este es un error comun: `ls -l` puede mostrar los nombres de los archivos pero no sus detalles (tamano, permisos) si el directorio no tiene permiso de ejecucion.

## Cambio de permisos con chmod

`chmod` modifica los permisos de un archivo o directorio. Acepta notacion numerica (octal) o simbolica:

```bash
# Notacion octal (cada digito suma r=4, w=2, x=1)
chmod 755 script.sh    # rwxr-xr-x (propietario: todo, grupo y otros: lectura y ejecucion)
chmod 644 archivo.txt  # rw-r--r-- (propietario: lectura/escritura, otros: solo lectura)
chmod 700 directorio/  # rwx------ (solo propietario)

# Notacion simbolica
chmod u+x script.sh    # anadir ejecucion al propietario
chmod g-w archivo.txt  # quitar escritura al grupo
chmod o+r archivo.txt  # anadir lectura a otros
chmod a+r archivo.txt  # anadir lectura a todos (propietario, grupo, otros)
```

## Bits especiales

Ademas de los permisos basicos, existen bits especiales que modifican el comportamiento de los archivos y directorios:

**Bit setuid (4000):** cuando se establece en un ejecutable, el programa se ejecuta con los privilegios del propietario del archivo (no del usuario que lo ejecuta). Ejemplo: `/usr/bin/passwd` tiene setuid para que cualquier usuario pueda cambiar su contrasena, aunque el archivo `/etc/shadow` sea de lectura exclusiva de root.

```bash
chmod u+s programa
# -rwsr-xr-x  (la 's' indica setuid)
```

**Bit setgid (2000):** en un ejecutable, el programa se ejecuta con los privilegios del grupo. En un directorio, los archivos creados dentro de heredan el grupo del directorio en lugar del grupo primario del usuario. Util para directorios compartidos en equipo.

```bash
chmod g+s directorio/
# drwxrwsr-x  (la 's' indica setgid)
```

**Sticky bit (1000):** en un directorio, impide que los usuarios eliminen archivos que no son suyos, aunque tengan permiso de escritura en el directorio. El ejemplo clasico es `/tmp`, donde cualquier usuario puede crear archivos pero no puede eliminar los de otros usuarios.

```bash
chmod +t directorio/
# drwxrwxrwt  (la 't' indica sticky bit)
```

## Umask

La `umask` define los permisos por defecto que se conceden automaticamente al crear archivos y directorios. Resta los permisos indicados de los permisos maximos del sistema (666 para archivos, 777 para directorios).

```bash
# Ver la umask actual
umask

# Establecer una umask: restringe escritura para grupo y otros
umask 022
# Archivos: 666 - 022 = 644 (rw-r--r--)
# Directorios: 777 - 022 = 755 (rwxr-xr-x)
```

Una umask de `027` es comun en entornos de servidores: archivos 640 (solo propietario y grupo leen) y directorios 750 (propietario accede, grupo lee, otros nada).

> Los permisos Unix son efectivos pero limitados: no permiten auditoria de quién accedió a un archivo, ni control granular sobre operaciones especificas (como permitir escritura parcial). Para requisitos de seguridad avanzados, se utilizan mecanismos complementarios como SELinux, AppArmor o ACLs (*Access Control Lists*), que permiten definir permisos por usuario individual, no solo por propietario y grupo.

