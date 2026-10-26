# Auditoria y logging

La auditoria y el registro de eventos (*logging*) son componentes esenciales de la seguridad operativa. El logging permite detectar intrusiones, diagnosticar fallos, cumplir con requisitos regulatorios y realizar analisis forense despues de un incidente. Un sistema sin registro de eventos es un sistema ciego ante amenazas y problemas.

Linux dispone de multiples subsistemas de logging, desde los logs basicos de syslog hasta los sistemas de auditoria avanzados como auditd, que registran cada accion de cada usuario y proceso del sistema.

## Syslog y journald

**syslog** es el sistema tradicional de registro de eventos en Unix/Linux. Los mensajes se envian a un demonio syslog (rsyslog o syslog-ng) que los clasifica por facility (kern, user, mail, daemon, etc.) y severity (emergency, alert, critical, error, warning, notice, info, debug), y los almacena en archivos de log o los envia a un servidor centralizado.

```bash
# Logs del sistema en /var/log/
/var/log/syslog         # Log general del sistema (Debian/Ubuntu)
/var/log/messages       # Log general (RHEL/CentOS)
/var/log/auth.log       # Autenticacion y autorizacion (Debian/Ubuntu)
/var/log/secure         # Autenticacion y autorizacion (RHEL/CentOS)
/var/log/kern.log       # Mensajes del kernel
/var/log/dmesg          # Mensajes de arranque del hardware
```

**journald** es el sistema de logging estructurado introducido por systemd. Almacena logs en formato binario indexado, permite busquedas por campos y metadatos, y puede almacenar logs temporalmente en memoria (volatile) o permanentemente en disco:

```bash
# Ver logs del arranque actual
journalctl -b

# Buscar errores desde la ultima hora
journalctl --since "1 hour ago" --priority=err

# Ver logs de un servicio especifico
journalctl -u nginx.service

# Seguir logs en tiempo real
journalctl -f

# Logs persistentes vs volatiles
# /etc/systemd/journald.conf
[Journal]
Storage=persistent  # o 'volatile' o 'auto'
SystemMaxUse=500M
```

## Auditoria con auditd

`auditd` es el subsistema de auditoria del kernel Linux. A diferencia de syslog, que registra eventos de alto nivel, auditd puede rastrear cada accion a nivel de syscall: que archivos se abrieron, que comandos se ejecutaron, que conexiones de red se establecieron.

```bash
# Instalar auditd
sudo apt install auditd

# Regla: monitorizar cambios en /etc/passwd
sudo auditctl -w /etc/passwd -p wa -k password_changes

# Regla: monitorizar ejecucion de comandos
sudo auditctl -a always,exit -F arch=b64 -S execve -k comandos_ejecutados

# Buscar eventos
sudo ausearch -k password_changes
sudo ausearch -k comandos_ejecutados --start today

# Generar informe
sudo aureport --auth        # Informe de autenticacion
sudo aureport --failed      # Intentos fallidos
sudo aureport --login       # Informe de login
```

Las reglas de auditoria se pueden definir en `/etc/audit/rules.d/audit.rules` para que persistan entre reinicios.

## Log rotation

La rotacion de logs es esencial para evitar que los archivos de log crezcan indefinidamente y llenen el disco. `logrotate` es la herramienta estandar de Linux para gestionar la rotacion:

```
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
```

Este ejemplo rotaria los logs de Nginx diariamente, manteniendo 14 copias comprimidas, creando archivos nuevos con permisos 0640, y enviando una senyal USR1 a Nginx para que reabra sus archivos de log despues de la rotacion.

## Analisis forense

Cuando ocurre un incidente de seguridad, los logs son la fuente principal de informacion para el analisis forense:

```bash
# Buscar intentos de login fallidos
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Buscar conexiones SSH exitosas desde IPs inusuales
grep "Accepted publickey" /var/log/auth.log | awk '{print $11}' | sort | uniq -c

# Verificar procesos que se ejecutaron como root
journalctl _UID=0 --since "2 days ago"

# Revisar cambios en archivos criticos
ausearch -f /etc/passwd -ts recent
```

> Los logs son la memoria del sistema: sin ellos, un incidente de seguridad es un misterio sin resolver. La politica de retencion de logs debe equilibrar los requisitos de seguridad y cumplimiento (RGPD, ISO 27001) con el espacio en disco disponible. Los servidores criticos deben centralizar sus logs en un servidor externo (SIEM) para evitar que un atacante los elimine despues de comprometer el sistema.

