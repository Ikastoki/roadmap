# Firewall basico con ufw

Un firewall es la primera linea de defensa de la seguridad de la red. Filtra el trafico entrante y saliente segun reglas configuradas por el administrador, bloqueando conexiones no deseadas y permitiendo las legitimas. En Linux, `ufw` (*Uncomplicated Firewall*) es la herramienta mas simple y comun para gestionar el firewall.

UFW proporciona una interfaz de alto nivel sobre `iptables`/`nftables`, simplificando la creacion de reglas de firewall sin necesidad de entender la sintaxis compleja de las herramientas inferiores.

## Conceptos basicos de firewall

Un firewall de red opera sobre reglas que definen que trafico permitir y que trafico bloquear. Las reglas se basan en:

- **Direccion IP:** origen o destino del paquete.
- **Puerto:** puerto TCP o UDP de destino.
- **Protocolo:** TCP, UDP, ICMP.
- **Interfaz:** interfaz de entrada o salida.
- **Direccion:** INPUT (trafico entrante), OUTPUT (trafico saliente), FORWARD (trafico reenviado).

La estrategia por defecto determina que pasa con el trafico que no coincide con ninguna regla: **allow** (permitir) o **deny** (bloquear).

## UFW: comandos esenciales

```bash
# Activar ufw
sudo ufw enable

# Ver estado
sudo ufw status verbose

# Estrategia por defecto: bloquear todo entrante, permitir todo saliente
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir servicios
sudo ufw allow 22/tcp     # SSH
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS
sudo ufw allow 3306/tcp   # MySQL (solo si necesario)

# Bloquear un puerto
sudo ufw deny 23/tcp      # Bloquear Telnet

# Permitir por IP
sudo ufw allow from 192.168.1.0/24
sudo ufw allow from 10.0.0.5 to any port 22

# Reglas con rango de puertos
sudo ufw allow 6000:6100/tcp  # Rango de puertos
```

## Reglas para servicios comunes

```bash
# Servidor web
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# SSH (restringido a una IP)
sudo ufw allow from 203.0.113.50 to any port 22

# MySQL (solo desde la red local)
sudo ufw allow from 192.168.1.0/24 to any port 3306

# DNS
sudo ufw allow 53
```

## Gestion de reglas

```bash
# Ver reglas numeradas
sudo ufw status numbered

# Eliminar una regla por numero
sudo ufw delete 3

# Eliminar una regla por contenido
sudo ufw delete allow 80/tcp

# Insertar regla en una posicion especifica
sudo ufw insert 1 allow from 10.0.0.0/8

# Reiniciar ufw
sudo ufw disable
sudo ufw enable
```

## Logs del firewall

```bash
# Activar logs
sudo ufw logging on

# Ver logs del firewall
sudo tail -f /var/log/ufw.log

# Niveles de log: off, low, medium, high, full
sudo ufw logging medium
```

Los logs del firewall son esenciales para detectar intentos de acceso no autorizados y para verificar que las reglas estan funcionando correctamente.

## Limitaciones de ufw

UFW es una herramienta basica que puede no ser suficiente para entornos empresariales. Para escenarios mas complejos, se utilizan:

- **iptables/nftables:** herramientas de bajo nivel con mayor flexibilidad.
- **firewalld:** firewall dinamico utilizado en RHEL/CentOS/Fedora.
- **pfsense/OPNsense:** firewalls basados en BSD con interfaz grafica.
- **WAF (*Web Application Firewall*):** firewall de capa de aplicacion para proteger servidores web.

> UFW es la puerta de entrada a la seguridad de red en Linux. Activar un firewall con una politica de "deny incoming, allow outgoing" y abrir solo los puertos necesarios es la accion de seguridad mas efectiva y simple que cualquier administrador puede implementar. Un firewall bien configurado reduce dramaticamente la superficie de ataque del sistema.

