# Configuracion de red

La configuracion de red en Linux determina como el sistema se conecta a la red: si utiliza una direccion IP asignada automaticamente (DHCP) o manualmente (estatica), que servidores DNS utiliza y que puerta de enlace predeterminada tiene. La configuracion correcta es esencial para la conectividad y el rendimiento.

Linux ofrece multiples metodos de configuracion de red: herramientas en tiempo de ejecucion (ip, nmcli), archivos de configuracion estaticos y sistemas de gestion de red como NetworkManager y Netplan.

## IP estatica vs DHCP

**DHCP (*Dynamic Host Configuration Protocol*):** el servidor DHCP asigna automaticamente direccion IP, mascara, gateway y DNS al dispositivo. Es la configuracion predeterminada en la mayoria de sistemas y es ideal para dispositivos que no requieren una direccion fija.

**IP estatica:** el administrador configura manualmente la direccion IP, mascara, gateway y DNS. Es necesaria para servidores, dispositivos de red (routers, switches) y cualquier equipo que deba ser localizable por una IP constante.

```bash
# Verificar configuracion actual
ip addr show
ip route show
cat /etc/resolv.conf
```

## Netplan (Ubuntu 18.04+)

Netplan es el sistema de configuracion de red predeterminado en Ubuntu moderno. Utiliza archivos YAML en `/etc/netplan/` para definir la configuracion de red, que luego se genera para NetworkManager o systemd-networkd.

```yaml
# /etc/netplan/01-config.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses:
        - 192.168.56.10/24
      routes:
        - to: default
          via: 192.168.56.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
# Aplicar configuracion de Netplan
sudo netplan apply
sudo netplan try    # Aplica con timeout para revertir si hay problemas
```

## /etc/network/interfaces (sistemas antiguos)

En distribuciones antiguas (Debian sin systemd-networkd, etc.), la configuracion se define en `/etc/network/interfaces`:

```
# Interfaz dinamica
auto eth0
iface eth0 inet dhcp

# Interfaz estatica
auto eth1
iface eth1 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

```bash
# Reiniciar interfaz
sudo ifdown eth1 && sudo ifup eth1
```

## dhclient

`dhclient` es el cliente DHCP clasico de Linux. Solicita una configuracion IP al servidor DHCP y la aplica a la interfaz.

```bash
# Solicitar IP por DHCP
sudo dhclient eth0

# Liberar la IP y solicitar una nueva
sudo dhclient -r eth0
sudo dhclient eth0

# Ver informacion de la concesion DHCP
cat /var/lib/dhclient/dhclient.leases
```

## Configuracion de DNS

Los servidores DNS se configuran en `/etc/resolv.conf` o a traves de NetworkManager/systemd-resolved.

```
# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
search ejemplo.com
```

```bash
# Ver resolvers actuales
resolvectl status
cat /etc/resolv.conf

# Probar resolucion DNS
dig @8.8.8.8 google.com
```

## Configuracion con NetworkManager

NetworkManager es el demonio de gestion de red predeterminado en la mayoria de distribuciones de escritorio y servidor modernas. Gestionar la red con NetworkManager es ideal para entornos donde la conectividad cambia frecuentemente (portatiles, moviles).

```bash
# Ver interfaces gestionadas por NetworkManager
nmcli device status

# Ver propiedades de una conexion
nmcli connection show "Wired connection 1"

# Modificar una conexion
nmcli connection modify "Wired connection 1" \
  ipv4.addresses 10.0.1.50/24 \
  ipv4.method manual

# Activar/desactivar una interfaz
nmcli device disconnect enp0s3
nmcli device connect enp0s3
```

> La configuracion de red es una de las tareas mas comunes en la administracion de Linux. Netplan simplifica la configuracion declarativa en Ubuntu; NetworkManager gestiona la conectividad dinamica; y las herramientas `ip` y `nmcli` proporcionan control en tiempo de ejecucion. Combinar la configuracion estatica (para servidores) con DHCP (para clientes) es la estrategia mas habitual en entornos productivos.

