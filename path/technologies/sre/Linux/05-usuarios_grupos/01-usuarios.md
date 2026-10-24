# Creacion, eliminacion y modificacion de usuarios

Cada proceso en Linux se ejecuta bajo una identidad de usuario, y cada archivo pertenece a un usuario. Esta separacion de identidades es la base del modelo de seguridad del sistema: sin usuarios diferenciados, cualquier proceso tendria acceso a todos los recursos. La administracion de cuentas de usuario es, por tanto, una tarea fundamental que determina quien puede hacer que y que recursos puede consumir.

Los archivos de configuracion `/etc/passwd`, `/etc/shadow` y `/etc/group` almacenan la informacion de cuentas, pero nunca se editan manualmente; los comandos `useradd`, `userdel` y `usermod` son la interfaz correcta para gestionar usuarios, ya que actualizan de forma atomica y consistente todos los archivos involucrados.

## useradd

`useradd` crea una nueva cuenta de usuario en el sistema. A diferencia de simplemente anadir una linea en `/etc/passwd`, este comando crea el directorio home, copia los archivos de esqueleto, asigna un UID y establece la shell por defecto.

```bash
# Crear un usuario con valores por defecto del sistema
sudo useradd juan

# Crear usuario con directorio home y shell especifica
sudo useradd -m -s /bin/bash maria

# Crear usuario con UID y GID explicitos
sudo useradd -u 1500 -g 100 -m -s /usr/bin/zsh antonio

# Crear usuario sin directorio home (util para servicios)
sudo useradd -M -s /usr/sbin/nologin nginx

# Crear usuario con fecha de expiracion de cuenta
sudo useradd -e 2026-12-31 -m -s /bin/bash temporal
```

Los parametros mas relevantes de `useradd` son:

- `-m` crea el directorio home en `/home/usuario` y copia los archivos de `/etc/skel/`.
- `-s` define la shell de inicio de sesion. Usar `/usr/sbin/nologin` o `/bin/false` para cuentas de servicio que no deben iniciar sesion.
- `-u` asigna un UID especifico en lugar de usar el siguiente disponible.
- `-g` define el grupo primario (debe existir previamente).
- `-G` anade el usuario a grupos secundarios (separados por coma).
- `-e` establece la fecha de expiracion de la cuenta en formato `YYYY-MM-DD`.
- `-d` especifica un directorio home personalizado (no necesariamente en `/home/`).
- `-c` anade un comentario o descripcion (visible en el campo GECOS de `/etc/passwd`).

## userdel

`userdel` elimina una cuenta de usuario del sistema. Por defecto no elimina el directorio home ni el mail spool, lo que puede dejar archivos huerfanos.

```bash
# Eliminar usuario (conserva home y mail)
sudo userdel juan

# Eliminar usuario junto con su directorio home y mail spool
sudo userdel -r juan

# Forzar eliminacion incluso si el usuario tiene procesos activos
sudo userdel -f maria
```

> Usar `userdel -f` con precaucion: fuerza la eliminacion incluso si el usuario tiene procesos en ejecucion, lo que puede dejar procesos huerfanos ejecutandose sin control.

## usermod

`usermod` modifica los parametros de una cuenta existente. Es la herramienta para ajustar cuentas sin necesidad de eliminarlas y crearlas de nuevo.

```bash
# Cambiar el directorio home y mover los archivos existentes
sudo usermod -d /nuevo/home -m juan

# Cambiar la shell de inicio de sesion
sudo usermod -s /bin/zsh maria

# Bloquear la cuenta (impide el inicio de sesion)
sudo usermod -L antonio

# Desbloquear la cuenta
sudo usermod -U antonio

# Anadir a grupos secundarios
sudo usermod -aG sudo,docker juan

# Cambiar el UID
sudo usermod -u 2000 maria

# Cambiar el nombre de usuario
sudo usermod -l juan_nuevo juan
```

> El parametro `-a` (append) debe usarse siempre junto con `-G` para anadir grupos secundarios. Sin `-a`, `-G` reemplaza la lista completa de grupos secundarios del usuario.

### Bloqueo de cuentas

Existen varias tecnicas para bloquear una cuenta, cada una con distintos efectos:

```bash
# Metodo 1: Bloquear con usermod (anade ! al hash en /etc/shadow)
sudo usermod -L usuario

# Metodo 2: Bloquear con passwd
sudo passwd -l usuario

# Metodo 3: Forzar cambio de contrasena en el proximo inicio (expirar cuenta)
sudo passwd -e usuario

# Metodo 4: Bloquear cuenta por fecha de expiracion pasada
sudo usermod -e 1 usuario

# Verificar estado de la cuenta
sudo passwd -S usuario
```

El bloqueo con `usermod -L` o `passwd -l` anade el caracter `!` al inicio del hash de la contrasena en `/etc/shadow`, lo que hace imposible que el hash coincida con ninguna contrasena. La cuenta permanece bloqueada hasta que se desbloquee explicitamente.

