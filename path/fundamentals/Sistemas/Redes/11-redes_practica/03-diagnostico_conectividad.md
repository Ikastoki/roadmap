# Diagnostico de conectividad

El diagnostico de conectividad es el proceso sistematico para identificar y resolver problemas de red. Cuando un usuario reporta "no tengo internet", el administrador debe seguir un metodo que le permita aislar la causa del fallo, desde el nivel fisico (cable desconectado) hasta el nivel de aplicacion (DNS o firewall).

El diagnostico efectivo se basa en trabajar de abajo hacia arriba: primero verificar que el hardware funciona, luego la capa de enlace, luego la IP, luego el enrutamiento, luego DNS y finalmente la aplicacion.

## Metodologia de diagnostico

1. **Verificar la interfaz:** comprobar que la interfaz de red esta activa y tiene una direccion IP.
2. **Verificar la conectividad local:** hacer ping al gateway local.
3. **Verificar la conectividad externa:** hacer ping a una IP publica (8.8.8.8).
4. **Verificar DNS:** resolver un nombre de dominio.
5. **Verificar el servicio:** probar el servicio especifico (HTTP, SSH, etc.).

## Herramientas de diagnostico

### ping

`ping` es la herramienta mas basica de diagnostico. Verifica si un host es alcanzable y mide la latencia.

```bash
# Verificar conectividad basica
ping -c 4 8.8.8.8

# Si falla: problema de IP, enrutamiento o cable
# Si funciona: el problema es probablemente DNS

# Ping a multiples saltos
ping -c 4 192.168.1.1    # Gateway local
ping -c 4 8.8.8.8        # DNS Google (verifica internet)
ping -c 4 google.com     # Verifica DNS
```

### traceroute y mtr

`traceroute` muestra el camino que siguen los paquetes hacia un destino. `mtr` combina traceroute con ping en tiempo real.

```bash
# Ruta completa
traceroute google.com

# MTR: monitoreo continuo
mtr google.com
mtr -r -c 10 google.com   # Reporte con 10 ciclos

# Buscar donde se pierden paquetes
traceroute -n 8.8.8.8
```

### ip route show y ip addr show

Estos comandos muestran el estado actual de la configuracion de red.

```bash
# Ver direcciones IP
ip addr show

# Ver rutas
ip route show

# Verificar que la interfaz tiene IP y UP
ip -br addr show
```

### ss -tuln

`ss` muestra los puertos en escucha, lo que permite verificar si un servicio esta activo y escuchando.

```bash
# Ver puertos en escucha
ss -tuln

# Verificar que un servicio esta escuchando
ss -tulnp | grep :80      # Nginx/Apache
ss -tulnp | grep :22      # SSH
ss -tulnp | grep :3306    # MySQL
```

## Resolucion DNS

Los problemas de DNS son extremadamente comunes: el usuario no puede acceder a una pagina web pero puede hacer ping a su IP directamente.

```bash
# Probar resolucion DNS
dig google.com
nslookup google.com

# Usar un servidor DNS especifico
dig @8.8.8.8 google.com

# Ver el contenido de /etc/resolv.conf
cat /etc/resolv.conf

# Verificar systemd-resolved
resolvectl status
resolvectl query google.com
```

## Pruebas de servicios

```bash
# Servidor web local
python3 -m http.server 8000

# Probar conectividad TCP a un puerto
nc -zv servidor 80
nc -zv servidor 443

# Probar SSH
ssh -v usuario@servidor

# Ver logs del sistema
journalctl -u nginx
sudo tail -f /var/log/syslog
```

## Flujo de diagnostico tipico

```
"No tengo internet"
│
├─¿Puedo ping al gateway (192.168.1.1)?
│  ├─NO → Verificar cable, interfaz, IP
│  └─SI ↓
│
├─¿Puedo ping a 8.8.8.8?
│  ├─NO → Problema de enrutamiento o ISP
│  └─SI ↓
│
├─¿Puedo resolver DNS (dig google.com)?
│  ├─NO → Problema DNS (/etc/resolv.conf)
│  └─SI ↓
│
└─¿Puedo acceder al servicio (curl https://ejemplo.com)?
   ├─NO → Firewall, servicio caido, proxy
   └─SI → El problema esta en la aplicacion
```

> El diagnostico de red es un proceso logico, no un guessing game. Cada herramienta responde a una pregunta concreta sobre una capa especifica de la red. Seguir un metodo sistemático de abajo hacia arriba garantiza que se identifique la causa raiz del problema sin perder tiempo explorando caminos incorrectos.

