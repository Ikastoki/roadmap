# Informacion de usuarios y grupos

Linux proporciona varios mecanismos para consultar informacion sobre usuarios, grupos y sesiones activas. Estos archivos y comandos son esenciales para auditar quien tiene acceso al sistema, que usuarios estan conectados y a que grupos pertenecen.

## /etc/passwd

El archivo `/etc/passwd` contiene informacion basica de cada cuenta de usuario. Cada linea representa un usuario con siete campos separados por dos puntos:

```
juan:x:1001:1001:Juan Perez:/home/juan:/bin/bash
```

| Campo | Significado |
|---|---|
| `juan` | Nombre de usuario |
| `x` | Indica que la contrasena esta en `/etc/shadow` |
| `1001` | UID (User ID) |
| `1001` | GID (Group ID) del grupo primario |
| `Juan Perez` | Campo GECOS (informacion del usuario, separado por comas) |
| `/home/juan` | Directorio home |
| `/bin/bash` | Shell de inicio de sesion |

```bash
# Ver todos los usuarios del sistema
cat /etc/passwd

# Ver solo los usuarios humanos (UID >= 1000 tipicamente)
awk -F: '$3 >= 1000 {print $1, $6}' /etc/passwd

# Buscar un usuario especifico
grep "^juan:" /etc/passwd

# Ver solo los nombres de usuario
cut -d: -f1 /etc/passwd
```

Los rangos de UID estandar son:

| Rango | Tipo |
|---|---|
| 0 | root |
| 1-999 | Cuentas de sistema (servicios, daemons) |
| 1000-65533 | Usuarios humanos |
| 65534 | nobody |
| 65535-4294967295 | UIDs de sistema en sistemas 32 bits |

> En distribuciones modernas, el limite inferior para usuarios humanos puede ser 1000 (Debian/Ubuntu) o 500 (Red Hat/Fedora). Verificar con `grep "^UID_MIN" /etc/login.defs`.

## /etc/group

El archivo `/etc/group` almacena la informacion de los grupos del sistema:

```
docker:x:999:juan,maria
```

| Campo | Significado |
|---|---|
| `docker` | Nombre del grupo |
| `x` | Contrasena de grupo (normalmente vacia, almacenada en `/etc/gshadow`) |
| `999` | GID (Group ID) |
| `juan,maria` | Lista de miembros del grupo (solo grupos secundarios) |

Los miembros del grupo primario no aparecen en `/etc/group`; estan definidos por el GID en `/etc/passwd`.

```bash
# Ver todos los grupos
cat /etc/group

# Ver los grupos a los que pertenece un usuario
groups juan

# Buscar miembros de un grupo especifico
grep "^docker:" /etc/group
```

## Comando id

`id` muestra la identidad completa de un usuario: UID, GID primario y todos los GIDs secundarios.

```bash
# Ver identidad del usuario actual
id

# Ver identidad de un usuario especifico
id juan

# Mostrar solo UID
id -u juan

# Mostrar solo GID primario
id -g juan

# Mostrar todos los GIDs
id -G juan

# Mostrar nombres en lugar de numeros
id -n juan
```

## Comando who

`who` lista los usuarios que tienen una sesion activa en el sistema.

```bash
# Usuarios conectados actualmente
who

# Mostrar solo nombres de usuario
who -q

# Mostrar hora de inicio del sistema
who -b

# Mostrar todos los datos disponibles
who -a
```

## Comando w

`w` muestra los usuarios conectados y su actividad actual, incluyendo carga del sistema, tiempo de actividad y el comando que esta ejecutando cada usuario.

```bash
# Mostrar sesiones activas con actividad
w

# Mostrar sin encabezado
w -h

# Mostrar informacion de un usuario especifico
w juan
```

La salida tipica de `w`:

```
 10:30:15 up 3 days,  2:15,  2 users,  load average: 0.08, 0.03, 0.01
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
juan     pts/0    192.168.1.10     09:15    0.00s  0.25s  0.10s sshd: juan
maria    pts/1    192.168.1.11     10:00    5:30   0.15s  0.05s -bash
```

## Comando last

`last` consulta el archivo `/var/log/wtmp` para mostrar el historial de inicios de sesion.

```bash
# Mostrar historial de inicios de sesion
last

# Mostrar inicios de un usuario especifico
last juan

# Mostrar los ultimos N inicios
last -5

# Mostrar intentos fallidos de inicio de sesion
sudo lastb
```

> Los archivos `wtmp` y `btmp` (intentos fallidos) son binarios y rotan periodicamente. Su tamanio puede crecer significativamente en sistemas con muchos inicios de sesion.

