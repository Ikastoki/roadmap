# Concepto de red y tipos

Una red de computadoras es un conjunto de dispositivos interconectados que comparten recursos y se comunican entre si mediante protocolos y medios de transmision. Las redes son la base de la comunicacion moderna: desde la conexin a internet en un hogar hasta las redes de centros de datos que sustentan la nube. Comprender los fundamentos de las redes es esencial para cualquier profesional de tecnologia, ya que la mayoria de los sistemas operan en entornos conectados.

Las redes permiten que computadoras que estan fisicamente separadas intercambien datos, compartan impresoras y almacenamiento, acedan a bases de datos remotas y se comuniquen en tiempo real. Sin las redes, la informatica moderna se redujera a computadoras aisladas sin posibilidad de colaboracion.

## Que es un nodo

Un **nodo** es cualquier dispositivo activo conectado a una red que es capaz de enviar, recibir o reenviar informacion. No todas las computadoras son nodos: un simple repetidor de seal tambien es un nodo, mientras que un cable sin inteligencia no lo es. Tipicamente, los nodos son computadoras, servidores, routers, switches, impresoras de red o dispositivos moviles.

Cada nodo tiene una identificacion unica en la red, bien sea una direccion MAC (capa 2) o una direccion IP (capa 3), que permite a los demas nodos dirigir la informacion hacia el o desde el.

## Que es un protocolo

Un **protocolo** es un conjunto de reglas y convenciones que definen como se intercambian datos entre dos o mas dispositivos en una red. Los protocolos especifican el formato de los mensajes, el orden de las operaciones, los mecanismos de error y la sincronizacion entre emisor y receptor.

Un protocolo es comparable a un idioma comun: sin el, dos personas que hablan idiomas diferentes no pueden comunicarse. Sin TCP/IP, dos computadoras no pueden intercambiar datos de forma fiable.

## Evolucion de las redes

Las redes han evolucionado desde sistemas aislados hasta la conectividad global que conocemos hoy:

- **Decada de 1960:** ARPANET, la red precursora de internet, conectaba universidades y centros de investigacion en EE.UU. Utilizaba conmutacion de paquetes en lugar de conmutacion de circuitos.
- **Decada de 1970:** desarrollo de TCP/IP como protocolo de comunicacion.
- **Decada de 1980:** estandarizacion de Ethernet y expansion de las LAN. Aparicion de las primeras redes inalambricas.
- **Decada de 1990:** World Wide Web y comercializacion de internet. Crecimiento explosivo de las conexiones.
- **Decada de 2000:** redes inalambricas masivas (Wi-Fi), telefonos inteligentes y nube.
- **Decada de 2010-2020:** IoT, 5G, edge computing y redes de alta velocidad.

## Objetivos de las redes

Los objetivos principales de una red son:

- **Compartir recursos:** impresoras, almacenamiento, conexiones a internet y aplicaciones pueden ser compartidos entre multiples usuarios, reduciendo costes y mejorando la eficiencia.
- **Comunicacion:** correo electronico, mensajeria instantanea, videollamadas y colaboracion en tiempo real son posibles gracias a las redes.
- **Redundancia y disponibilidad:** la replicacion de datos y la redundancia de caminos garantizan que el sistema permanezca operativo incluso ante fallos de hardware.
- **Centralizacion de la gestion:** las redes permiten administrar y monitorizar multiples dispositivos desde un punto centralizado.

## Tipos de redes por alcance geografico

**PAN (*Personal Area Network*):** red de corto alcance que conecta dispositivos personales. Bluetooth y USB son los protocolos tipicos de una PAN. Un telefono conectado a unas auriculares bluetooth forma una PAN.

**LAN (*Local Area Network*):** red que cubre un area geografica limitada, como una oficina, un edificio o un campus. Las LAN son las redes mas comunes: la red de tu hogar, la red de una empresa. Ethernet (cableada) y Wi-Fi (inalambrica) son los protocolos predominantes en LAN.

**MAN (*Metropolitan Area Network*):** red que cubre un area metropolitana, como una ciudad. Las MAN suelen ser operadas por proveedores de servicios de telecomunicaciones y utilizan tecnologias como fibra optica y microondas.

**WAN (*Wide Area Network*):** red que cubre un area geografica amplia, como un pais o un continente. Internet es la WAN mas grande del mundo. Las WAN conectan LAN distantes a traves de enlaces dedicados, conmutacion de paquetes o tecnologias de transporte.

**WLAN (*Wireless Local Area Network*):** variante inalambrica de la LAN, basada en el estandar IEEE 802.11 (Wi-Fi). Es la forma mas comun de conectar dispositivos moviles en entornos domesticos y empresariales.

> La clasificacion de las redes por alcance geografico es una simplificacion util pero no absoluta. Una red corporativa pode utilizar fibra optica monomodo (tipica de WAN) para conectar edificios en la misma ciudad, o una VPN sobre internet para crear una WAN logica a traves de una infraestructura publica. Lo importante es entender que la tecnologia de transporte y el alcance son conceptos independientes.

