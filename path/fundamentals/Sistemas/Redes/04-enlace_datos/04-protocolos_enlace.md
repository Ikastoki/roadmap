# Protocolos de capa de enlace: Ethernet, PPP y HDLC

Los protocolos de la capa de enlace de datos definen como se estructuran las tramas, como se accede al medio y como se detectan y corrigen errores en la transmision. Los tres protocolos principales de esta capa son Ethernet (dominante en LAN), PPP (enlaces punto a punto) y HDLC (enlaces seriales).

Cada protocolo esta disenado para un caso de uso especifico: Ethernet para redes de area local compartidas, PPP para enlaces punto a punto simples y configurables, y HDLC para enlaces seriales de alta velocidad.

## Ethernet

Ethernet es el protocolo de enlace de datos mas extendido del mundo. Funciona en las capas 1 y 2 del modelo OSI y es la base de todas las redes LAN cableadas modernas.

**Formato de trama Ethernet II (DIX):**

```
Preambulo (8B) | MAC Dest (6B) | MAC Orig (6B) | Tipo (2B) | Datos (46-1500B) | FCS (4B)
```

El campo *Tipo* identifica el protocolo encapsulado: 0x0800 para IPv4, 0x0806 para ARP, 0x86DD para IPv6. Si el campo contiene un valor menor a 1536, se interpreta como longitud (formato IEEE 802.3).

**Velocidades y estandares:** Ethernet ha evolucionado desde 10 Mbps (10BASE-T) hasta 400 Gbps (400GBASE-FR8), manteniendo compatibilidad con los formatos de tramaBasicos. La trama Ethernet II es compatible retroactivamente con todas las versiones.

**Funcionamiento basico:** Ethernet es un protocolo de capa de enlace que opera en topologia logica de bus (historicamente) o estrella (con switches). Utiliza direcciones MAC para la comunicacion en la red local y no proporciona fiabilidad end-to-end (delega esta funcion a TCP en capa 4).

## PPP (Point-to-Point Protocol)

PPP es un protocolo de enlace punto a punto disenado para establecer conexiones directas entre dos nodos. Se utilza en enlaces seriales, conexiones de banda ancha (PPPoE) y VPNs simples.

**Caracteristicas principales:**

- Soporta multiplexion de protocolos a traves del campo *Protocol*.
- Incluye mecanismos de autenticacion (PAP, CHAP).
- Negocia opciones de configuracion al inicio de la conexion (LCP - Link Control Protocol).
- Soporta compresion de cabeceras.

**Autenticacion en PPP:**

- **PAP (*Password Authentication Protocol*):** autenticacion simple por nombre de usuario y contrasena en texto plano. No es seguro.
- **CHAP (*Challenge-Handshake Authentication Protocol*):** autenticacion mas segura que utiliza un desafio (*challenge*) del servidor, una respuesta hasheada del cliente y un verificacion del servidor. La contrasena nunca se transmite en texto plano.

**Formato de trama PPP:**

```
Flag (7E) | Address (FF) | Control (03) | Protocol (2B) | Datos | FCS (2B) | Flag (7E)
```

PPP se ha utilizado historicamente para conexiones de acceso a internet por linea telefonica y ADSL (PPPoE: PPP over Ethernet). Aunque su uso ha disminuido con la llegada de fiber y 4G/5G, sigue siendo relevante en enlaces punto a punto dedicados.

## HDLC (High-Level Data Link Control)

HDLC es un protocolo de enlace punto a punto derivado del protocolo SDLC de IBM. Es un protocolo orientado a bits, no a bytes, lo que lo hace mas flexible que PPP para diferentes tipos de carga util.

**Caracteristicas:**

- Soporta modos de operacion: *Normal Response Mode* (NRM, maestro-esclavo), *Asynchronous Balanced Mode* (ABM, entre iguales).
- Tipos de tramas: *Information* (datos), *Supervisory* (control de flujo y errores), *Unnumbered* (gestion de enlace).
- Numeracion de tramas con contadores modulo 8 o modulo 128.

HDLC se utiliza en enlaces seriales dedicados y como base para otros protocolos: PPP esta basado en HDLC con campos adicionales de negociacion y autenticacion.

## Cuandos se usa cada uno

| Protocolo | Caso de uso | Ventaja |
|---|---|---|
| Ethernet | LAN cableada, centros de datos | Alta velocidad, madurez, bajo coste |
| PPP | Enlaces punto a punto, PPPoE | Autenticacion, configuracion flexible |
| HDLC | Enlaces seriales, backbones WAN | Eficiente, bajo overhead |

> En la practica, Ethernet domina completamente las redes locales, PPP se ha reducido a escenarios legacy (ADSL) y enlaces punto a punto, y HDLC es poco conocido fuera del mundo de las telecomunicaciones. Sin embargo, comprender los tres protocolos proporciona una vision completa de como la capa de enlace se adapta a diferentes requisitos de comunicacion.

