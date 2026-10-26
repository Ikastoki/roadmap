# Mantenimiento rutinario

Un servidor no es una maquina que se configura una vez y funciona para siempre. La infraestructura tecnica acumula residuos con el paso del tiempo: paquetes obsoletos que nunca se eliminan, logs que crecen silenciosamente, certificados que caducan, dependencias que se vuelven incompatibles. El mantenimiento rutinario es la disciplina de revisar y limpiar estos residuos de forma periodica, evitando que la deuda tecnica se acumule hasta provocar un fallo o dificultar una actualizacion critica.

## Limpieza de paquetes

Los sistemas de paquetes como `apt` mantienen un historial de instalaciones y actualizaciones que genera residuos: paquetes instalados como dependencias que ya no son necesarios, cache de descargas que ocupa espacio, paquetes retroidos que permanecen instalados aunque versiones mas nuevas estan disponibles. Estos residuos no solo ocupan espacio, sino que pueden crear conflictos de dependencias y ampliar la superficie de ataque al mantener versiones antiguas con vulnerabilidades conocidas.

```bash
# Eliminar paquetes huérfanos (instalados como dependencias, ya no requeridos)
sudo apt autoremove

# Limpiar cache de descargas de paquetes
sudo apt clean

# Mostrar paquetes retroidos (con versiones mas nuevas disponibles)
apt list --upgradable
```

El comando `apt autoremove` identifica paquetes que fueron instalados automaticamente para satisfacer dependencias de otros paquetes, pero que ya no son necesarios. `apt clean` elimina los archivos `.deb` descargados en `/var/cache/apt/archives/`, que pueden acumular cientos de megabytes. Estas operaciones son seguras en la mayoria de casos, pero es buena practica revisar la lista de paquetes a eliminar antes de confirmar.

## Revision de logs de seguridad

`/var/log/auth.log` es uno de los archivos mas importantes en un servidor Linux. Registra todos los intentos de autenticacion, tanto exitosos como fallidos, asi como cambios de usuario y operaciones de sudo. Una revision periodica de este archivo puede revelar intentos de fuerza bruta, accesos desde IPs desconocidas o usos inesperados de privilegios elevados.

Los patrones mas relevantes a buscar incluyen multiples intentos fallidos de login desde una misma IP (posible fuerza bruta), logins exitosos desde ubicaciones geograficas inesperadas, uso de `sudo` por usuarios que normalmente no lo utilizan, y errores de autenticacion de servicios como SSH o bases de datos. Herramientas como `fail2ban` complementan esta revision al bloquear automaticamente IPs con multiples intentos fallidos, pero la revision manual periodica sigue siendo necesaria para detectar patrones mas sutiles.

```bash
# Intentos fallidos de SSH en las ultimas 24 horas
sudo journalctl -u ssh --since "24 hours ago" | grep "Failed password"

# Logins exitosos hoy
sudo last -i | head -20

# IPs con mas intentos fallidos
sudo grep "Failed password" /var/log/auth.log | \
  awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
```

## Prueba de restauracion de backups

Como se menciono en la seccion de copias de seguridad, un backup sin prueba de restauracion es una suposicion. El mantenimiento rutinario debe incluir la verificacion periodica de que los backups se estan creando correctamente y que los datos son restaurables. Esto no significa restaurar el servidor completo cada semana, sino al menos verificar la integridad del repositorio de backups y restaurar un archivo o directorio de prueba.

Herramientas como Borg y Restic incluyen comandos de verificacion de integridad que revisan que todos los bloques del repositorio sean legibles y que las sumas de verificacion coincidan. Ejecutar esta verificacion de forma regular detecta problemas de degradacion de almacenamiento antes de que se necesite una restauracion real.

```bash
# Verificar integridad de un repositorio Borg
borg check /mnt/backup

# Listar snapshots disponibles
borg list /mnt/backup
```

## Deuda tecnica acumulada

El concepto de deuda tecnica, originalmente del desarrollo de software, aplica igualmente a la gestion de servidores. Cada decision postergada —una actualizacion pendiente, una configuracion temporal que nunca se revirtio, un servicio que necesita reiniciarse pero nunca se reinicia— es un interes que se acumula con el tiempo. Un servidor sin mantenimiento rutinario puede funcionar durante meses, pero cuando finalmente ocurre un incidente, la recuperacion es mucho mas compleja porque los problemas se han entrelazado.

El mantenimiento rutinario no es un gasto de tiempo, es una inversion en estabilidad. Un servidor limpio, actualizado y monitoreado es mas predecible, mas seguro y mas facil de recuperar ante un fallo. La disciplina de dedicar un bloque de tiempo periodico —semanal o mensual— a tareas de mantenimiento previene la acumulacion de problemas que, individualmente, parecen insignificantes pero conjuntamente pueden provocar un colapso.

