# Configuracion de red

Un servidor sin configuracion de red funcional es un sistema operativo aislado, incapaz de recibir peticiones de clientes o comunicarse con otros servicios. La red es el primer servicio que debe configurarse correctamente, y su configuracion determina si el servidor es accesible, si responde a las solicitudes esperadas y si puede mantener comunicacion fiable con el resto de la infraestructura. A diferencia de un escritorio donde DHCP asigna automaticamente una direccion IP y el usuario no necesita pensar en ello, un servidor requiere una configuracion explicita, predecible y resistente a cambios inesperados.

## Direccion IP estatica vs dinamica (DHCP)

### Direccion IP estatica

Una direccion IP estatica es una configuracion donde el servidor tiene una direccion IP asignada permanentemente que no cambia a menos que un administrador la modifique. Esta es la configuracion estandar para servidores en entornos locales porque garantiza que los clientes, los balances de carga y los registros DNS siempre encuentran el servidor en la misma direccion.

La necesidad de IP estatica se origina en la naturaleza de los servicios que un servidor ejecuta. Un servidor web que cambia de IP periodicamente haria que los registros DNS apuntaran a una direccion que podria no corresponderselos, dejando la web inaccesible. Un servidor de bases de datos cuya IP cambia romperia la conexion de las aplicaciones que dependen de el. Un servidor NFS o Samba que cambia de IP causaria la desconexion de todos los clientes montando recursos compartidos.

### Direccion IP dinamica (DHCP)

En entornos de nube publica, los servidores virtualizados y los contenedores, DHCP es frecuentemente la norma. Los proveedores de nube gestionan las direcciones IP a traves de sus propios servidores DHCP y las asocian a las instancias durante su ciclo de vida. En estos contextos, la IP externa puede cambiar (especialmente en instancias efimeras o con elasticidad), y las aplicaciones estan disenadas para tolerar estos cambios o se basan en nombres DNS que se actualizan dinamicamente.

La decision entre estatica y dinamica no es ideologica: depende del contexto de despliegue. En un datacenter propio con servidores fisicos, estatica es casi siempre la respuesta. En la nube, la IP externa puede ser dinamica pero la IP interna de la VPC suele ser asignada por DHCP con reservas que la hacen efectivamente estatica.

## Configuracion de red en distribuciones basadas en Debian/Ubuntu: Netplan

Netplan es la herramienta de configuracion de red en Ubuntu Server desde la version 17.10. Utiliza archivos YAML en `/etc/netplan/` para describir la configuracion de red deseada y genera la configuracion para el backend subyacente (NetworkManager o systemd-networkd).

```yaml
# /etc/netplan/01-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens160:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

La separacion en multiples archivos YAML dentro de `/etc/netplan/` permite organizar configuraciones por proposito: un archivo para la interfaz principal, otro para VLANs, otro para bridges. Netplan aplica los cambios de forma atomica, lo que minimiza el riesgo de perder conectividad durante la reconfiguracion.

> Antes de aplicar cambios de red, verificar siempre la sintaxis con `sudo netplan try` que aplica la configuracion temporalmente y revierte automaticamente despues de 120 segundos si no se confirma. Esto evita bloquear el acceso al servidor por una mala configuracion.

## Configuracion de red en distribuciones RHEL/Rocky: NetworkManager

RHEL y sus derivadas utilizan NetworkManager como demonio principal de gestion de red. La herramienta `nmcli` permite gestionar conexiones desde la linea de comandos sin necesidad de editar archivos manualmente.

```bash
# Ver conexiones existentes
nmcli con show

# Configurar IP estatica
sudo nmcli con mod "ens160" \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4" \
  ipv4.method manual

# Aplicar cambios
sudo nmcli con up "ens160"
```

NetworkManager gestiona automaticamente conexiones WiFi, VPN y otros tipos de red, pero en servidores con conexiones cableadas estaticas, su complejidad adicional puede ser innecesaria. En Ubuntu Server, el backend por defecto es `networkd`, un componente de systemd mas ligero y orientado a servidores.

## Configuracion de hostname y resolucion de nombres

El nombre del servidor no solo es un identificador humano: es parte del sistema de resolucion de nombres DNS y de la configuracion de multiples servicios. Un servidor con un hostname inconsistente o mal configurado puede generar problemas con servicios de directorio (LDAP, Active Directory), con la emision de certificados TLS y con la identificacion en sistemas de monitoreo.

```bash
# Verificar hostname actual
hostname -f

# Configurar /etc/hosts para resolucion local
# 127.0.0.1   localhost
# 192.168.1.100  webprod01.miempresa.local  webprod01
```

El archivo `/etc/hosts` proporciona resolucion local antes de que DNS este disponible. Durante el arranque, servicios como `systemd-resolved` o el demonio DNS local necesitan resolver el propio hostname del servidor, y si `/etc/hosts` no contiene la entrada correspondiente, el arranque puede ser significativamente mas lento o incluso fallar.

## Interfaz de red: conceptualizacion

Los servidores fisicos suelen tener multiples interfaces de red: una para la red de administracion (acceso SSH, monitoreo), otra para la red de datos (trafico de clientes), y posiblemente una tercera para una red de almacenamiento (iSCSI, NFS). Separar estos flujos en interfaces distintas mejora la seguridad (la red de administracion no debe ser accesible desde la red de datos), el rendimiento (el trafico de almacenamiento no compite con el de clientes) y el monitoreo (se puede analizar cada flujo independientemente).

En entornos virtualizados, las interfaces de red virtuales (veth, tap, bridge) replican este concepto. Docker, por ejemplo, crea un bridge virtual (`docker0`) y conecta cada contenedor a el mediante interfaces virtuales.

## Pruebas de conectividad

Despues de configurar la red, verificar la conectividad es obligatorio antes de considerar el servidor operativo. Las pruebas deben ser progresivas: primero conectividad local, luego con el gateway, y finalmente con destinos externos.

```bash
# Verificar interfaces activas
ip addr show

# Probar conectividad con el gateway
ping -c 4 192.168.1.1

# Probar resolucion DNS
ping -c 4 google.com

# Verificar rutas
ip route show
```

Si la conectividad falla, la depuracion sigue un orden logico: verificar que la interfaz esta activa (`ip link show`), que la IP esta asignada correctamente (`ip addr show`), que la ruta por defecto existe (`ip route show`) y que el gateway responde (`ping`). Este orden evita perder tiempo verificando DNS cuando el problema esta en la capa fisica o enlace.

