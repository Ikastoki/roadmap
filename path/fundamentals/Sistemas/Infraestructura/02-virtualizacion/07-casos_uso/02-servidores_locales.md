# Servidores locales (homelab)

Un homelab es un laboratorio de TI personal basado en servidores fisicos que ejecutan multiples servicios en VMs o contenedores. Es la aplicacion practica de la virtualizacion para aprendizaje, experimentacion y ejecucion de servicios personales. Con un solo servidor fisico y un hipervisor como Proxmox, se puede crear un ecosistema completo de servicios que normalmente requeririan multiples dispositivos.

## Arquitectura tipica de un homelab

```
Servidor fisico (Proxmox VE)
├── VM 1: OPNsense (firewall/router)
│   ├── WAN: conexion a Internet
│   └── LAN: red interna del homelab
├── VM 2: Ubuntu Server (servidor web)
│   ├── Nginx (proxy inverso)
│   ├── Docker (contenedores)
│   └── Portainer (gestion de contenedores)
├── VM 3: Debian (base de datos)
│   ├── PostgreSQL
│   └── Redis
├── VM 4: Ubuntu Desktop (escritorio remoto)
│   ├── XFCE (entorno grafico ligero)
│   └── Guacamole (acceso via navegador)
├── LXC 1: Pi-hole (DNS bloqueador de anuncios)
├── LXC 2: Home Assistant (automatizacion del hogar)
└── LXC 3: Gitea (servidor Git privado)
```

## Servicios comunes en un homelab

**Infraestructura de red:**
- **OPNsense/pfSense:** firewall y router que reemplaza al router del ISP.
- **Pi-hole:** servidor DNS que bloquea anuncios y tracking a nivel de red.
- **WireGuard:** VPN para acceder al homelab desde cualquier lugar.

**Servidores de medios:**
- **Jellyfin/Plex:** servidor de videos, musica y fotos.
- **Sonarr/Radarr:** gestion automatica de series y peliculas.
- **Transmission:** cliente de descargas.

**Automatizacion y monitoreo:**
- **Home Assistant:** automatizacion del hogar (luces, sensores, climatizacion).
- **Grafana + Prometheus:** monitoreo de servidores y redes.
- **Uptime Kuma:** monitor de disponibilidad de servicios.

**Almacenamiento:**
- **Nextcloud:** almacenamiento y sincronizacion de archivos (alternativa a Google Drive).
- **Syncthing:** sincronizacion P2P de archivos entre dispositivos.
- **MinIO:** almacenamiento de objetos compatible con S3.

**Desarrollo:**
- **Gitea/Forgejo:** servidor Git privado.
- **Drone CI:** servidor de integracion continua.
- **Portainer:** gestion grafica de contenedores Docker.

## Configuracion basica del servidor

```bash
# Proxmox: instalar desde ISO
# 1. Descargar ISO de Proxmox VE
# 2. Crear USB booteable
# 3. Arrancar y seguir asistente
# 4. Acceder a https://ip-servidor:8006

# Crear primera VM
qm create 100 --name ubuntu-web --memory 4096 --cores 2 \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci \
  --scsi local-lvm:32 --ide2 local:iso/ubuntu-22.04.iso,media=cdrom \
  --boot order=scsi0;ide2

# Iniciar VM
qm start 100
```

## Virtualizacion de red doméstica

```bash
# Crear bridge de red para VMs
# En Proxmox: /etc/network/interfaces
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.2/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

## Consideraciones

- **Consumo electrico:** un servidor 24/7 consume entre 50-150W. Calcular el costo anual.
- **Ruido:** los servidores con ventiladores pueden ser ruidosos. Considerar hardware silencioso.
- **Calor:** un servidor genera calor significativo. Asegurar ventilacion adecuada.
- **Respaldo:** siempre tener backups fuera del homelab (nube, disco externo).

> Un homelab es la mejor forma de aprender infraestructura de TI sin riesgo. Puedes experimentar con firewalls, bases de datos, contenedores y redes virtuales, todo en un solo servidor que cabe en un armario. Los conocimientos adquiridos son directamente aplicables a entornos empresariales.

