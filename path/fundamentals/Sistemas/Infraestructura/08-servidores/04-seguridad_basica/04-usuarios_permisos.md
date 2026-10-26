# Gestion de usuarios y permisos

En un entorno multiusuario como un servidor Linux, la administracion cuidadosa de usuarios y permisos es fundamental para mantener la integridad del sistema. Cada proceso se ejecuta bajo la identidad de un usuario determinado, y el kernel utiliza esa identidad para decidir si el proceso tiene acceso a archivos, dispositivos o recursos del sistema. Un modelo de permisos mal gestionado puede permitir que un servicio comprometido acceda a datos que no le corresponden, o que un usuario normal ejecute operaciones administrativas sin autorizacion.

## Principio del minimo privilegio

Este principio establece que cada usuario y proceso debe tener solo los permisos estrictamente necesarios para realizar su funcion. Un servidor web no necesita un usuario con acceso de escritura a `/etc/passwd`, ni un usuario de soporte necesita permisos para instalar paquetes. El minimo privilegio reduce la superficie de ataque: si un atacante compromete un proceso, el dano esta limitado por los permisos del usuario bajo el cual se ejecuta ese proceso.

En la practica, esto implica crear usuarios dedicados para cada servicio (un usuario `www-data` para el servidor web, otro para la base de datos, otro para aplicaciones), nunca ejecutar servicios como root salvo que sea estrictamente necesario, y utilizar `sudo` para otorgar privilegios elevados de forma granular y auditada.

## Modelo de permisos de Linux

Cada archivo y directorio en Linux pertenece a un **usuario propietario** y a un **grupo propietario**. El sistema de permisos define tres tipos de acceso —lectura (r), escritura (w) y ejecucion (x)— para tres categorias de sujetos: el propietario, los miembros del grupo, y el resto de usuarios. Esto genera la notacion en tres bloques que se observa con `ls -l`:

```
-rwxr-x---  1 www-data www-data  4096 mar 15 10:30 config.yaml
```

Los primeros diez caracteres describen el tipo de archivo y los permisos: `rwxr-x---` significa que el propietario tiene lectura, escritura y ejecucion; el grupo tiene lectura y ejecucion; y los demas usuarios no tienen ningun acceso. Estos permisos se pueden modificar con `chmod` en su forma numerica (octal) o simbolica:

```bash
chmod 750 config.yaml    # rwxr-x---
chmod u+x script.sh      # anyadir ejecucion al propietario
chmod g-w data.csv       # quitar escritura al grupo
```

El cambio de propietario y grupo se realiza con `chown`, que requiere privilegios de root:

```bash
chown mysql:mysql /var/lib/mysql
chown -R www-data:www-data /var/www/html
```

> El bit **setuid** (s en la posicion de ejecucion del propietario) permite que un archivo se ejecute con los privilegios de su propietario independientemente de quien lo ejecute. `passwd` utiliza este bit para que los usuarios normales puedan cambiar su propia contraseña sin necesidad de root. Un setuid mal configurado en un binario puede ser un vector de escalada de privilegios.

## Grupos y administracion de usuarios

Los grupos facilitan la gestion de permisos colectivos. En lugar de asignar permisos a cada usuario individualmente, se crea un grupo (por ejemplo, `desarrollo` o `deploy`) y se añaden los usuarios que necesitan acceso. Luego se asignan permisos al grupo en los directorios y archivos relevantes:

```bash
groupadd deploy
usermod -aG deploy usuario1
usermod -aG deploy usuario2
chown -R :deploy /opt/aplicacion
chmod -R 775 /opt/aplicacion
```

Para gestionar usuarios se utilizan los comandos `useradd` (para crear), `usermod` (para modificar) y `userdel` (para eliminar). Es recomendable deshabilitar cuentas inactivas con `usermod -L` en lugar de eliminarlas, para preservar el historial y los archivos del usuario. La informacion de todos los usuarios se almacena en `/etc/passwd` (cuentas), `/etc/shadow` (hashes de contraseñas) y `/etc/group` (grupos), con permisos estrictos que solo permiten lectura a root.

