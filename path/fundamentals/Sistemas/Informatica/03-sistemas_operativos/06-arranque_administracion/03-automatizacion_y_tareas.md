# Automatizacion y tareas

La automatizacion es el proceso de ejecutar tareas repetitivas sin intervencion manual, liberando al administrador de sistemas de operaciones mundanas y reduciendo la posibilidad de error humano. En Linux, existen multiples herramientas de automatizacion que van desde la ejecucion periodica de comandos hasta la gestion configurada de infraestructura completa.

La automatizacion no es un lujo sino una necesidad en cualquier entorno con mas de un servidor: mantener actualizaciones de seguridad, rotar logs, hacer copias de seguridad y monitorizar servicios son tareas que deben ejecutarse sin intervencion humana.

## Cron y crontab

`cron` es el demonio de tareas periodicas mas antiguo y extendido de Unix/Linux. Cada usuario puede tener su propio `crontab` con tareas programadas, y el sistema tiene un crontab global.

```bash
# Editar crontab del usuario actual
crontab -e

# Formato de una linea de crontab
# ┌───── minuto (0-59)
# │ ┌───── hora (0-23)
# │ │ ┌───── dia del mes (1-31)
# │ │ │ ┌───── mes (1-12)
# │ │ │ │ ┌───── dia de la semana (0-7, 0 y 7 = domingo)
# │ │ │ │ │
# * * * * * comando

# Ejemplos
0 2 * * * /usr/local/bin/backup.sh           # Todos los dias a las 2:00
*/5 * * * * /usr/local/bin/monitorizar.sh     # Cada 5 minutos
0 0 * * 0 /usr/local/bin/limpiar_cache.sh     # Todos los domingos a medianoche
30 8 1 * * /usr/local/bin/reporte_mensual.sh  # Dia 1 de cada mes a las 8:30
```

Los crontabs del sistema se almacenan en `/etc/cron.d/` y se ejecutan con los privilegios especificados. `/etc/crontab` es el crontab global del sistema.

```bash
# Ver crontab del usuario
crontab -l

# Ver crontabs del sistema
ls /etc/cron.d/
cat /etc/crontab

# Logs de cron
grep CRON /var/log/syslog
journalctl -u cron
```

## Systemd timers

Los timers de systemd son el reemplazo moderno de cron, con ventajas como la activacion persistente (ejecutar tareas perdidas durante apagones), la integracion con el ecosistema systemd y la capacidad de activar servicios por demanda:

```bash
# /etc/systemd/system/backup.timer
[Unit]
Description=Backup diario

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target

# /etc/systemd/system/backup.service
[Unit]
Description=Ejecutar backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```bash
# Activar y verificar timers
sudo systemctl enable --now backup.timer
systemctl list-timers --all
systemctl status backup.timer
```

## At y batch

`at` ejecuta un comando una vez en un momento especifico en el futuro. Es util para tareas unicas programadas:

```bash
# Ejecutar un comando en 30 minutos
echo "/usr/local/bin/tarea.sh" | at now + 30 minutes

# Ejecutar manana a las 8:00
echo "/usr/local/bin/reportes.sh" | at 8:00 tomorrow

# Ver tareas pendientes
atq

# Eliminar una tarea
atrm 3
```

`batch` ejecuta comandos cuando la carga del sistema baja de un umbral especifico (por defecto 1.5):

```bash
# Ejecutar tarea pesada cuando el sistema este tranquilo
batch <<< "/usr/local/bin/reindexar.sh"
```

## Ansible: automatizacion de infraestructura

Ansible es una herramienta de automatizacion y provisionamiento que utiliza un enfoque declarativo basado en archivos YAML. A diferencia de cron o systemd timers, Ansible gestiona multiples servidores simultaneamente desde un nodo central, sin necesidad de instalar agentes en los servidores remotos (usa SSH).

```yaml
# playbook.yml - Instalar y configurar Nginx en multiples servidores
---
- hosts: web_servers
  become: yes
  vars:
    nginx_port: 80
    server_name: ejemplo.com

  tasks:
    - name: Instalar Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Copiar configuracion
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Reiniciar Nginx

    - name: Asegurar que Nginx esta iniciado
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Reiniciar Nginx
      service:
        name: nginx
        state: restarted
```

```bash
# Ejecutar playbook
ansible-playbook -i inventory.ini playbook.yml

# Verificar cambios sin aplicar (--check es dry-run)
ansible-playbook -i inventory.ini playbook.yml --check
```

## Cron vs systemd timers vs Ansible

| Caracteristica | Cron | Systemd Timers | Ansible |
|---|---|---|---|
| Alcance | Local | Local | Multiples servidores |
| Configuracion | Lineas de texto | Archivos .unit | Playbooks YAML |
| Persistencia | Si | Si (Persistent=true) | No (ejecucion bajo demanda) |
| Activacion por demanda | No | Si (.socket) | Bajo demanda |
| Logs | /var/log/syslog | journalctl | Salida del playbook |
| Complejidad | Baja | Media | Alta |

> La automatizacion bien implementada es la diferencia entre un sistema que funciona y un sistema que se gestiona eficientemente. Comenzar con cron para tareas simples, avanzar a systemd timers para tareas que requieren persistencia, y adoptar Ansible o similar cuando la gestion abarca multiples servidores es una progresion natural y recomendable.

