# Usuarios y privilegios

El modelo de seguridad de Unix y Linux se basa en un principio fundamental: el usuario que ejecuta un proceso tiene exactamente los privilegios de ese usuario, ni mas ni menos. En un servidor, donde multiples servicios comparten el mismo sistema operativo y donde el acceso remoto es la norma, la gestion de usuarios y privilegios es la primera linea de defensa. Un compromise de credenciales de un usuario sin privilegios limita el dano al contexto de ese usuario; un compromise de credenciales de root equivole al control total del sistema.

## Principio de minimo privilegio

El principio de minimo privilegio (principle of least privilege) establece que cada usuario, proceso o servicio debe tener solo los permisos necesarios para realizar su funcion y nada mas. En contexto de servidor, esto se traduce en practicas concretas: el servicio web no necesita permisos para leer el directorio de la base de datos; el usuario de backup no necesita acceso de escritura a los logs; el administrador remoto no necesita root directo sino que opera con sudo desde un usuario regular.

Este principio no es solo una recomendacion de seguridad, es un requisito de cumplimiento normativo en estandares como ISO 27001, PCI-DSS y SOC 2. Las auditorias de seguridad verifican explicitamente que los privilegios estan asignados de forma granular y justificada.

## Deshabilitar el login directo de root

El usuario `root` es el superusuario de un sistema Unix: tiene acceso irrestricto a todos los archivos, dispositivos y configuraciones. Permitir el inicio de sesion directo como root via SSH es una de las practicas mas peligrosas en un servidor. Si un atacante obtiene la contraseña de root, puede instalar backdoors, cifrar el sistema con ransomware o exfiltrar datos sin ningun obstaculo adicional.

Deshabilitar el login directo de root no elimina al usuario root; simplemente impide que se autentique directamente. En su lugar, los administradores se autentican como usuarios regulares y elevan privilegios con `sudo` cuando es necesario. Esto genera un registro de auditoria (quien ejecuto que comando con privilegios elevados) y obliga a un paso adicional que dificulta ataques de fuerza bruta.

```bash
# Editar la configuracion SSH
sudo vim /etc/ssh/sshd_config

# Establecer
PermitRootLogin no

# Reiniciar el servicio SSH para aplicar cambios
sudo systemctl restart sshd
```

> Deshabilitar el login de root via SSH solo tiene sentido si existe al menos un usuario con capacidad sudo. Si se deshabilita root sin haber creado previamente un usuario administrador, el servidor queda inaccesible. Verificar siempre que la elevacion de privilegios funciona antes de bloquear root.

## Crear usuarios con sudo

### Creacion de usuario administrador

El primer paso es crear un usuario que servira como punto de entrada administrativo. Este usuario debe tener un nombre que no sea facilmente adivinable (evitar nombres como `admin`, `administrator` o `user`) y debe pertenecer al grupo `sudo` o `wheel` segun la distribucion.

```bash
# Crear usuario (ejemplo con nombre no obvio)
sudo adduser unai_admin

# Añadir al grupo sudo (Debian/Ubuntu)
sudo usermod -aG sudo unai_admin

# En RHEL/Rocky, el grupo es wheel
sudo usermod -aG wheel unai_admin
```

### Configuracion de sudo

El archivo `/etc/sudoers` controla que usuarios pueden ejecutar comandos con privilegios elevados y bajo que condiciones. La herramienta `visudo` edita este archivo con validacion de sintaxis, lo que previene errores de formato que bloquearian el acceso sudo para todos los usuarios.

```bash
# Editar sudoers de forma segura
sudo visudo

# Conceder acceso sudo completo a un usuario
unai_admin ALL=(ALL:ALL) ALL

# Conceder acceso sudo sin contraseña (menos seguro, pero util en automatizaciones)
unai_admin ALL=(ALL:ALL) NOPASSWD: ALL
```

### Configuracion de sudo por comando

En lugar de conceder acceso sudo completo, se pueden restringir los comandos que un usuario puede ejecutar con privilegios elevados. Esto es util para usuarios de soporte que necesitan reiniciar servicios especificos pero no deberian poder modificar la configuracion del sistema.

```bash
# Permitir solo reiniciar el servicio nginx
unai_admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx

# Permitir ejecutar solo un script de mantenimiento
unai_admin ALL=(ALL) NOPASSWD: /opt/scripts/maintenance.sh
```

## Gestion de grupos

Los grupos organizan usuarios con necesidades de permisos similares. En lugar de asignar permisos individuales a cada usuario, se crean grupos como `webadmin`, `dbadmin` o `deploy` y se asignan permisos a nivel de grupo. Cuando un nuevo usuario necesita los mismos permisos, se añade al grupo existente sin modificar permisos individuales.

```bash
# Crear grupo para administradores web
sudo groupadd webadmin

# Añadir usuarios al grupo
sudo usermod -aG webadmin unai
sudo usermod -aG webadmin carlos

# Aplicar permisos de directorio al grupo
sudo chown -R root:webadmin /var/www
sudo chmod -R 775 /var/www
```

## Contraseñas y politicas de autenticacion

### Fortaleza de contraseñas

Un servidor expuesto a Internet recibe constantemente intentos de conexion automatizados. Las contraseñas debiles son la primera barrera que un atacante supera con herramientas de fuerza bruta. Las politicas de contraseñas deben exigir longitud minima (12 caracteres o mas), complejidad razonable (mezcla de caracteres) y prohibir contraseñas comprometidas en filtraciones conocidas.

```bash
# Configurar politica de contraseñas con pam_pwquality
# /etc/security/pwquality.conf
minlen = 12
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
```

### Bloqueo de cuentas tras fallos de autenticacion

`pam_faillock` bloquea una cuenta despues de un numero determinado de intentos fallidos, protegiendo contra ataques de fuerza bruta. Esto es especialmente importante para servicios con acceso directo como SSH.

```bash
# Configurar bloqueo tras 5 intentos fallidos
# /etc/pam.d/common-auth o /etc/security/faillock.conf
deny = 5
unlock_time = 900  # Desbloquear automaticamente tras 15 minutos
```

## Eliminacion de usuarios innecesarios

Los sistemas operativos vienen con usuarios del sistema creados por defecto (daemon, mail, www-data, etc.). Estos usuarios existen para que los servicios se ejecuten bajo identidades aisladas y con los minimos permisos necesarios. No deben eliminarse ni modificarse a menos que se entienda exactamente que servicio depende de ellos. Sin embargo, los usuarios de login creados durante la instalacion o con posterioridad que ya no son necesarios deben ser eliminados para reducir la superficie de ataque.

```bash
# Eliminar usuario y su directorio home
sudo deluser --remove-home usuario_obsoleto
```

