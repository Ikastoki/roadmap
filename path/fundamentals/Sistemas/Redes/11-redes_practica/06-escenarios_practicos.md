# Escenarios practicos

La teoria de las redes cobra vida en los escenarios practicos donde se aplica lo aprendido para resolver problemas reales. Estos escenarios cubren situaciones comunes que un administrador de sistemas o ingeniero de red puede encontrar: desde compartir una conexion a internet hasta configurar un servidor DHCP local.

Cada escenario combina multiples conceptos de las capas anteriores: configuracion de IP, enrutamiento, NAT, firewall y servicios de red.

## Compartir internet (IP forwarding + NAT)

Permitir que dispositivos de una red local accedan a internet a traves de un servidor Linux que actua como router:

```bash
# 1. Activar IP forwarding
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 2. Configurar NAT con iptables
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT

# 3. Guardar reglas
sudo apt install iptables-persistent
sudo netfilter-persistent save

# Verificar
sudo iptables -t nat -L -v
```

## Crear una red entre dos maquinas virtuales

Para practicar redes sin hardware real, se pueden crear dos VMs conectadas por una red privada:

```bash
# En VirtualBox: crear red interna
# VM1: enp0s3 (NAT para internet), enp0s8 (interna "red_practica")
# VM2: enp0s3 (NAT para internet), enp0s8 (interna "red_practica")

# VM1: asignar IP en la red interna
sudo ip addr add 192.168.100.1/24 dev enp0s8
sudo ip link set enp0s8 up

# VM2: asignar IP en la misma red
sudo ip addr add 192.168.100.2/24 dev enp0s8
sudo ip link set enp0s8 up

# Verificar conectividad
# VM1: ping 192.168.100.2
# VM2: ping 192.168.100.1
```

## Configurar un servidor DHCP local

Con `isc-dhcp-server`, se puede crear un servidor DHCP que asigne direcciones IP a los dispositivos de la red:

```bash
# Instalar
sudo apt install isc-dhcp-server

# Configurar interfaz
# /etc/default/isc-dhcp-server
INTERFACESv4="enp0s8"

# Configurar el servidor DHCP
# /etc/dhcp/dhcpd.conf
subnet 192.168.100.0 netmask 255.255.255.0 {
    range 192.168.100.10 192.168.100.100;
    option routers 192.168.100.1;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    option domain-name "practica.local";
    default-lease-time 600;
    max-lease-time 7200;
}

# Iniciar servicio
sudo systemctl start isc-dhcp-server
sudo systemctl enable isc-dhcp-server

# Ver concesiones activas
sudo cat /var/lib/dhcp/dhcpd.leases
```

## Analisis de puertos y trafico

```bash
# Escaneo con nmap
nmap localhost                    # Puertos abiertos en localhost
nmap -sV 192.168.1.10            # Version de servicios
nmap -p 22,80,443 192.168.1.10   # Puertos especificos
nmap -O 192.168.1.10             # Detectar SO

# Captura de trafico con tcpdump
sudo tcpdump -i enp0s3 -n port 80
sudo tcpdump -i enp0s3 -w trafico.pcap
sudo tcpdump -r trafico.pcap | head -20

# Analizar con Wireshark (grafico)
wireshark trafico.pcap
```

## Gestion de servicios de red

```bash
# Reiniciar la pila de red
sudo systemctl restart networking
sudo systemctl restart NetworkManager

# Reiniciar una interfaz especifica
sudo ip link set enp0s3 down
sudo ip link set enp0s3 up

# Ver logs de red
journalctl -u NetworkManager
journalctl -u networking

# Ver estado general de la red
ip -br addr show
ip route show
ss -tuln
```

> La practica es la mejor forma de aprender redes. Configurar un servidor DHCP, crear una red entre VMs, capturar trafico con tcpdump y diagnosticar problemas con ping/traceroute son exercicios que consolidan el conocimiento teorico y preparan para situaciones reales en entornos de produccion.

