# Herramientas de seguridad

Mas alla de las politicas de firewall, autenticacion y permisos, existen herramientas especificas disenadas para detectar y prevenir amenazas en servidores Linux. Estas herramientas cubren dos frentes principales: la proteccion activa contra intentos de intrusion (como fuerza bruta contra servicios de autenticacion) y la deteccion pasiva de compromisos del sistema (como rootkits o binarios modificados). Ninguna herramienta por si sola garantiza la seguridad de un servidor, pero su uso combinado forma una defensa en profundidad efectiva.

## fail2ban: proteccion contra fuerza bruta

fail2ban es un守护神 daemon que monitoriza los ficheros de registro de servicios como SSH, Apache, Postfix o nginx en busca de patrones que indiquen un intento de ataque —normalmente contraseñas incorrectas repetidas desde la misma IP—. Cuando detecta un numero de intentos fallidos que supera un umbral configurado, genera automaticamente una regla temporal en el firewall (iptables, nftables o firewalld) que bloquea la direccion IP del atacante durante un periodo de tiempo.

El funcionamiento se basa en un ciclo de tres pasos: primero, fail2ban ejecuta un filtro regex contra los logs del servicio para identificar las lineas relevantes (por ejemplo, "Failed password for root from 192.168.1.100"); segundo, cuenta las coincidencias por IP dentro de una ventana temporal; tercero, si el conteo supera el maxretry configurado, ejecuta un action (comando externo) que añade la IP a la cadena de bloqueo del firewall.

```bash
apt install fail2ban
systemctl enable --now fail2ban
```

La configuracion se organiza en `/etc/fail2ban/jail.local`. Cada servicio tiene su propia seccion con parametros como `maxintentos` (numero de fallos permitidos), `bantime` (duracion del bloqueo) y `findtime` (ventana de observacion). Un ejemplo de configuracion para SSH:

```ini
[sshd]
enabled = true
port = 22
maxretry = 3
bantime = 3600
findtime = 600
```

Con esta configuracion, una IP que falle 3 veces en 10 minutos al autenticarse en SSH sera bloqueada durante una hora. fail2ban tambien puede enviar notificaciones por correo electronico cada vez que ejecuta una accion de bloqueo.

> fail2ban no reemplaza un firewall configurado correctamente. Actua como una capa adicional que reacciona dinamicamente a comportamientos sospechosos. Un firewall bien configurado con politica default-deny ya bloquea el trafico no deseado; fail2ban protege los servicios que si estan abiertos al publico.

## Deteccion de rootkits: rkhunter y chkrootkit

Un rootkit es un conjunto de herramientas maliciosas diseñadas para ocultar la presencia de un atacante en el sistema. Los rootkits mas sofisticados modifican binarios del sistema (como `ls`, `ps` o `netstat`) para que no muestren archivos, procesos o conexiones sospechosas, lo que dificulta su deteccion con las herramientas normales del sistema.

**rkhunter** (Rootkit Hunter) compara los binarios del sistema contra una base de datos de hashes conocidos, verifica permisos y configuraciones inusuales, y detecta modificaciones sospechosas en archivos criticos como `/etc/passwd` o `/etc/shadow`. Tambien busca firmas conocidas de rootkits en archivos temporales y procesos activos.

```bash
apt install rkhunter
rkhunter --update
rkhunter --check
```

**chkrootkit** (Check Rootkit) adopta un enfoque similar pero mas ligero. Revisa archivos del sistema en busca de firmas de rootkits conocidos, verifica la integridad de binarios con `strings` (buscando cadenas inusuales que podrian indicar inyeccion de codigo), y examina procesos ocultos y conexiones de red no normales.

```bash
apt install chkrootkit
chkrootkit
```

Ambas herramientas son utiles como auditorias periodicas, pero tienen limitaciones importantes: solo detectan rootkits conocidos en la base de datos de firmas, no protegen contra rootkits zero-day o techniques avanzadas como los que operan en el kernel o en el nivel de hipervisor. Para una deteccion mas robusta se complementan con herramientas de monitoreo de integridad de archivos como AIDE o Tripwire.

