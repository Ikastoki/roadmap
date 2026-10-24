# Firewall en Linux

Un sistema conectado a una red sin un firewall configurado es como una puerta abierta: permite el paso de cualquier trafico entrante y saliente sin restriccion. El firewall en Linux actua como un filtro de paquetes que examina cada datagrama que atraviesa la pila de red y decide, segun las reglas definidas, si lo acepta, lo rechaza o lo descarta. Linux ofrece multiples capas de abstraccion para gestionar este filtrado, desde herramientas simplificadas como `ufw` hasta el control directo del kernel con `nftables`.

## Conceptos fundamentales: cadenas, politicas y reglas

El filtrado de paquetes en Linux se organiza en torno a tres conceptos:

- **Cadenas (chains):** conjuntos de reglas que se evaluan secuencialmente. Las cadenas principales son INPUT (trafico entrante), OUTPUT (trafico saliente) y FORWARD (trafico enrutado a traves del sistema).
- **Politicas (policies):** accion por defecto aplicada a un paquete que no coincide con ninguna regla de la cadena (ACCEPT o DROP).
- **Reglas (rules):** condiciones que evaluan atributos del paquete (IP origen, IP destino, puerto, protocolo, interfaz) y determinan la accion a tomar.

La politica por defecto define la postura del firewall: ACCEPT permite todo lo que no este explicitamente prohibido (whitelist de denegacion), mientras que DROP deniega todo lo que no este explicitamente permitido (whitelist de permisos). La segunda es la recomendada para servidores en entornos de produccion.

## ufw: firewall simplificado

`ufw` (Uncomplicated Firewall) es una interfaz de alto nivel sobre `iptables` disenada para facilitar la gestion del firewall en sistemas Ubuntu y derivados. Su sintaxis es intuitiva:

```bash
ufw enable                          # Activa el firewall
ufw disable                         # Desactiva el firewall
ufw default deny incoming           # Politica por defecto: denegar entrante
ufw default allow outgoing          # Politica por defecto: permitir saliente
ufw status verbose                  # Estado detallado con reglas activas
ufw status numbered                 # Reglas numeradas para gestion
```

Para abrir y cerrar puertos:

```bash
ufw allow 22/tcp                    # Permite SSH
ufw allow 80/tcp                    # Permite HTTP
ufw allow 443/tcp                   # Permite HTTPS
ufw allow 3000:3010/tcp             # Rango de puertos
ufw deny 23/tcp                     # Deniega telnet
ufw reject 25/tcp                   # Rechaza SMTP (con notificacion)
ufw allow from 192.168.1.0/24       # Permite toda una subred
ufw allow from 192.168.1.0/24 to any port 22  # SSH solo desde LAN
ufw allow out 53/udp                # Permite DNS saliente
ufw delete allow 80/tcp             # Elimina una regla
ufw delete 3                        # Elimina por numero de regla
```

> `ufw` registra los paquetes denegados si se activa el logging: `ufw logging on`. Los logs se escriben en `/var/log/ufw.log` y son utiles para depurar reglas demasiado restrictivas.

## firewalld: firewall dinamico por zonas

`firewalld` es el gestor de firewall por defecto en RHEL, CentOS, Fedora y derivados. Introduce el concepto de **zonas**, que agrupan reglas segun el nivel de confianza de la red:

- `drop`: el minimo de confianza, descarta todo el trafico entrante.
- `block`: similar a drop, pero rechaza con icmp-host-prohibited.
- `public`: para redes publicas (por defecto). Se permiten ciertas conexiones selectivas.
- `internal` / `trusted`: para redes internas de confianza.

```bash
firewall-cmd --state                   # Estado del firewall
firewall-cmd --get-default-zone        # Zona por defecto
firewall-cmd --get-active-zones        # Zonas activas por interfaz
firewall-cmd --zone=public --list-all  # Reglas de una zona
firewall-cmd --zone=public --add-service=http   # Permite HTTP
firewall-cmd --zone=public --add-service=https --permanent  # Persistente
firewall-cmd --zone=public --add-port=8080/tcp              # Puerto 8080
firewall-cmd --zone=public --add-port=3000-3010/tcp         # Rango
firewall-cmd --zone=public --remove-service=http            # Elimina servicio
firewall-cmd --reload                   # Recarga configuracion
firewall-cmd --runtime-to-permanent     # Guarda reglas actuales
```

La separacion entre configuracion runtime (volatil) y permanente es una de las ventajas de `firewalld`: se pueden aplicar cambios en caliente sin miedo a perder conectividad, y solo cuando se confirma que todo funciona se persisten con `--runtime-to-permanent`.

> La regla de oro con firewalld: anadir `--permanent` solo despues de verificar que el cambio runtime no bloquea la conexion actual. Si se pierde el acceso, un reinicio del servicio restaura la configuracion persistente anterior.

## iptables y nftables: el control directo del kernel

Por debajo de `ufw` y `firewalld` se encuentra el sistema de filtrado del kernel. `iptables` fue la interfaz clasica durante anos, basada en el framework Netfilter. `nftables` es su sucesor, integrado en el kernel desde la version 3.13, que unifica el tratamiento de IPv4, IPv6, ARP y bridge en una sola sintaxis.

### iptables

```bash
# Politicas por defecto
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Permitir trafico en loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Permitir conexiones establecidas y relacionadas
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Abrir puertos especificos
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Limitar por IP origen
iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT

# Denegar trafico explicito
iptables -A INPUT -p tcp --dport 23 -j DROP

# Listar reglas
iptables -L -n -v          # Lista detallada con contadores
iptables -S                # Lista como comandos (exportable)

# Gestion de cadenas
iptables -N MI_CADENA      # Crear cadena personalizada
iptables -A INPUT -j MI_CADENA  # Enlazar cadena personalizada
iptables -F                # Limpiar todas las reglas
iptables -X                # Eliminar cadenas personalizadas
```

### nftables

`nftables` simplifica la sintaxis y unifica la gestion:

```bash
# Tabla y cadena base
nft add table inet filtro
nft add chain inet filtro input { type filter hook input priority 0 \; policy drop \; }
nft add chain inet filtro output { type filter hook output priority 0 \; policy accept \; }

# Reglas
nft add rule inet filtro input iif lo accept
nft add rule inet filtro input ct state established,related accept
nft add rule inet filtro input tcp dport 22 accept
nft add rule inet filtro input tcp dport { 80, 443 } accept
nft add rule inet filtro input ip saddr 192.168.1.0/24 tcp dport 22 accept

# Listar reglas
nft list ruleset

# Guardar y restaurar
nft list ruleset > /etc/nftables.conf
nft -f /etc/nftables.conf
```

La principal ventaja de `nftables` sobre `iptables` es que elimina duplicidades: una sola herramienta para IPv4 e IPv6, reglas atomicas (se aplican todas o ninguna) y mejor rendimiento al evaluar conjuntos de datos en el kernel en lugar de regla por regla.

> `ufw` y `firewalld` son recomendables para administradores que necesitan un firewall funcional sin profundizar en Netfilter. `nftables` (o `iptables`) es necesario cuando se requieren reglas complejas, como balanceo de carga, NAT, marcado de paquetes, logging selectivo o limitacion de ancho de banda.

