# Redes domesticas y pequenas oficinas (SOHO)

Las redes SOHO (*Small Office/Home Office*) son las redes mas comunes: la conexion a internet de un hogar, la red de una pequeña oficina o el estudio de un profesional independiente. Aunque su complejidad es menor que la de una red empresarial, los mismos conceptos se aplican a menor escala.

Comprender como funciona la red de tu hogar es el primer paso para configurarla correctamente, solucionar problemas de conectividad y mejorar su seguridad y rendimiento.

## Diagrama tipico de una red SOHO

```
Internet
   │
   ▼
[ISP: Modem (coaxial/fibra/ADSL)]
   │
   ▼
[Router/Wi-Fi combo]
   ├── Puerto 1 → PC de escritorio (cable Ethernet)
   ├── Puerto 2 → NAS (almacenamiento en red)
   ├── Puerto 3 → (libre)
   └── Wi-Fi → Portatil
              → Movil
              → Tablet
              → Smart TV
```

El dispositivo central es tipicamente un modem-router combo proporcionado por el ISP, que integra modem, router, switch de 4 puertos, AP Wi-Fi y firewall. Todos los dispositivos del hogar se conectan a este equipo, que gestiona la comunicacion con internet y entre dispositivos.

## NAT (Network Address Translation)

NAT es el mecanismo que permite que multiples dispositivos con direcciones IP privadas compartan una unica direccion IP publica para acceder a internet. Sin NAT, cada dispositivo necesitaria una IP publica, lo que agotaria rapidamente el espacio de direcciones IPv4.

```bash
# Ver la tabla NAT del router (si se tiene acceso)
sudo iptables -t nat -L -v

# NAT tipico en un router domestico:
# 192.168.1.10 (PC) → 203.0.113.5 (IP publica del ISP)
# 192.168.1.20 (Movil) → 203.0.113.5 (misma IP publica)
```

El router mantiene una tabla de traduccion que asocia cada IP privada + puerto con un puerto efimero en la IP publica. Cuando el servidor responde, el router traduce la respuesta de vuelta a la IP privada correspondiente.

## Puerta de enlace predeterminada

Todos los dispositivos de la red local utilizan la IP del router como puerta de enlace predeterminada (*default gateway*). Cuando un dispositivo quiere enviar datos a una IP fuera de su subred (por ejemplo, a un servidor de Google), envia los datos al router, que se encarga de enrutarlos a traves de internet.

```bash
# Ver la puerta de enlace en Linux
ip route show default
route -n

# Ver la puerta de enlace en Windows
ipconfig
```

## DHCP en el hogar

El router domestico ejecuta un servidor DHCP que asigna automaticamente direcciones IP, mascaras, DNS y puertas de enlace a todos los dispositivos que se conectan. El usuario no necesita configurar nada manualmente.

```bash
# Forzar renovacion de DHCP en Linux
sudo dhclient -r eth0
sudo dhclient eth0

# Ver la configuracion DHCP asignada
ip addr show
```

## Seguridad basica de la red domestica

- **Cambiar la contrasena por defecto del router:** los routers vienen con credenciales por defecto (admin/admin) que son publicamente conocidas.
- **Usar WPA3 o WPA2:** cifrar la red Wi-Fi con la version mas reciente del protocolo de seguridad.
- **Actualizar el firmware:** los fabricantes publican parches de seguridad que corrigen vulnerabilidades.
- **Desactivar WPS:** el boton WPS (*Wi-Fi Protected Setup*) es vulnerable a ataques de fuerza bruta.
- **Segmentar invitados:** crear una red Wi-Fi separada para invitados, aislada de los dispositivos principales.

> La red domestica es donde la mayoria de las personas tienen su primera experiencia con redes. Los conceptos que se aprenden aqui (NAT, DHCP, Wi-Fi, firewall) son los mismos que se aplican en redes empresariales, solo que a mayor escala y con mas complejidad. Entender tu red domestica es la base para comprender redes mas grandes.

