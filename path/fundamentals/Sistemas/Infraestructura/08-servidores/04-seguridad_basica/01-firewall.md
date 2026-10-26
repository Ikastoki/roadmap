# Firewall

Un servidor expuesto a una red —ya sea una red corporativa o internet— recibe constantemente paquetes de datos solicitados y no solicitados. Sin un mecanismo de filtrado, cualquier servicio mal configurado o vulnerabilidad en uno de los puertos abiertos puede convertirse en un punto de entrada para un atacante. El firewall actua como la primera linea de defensa al decidir, paquete a paquete, que trafico debe permitirse y cual debe descartarse antes de que llegue a los servicios del sistema operativo.

La filosofia mas extendida en seguridad perimetral es **default-deny**: el firewall bloquea todo el trafico por defecto y solo abre explicitamente los puertos y protocolos necesarios. Esto reduce drasticamente la superficie de ataque. Un servidor web solo necesita puertos 80 y 443, no 22 (SSH), 3306 (MySQL) o cualquier otro servicio. Si un atacante escanea el servidor y no encuentra puertos abiertos, simplemente pasa de largo. En cambio, con una politica default-allow, el administrador debe recordar cerrar cada servicio individualmente, y uno solo olvidado puede comprometer todo el sistema.

## UFW (Uncomplicated Firewall)

UFW es la herramienta predeterminada en distribuciones basadas en Debian y Ubuntu. Su diseno prioriza la simplicidad: oculta la complejidad de iptables/nftables detras de una CLI intuitiva. La idea es que un administrador no necesite entender las cadenas de filtrado del kernel para configurar reglas basicas.

La forma mas segura de empezar es activar UFW con la politica default-deny para entrantes, permitiendo todo el trafico saliente:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw enable
```

Con esta configuracion basica, el servidor bloquea todo trafico entrante no solicitado. Las reglas se añaden para abrir puertos especificos segun los servicios que se ejecuten:

```bash
ufw allow 22/tcp    # SSH
ufw allow 80/tcp    # HTTP
ufw allow 443/tcp   # HTTPS
```

UFW tambien permite filtrar por subred, por direccion IP origen, o por numero de puerto con rangos. Las reglas se evaluan en orden numerico y la primera coincidencia determina si el paquete se acepta o rechaza. Para inspeccionar el estado actual se utiliza `ufw status verbose`, que muestra todas las reglas activas y la politica por defecto.

> UFW no es mas que un frontend para iptables o nftables. Detras de cada comando `ufw allow` se generan cadenas de filtrado en el kernel de Linux. Comprender esto ayuda a diagnosticar problemas cuando las reglas no se comportan como se esperaria.

## firewalld (RHEL/Fedora)

firewalld adopta un enfoque distinto al de UFW: organiza las reglas en **zonas**, cada una con un nivel de confianza asociado. Por ejemplo, la zona `public` es la mas restrictiva (apta para servidores en internet), mientras que `trusted` acepta todo el trafico. Cada interfaz de red se asigna a una zona, lo que permite configurar politicas distintas segun la red a la que este conectado el servidor (por ejemplo, una zona diferente para la interfaz corporativa que para la de produccion).

Una zona se configura añadiendo servicios o puertos que se desean abrir:

```bash
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-port=22/tcp --permanent
firewall-cmd --reload
```

El concepto de zona es especialmente valioso en servidores multihomed (con multiples interfaces de red). Un servidor con una tarjeta de red para la DMZ y otra para la red interna puede asignar la zona `public` a la primera y `internal` a la segunda, manteniendo politicas de trafico independientes sin necesidad de reglas iptables complejas.

> firewalld gestiona las reglas de nftables (o iptables en versiones anteriores) en tiempo real. Los cambios `--permanent` se escriben a disco pero solo se aplican tras un `--reload`, lo que permite ensayar configuraciones sin afectar al trafico activo.

## Por que el firewall es la primera linea de defensa

Sin firewall, un servidor esta completamente expuesto a cualquier escaneo de puertos o intento de conexion. Herramientas como `nmap` pueden enumerar todos los servicios activos en cuestion de segundos. Un firewall bien configurado no solo bloquea accesos no deseados, sino que tambien dificulta el reconocimiento: si un atacante no puede determinar que servicios estan ejecutandose, le resulta mucho mas dificil elegir un vector de ataque. Ademas, las herramientas de deteccion de intrusos como fail2ban dependen de que exista un firewall capaz de bloquear IPs en el kernel, no solo en nivel de aplicacion. Sin firewalld o UFW activo, fail2ban no tiene forma de impedir que un atacante siga conectandose.

