# Auditoria basica de accesos

Un servidor puede estar perfectamente configurado en cuanto a firewall, permisos y actualizaciones, pero si un atacante logra comprometer una cuenta de usuario y nadie lo detecta a tiempo, el dano sera igualmente grave. La auditoria de accesos es el proceso de revisar periodicamente los registros del sistema para identificar sesiones inusuales, usuarios con privilegios excesivos o conexiones desde origenes sospechosos. No es una herramienta de prevencion, pero es la unica forma de saber si las medidas preventivas estan funcionando o si algo ha pasado inadvertido.

## Consulta de sesiones pasadas: last y lastb

El comando `last` muestra el historial de todas las sesiones de login completadas en el sistema, incluyendo el usuario, la terminal desde la que se conecto, la IP de origen, la fecha y hora de inicio y fin, y la duracion. La informacion se extrae de `/var/log/wtmp`, que el kernel actualiza cada vez que un usuario inicia o cierra sesion.

```bash
last
last -10         # ultimas 10 sesiones
last -f /var/log/wtmp   # archivo especifico
```

El comando `lastb` muestra los intentos de login fallidos, extraidos de `/var/log/btmp`. Revisar `lastb` es particularmente valioso para detectar patrones de fuerza bruta que hayan sido bloqueados por fail2ban pero que no se habian inspeccionado previamente.

> Si `lastb` muestra miles de intentos fallidos desde una misma IP, es senal de que fail2ban (o el firewall) deberia estar activo. Si `last` muestra sesiones desde IPs desconocidas o en horarios inusuales, es motivo de investigacion inmediata.

## Sesion activa: who y w

Mientras `last` muestra historial, `who` y `w` muestran el estado actual del sistema: quienes estan conectados ahora, desde donde, y llevan cuanto tiempo sin actividad. `w` es mas detallado, incluyendo la carga del sistema y el proceso que esta ejecutando cada usuario.

```bash
who
w
```

Comparar los resultados de `w` con la lista de usuarios autorizados a conectarse al servidor revela rapida y visualmente si hay alguien que no deberia estar conectado. En un servidor de produccion con acceso restringido, la lista de sesiones activas deberia ser predecible y corta.

## Usuarios con acceso a shell

No todos los usuarios de un sistema Linux necesitan poder iniciar una sesion interactiva. Un usuario de servicio (como `www-data` o `mysql`) no deberia tener una shell valida en `/etc/passwd`, ya que eso le permitiria, en teoria, conectarse por SSH y ejecutar comandos. `getent passwd` lista todas las cuentas del sistema y sus shells asignadas:

```bash
getent passwd | grep -v '/nologin\|/false'
```

Esta simple comanda revela que usuarios tienen una shell real asignada (`/bin/bash`, `/bin/sh`, etc.) y cuales estan restringidos a `/usr/sbin/nologin` o `/bin/false`. Los usuarios que aparecen con shell real pero que no deberian tener acceso interactivo son candidatos a deshabilitar con `usermod -s /usr/sbin/nologin usuario`.

## Por que la auditoria periodica es esencial

Las herramientas de seguridad son reactivas: fail2ban bloquea despues de detectar un patron, y rkhunter busca firmas conocidas. La auditoria humana, por el contrario, puede identificar anomalias que las herramientas automatizadas no detectan —una conexion desde un pais inusual, un usuario que nunca se habia conectado antes de madrugada, o un patron de acceso que no encaja con el uso normal del servidor. Revisar los logs de acceso semanalmente, configurar alertas por correo para logins exitosos desde IPs nuevas, y mantener un registro de que usuarios deberian tener acceso al servidor son practicas basicas pero devastadoramente efectivas. Un atacante puede evadir un antivirus, pero le resulta mucho mas dificil pasar desapercibido si alguien revisa los logs con regularidad.

