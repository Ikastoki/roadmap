# Usuario root y sudo

El usuario root es el administrador del sistema con privilegios absolutos. Puede leer, escribir y ejecutar cualquier archivo, modificar la configuracion del sistema, crear y eliminar usuarios, y detener o iniciar cualquier servicio. Este poder es necesario para tareas administrativas, pero trabajar directamente como root introduce riesgos graves: un error tipografico como `rm -rf / var` puede destruir el sistema sin posibilidad de recuperacion, y cualquier programa ejecutado como root tiene acceso completo a todo el sistema.

## El problema de trabajar como root

Trabajar como root de forma permanente rompe el principio de menor privilegio y tiene consecuencias concretas:

- No hay auditoria de acciones: el sistema registra que root ejecuto algo, pero no permite identificar que usuario humano lo hizo.
- Cualquier vulnerabilidad en un programa ejecutado como root compromete todo el sistema.
- Un error en un comando puede tener consecuencias irreversibles.
- No hay distincion entre tareas administrativas y tareas cotidianas.

## sudo

`sudo` (superuser do) permite a usuarios autorizados ejecutar comandos con privilegios elevados sin necesidad de conocer la contrasena de root ni iniciar sesion como root. Cada ejecucion queda registrada en los logs del sistema (`/var/log/auth.log` en Debian/Ubuntu, `journalctl` en sistemas con systemd).

```bash
# Ejecutar un comando como root
sudo apt update

# Ejecutar un comando como otro usuario
sudo -u maria whoami

# Obtener una shell como root
sudo -i

# Obtener una shell como otro usuario
sudo -u maria -i

# Ejecutar el comando anterior con sudo
sudo !!
```

### Ventajas de sudo frente a su

| Caracteristica | sudo | su |
|---|---|---|
| Requiere contrasena de root | No (contrasena del usuario) | Si |
| Auditoria por usuario | Si | No |
| Privilegios granulares | Si (por comando) | No |
| Sesion root persistente | No (por comando) | Si |
| Registro en logs | Si | No |

## /etc/sudoers

El archivo `/etc/sudoers` define que usuarios y grupos pueden ejecutar que comandos con que privilegios. Nunca se edita directamente con un editor de texto; debe usarse `visudo`, que valida la sintaxis antes de guardar.

```bash
# Editar sudoers de forma segura
sudo visudo

# Verificar sintaxis sin abrir editor
sudo visudo -c
```

Estructura basica de una regla en sudoers:

```
usuario  hostname=(runas:runasgrupo) TAG: comando
```

Ejemplos de configuracion:

```
# Otorgar todos los privilegios a un usuario en el grupo sudo
%sudo   ALL=(ALL:ALL) ALL

# Permitir a un usuario ejecutar solo apt
juan    ALL=(ALL) /usr/bin/apt

# Permitir reiniciar servicios sin contrasena
maria   ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart *, /usr/bin/systemctl start *

# Permitir comandos como otro usuario especifico
antonio ALL=(www-data) /usr/bin/kill

# Denegar explicitamente un comando
pedro   ALL=(ALL) ALL, !/usr/bin/passwd root
```

### Alias en sudoers

Para simplificar configuraciones complejas, sudoers soporta alias:

```
User_Alias  ADMINS = juan, maria, antonio
User_Alias  DEV = pedro, lucia
Cmnd_Alias  SERVICIOS = /usr/bin/systemctl *, /usr/bin/journalctl *
Host_Alias  SERVIDORES = web01, web02, db01

ADMINS      SERVIDORES=(ALL) ALL
DEV         SERVIDORES=(ALL) SERVICIOS
```

### Buenas practicas de seguridad con sudo

- Limitar los comandos que cada usuario puede ejecutar con sudo al minimo necesario.
- Usar `NOPASSWD` solo para comandos especificos y no peligrosos.
- Evitar reglas con `ALL` excepto para usuarios administrativos de confianza.
- Revisar periodicamente los logs de sudo con `journalctl -t sudo` o `grep sudo /var/log/auth.log`.
- Usar `sudo -l` para que los usuarios verifiquen que comandos tienen permitidos.
- Mantener el archivo sudoers en un sistema de control de versiones si se gestiona centralizadamente.

```bash
# Ver los comandos que el usuario actual puede ejecutar con sudo
sudo -l

# Ver los comandos que otro usuario puede ejecutar
sudo -l -U juan

# Buscar intentos fallidos de sudo en logs
sudo journalctl -t sudo | grep -i "FAILED"
```

> Configurar sudo con `timestamp_timeout=0` en sudoers fuerza a introducir la contrasena en cada uso. Aunque mas seguro, puede resultar impracticable en entornos de administracion intensiva.

