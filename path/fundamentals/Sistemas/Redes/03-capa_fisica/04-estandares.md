# Estandares relevantes

Los estandares de redes son especificaciones tecnicas documentadas por organos de estandarizacion que garantizan la interoperabilidad entre dispositivos de diferentes fabricantes. Sin estandares, cada fabricante crearia su propia tecnologia incompatible, lo que haria imposible la comunicacion entre dispositivos de distintos proveedores.

Los estandares mas importantes en redes provienen de tres organos principales: IEEE (Institute of Electrical and Electronics Engineers) para estandares de hardware y protocolos de bajo nivel; IETF (Internet Engineering Task Force) para protocolos de internet; e ITU-T (International Telecommunication Union) para telecomunicaciones.

## IEEE 802.x

El IEEE 802 es un conjunto de estandares para redes de area local y metropolitana. Los subestandares mas relevantes son:

**IEEE 802.3 (Ethernet):** el estandar para redes cableadas. Define las capas fisica y de enlace de datos para Ethernet, incluyendo velocidades, tipos de cable, formatos de trama y mecanismos de control de acceso al medio.

**IEEE 802.11 (Wi-Fi):** el estandar para redes inalambricas LAN. Define las capas fisica y de enlace para comunicacion inalambrica, incluyendo frecuencias, velocidades, mecanismos de seguridad y gestion de movilidad.

**IEEE 802.15 (Bluetooth/WPAN):** el estandar para redes personales inalambricas (*Wireless Personal Area Network*). Incluye Bluetooth (802.15.1) y Zigbee (802.15.4), utilizados en dispositivos IoT, auriculares y sensores.

## Estandares de Ethernet

Ethernet ha evolucionado a lo largo de decadas, multiplicando su velocidad de 10 Mbps a 400 Gbps:

| Estandar | Velocidad | Medio | Distancia max. |
|---|---|---|---|
| 10BASE-T | 10 Mbps | Par trenzado Cat3+ | 100 m |
| 100BASE-TX | 100 Mbps | Par trenzado Cat5+ | 100 m |
| 1000BASE-T | 1 Gbps | Par trenzado Cat5e+ | 100 m |
| 10GBASE-T | 10 Gbps | Par trenzado Cat6a+ | 100 m |
| 25GBASE-T | 25 Gbps | Par trenzado Cat8 | 30 m |
| 1000BASE-SX | 1 Gbps | Fibra multimodo | 550 m |
| 1000BASE-LX | 1 Gbps | Fibra monomodo | 5 km |
| 10GBASE-SR | 10 Gbps | Fibra multimodo | 400 m |
| 10GBASE-LR | 10 Gbps | Fibra monomodo | 10 km |
| 40GBASE-SR4 | 40 Gbps | Fibra multimodo | 100 m |
| 100GBASE-SR10 | 100 Gbps | Fibra multimodo | 100 m |

Los nombres siguen una convencion: velocidad + BASE (baseband) + tipo de medio (T = twisted pair, SX/LX = fibra, SR/LR = fibra corta/larga distancia).

## Otros estandares importantes

**ANSI/TIA-568:** estandar para cableado estructurado en edificios comerciales. Define los requisitos de rendimiento, instalacion y prueba de cables y conectores. Es la referencia para instalaciones profesionales de red.

**ITU-T:** organo de normalizacion de telecomunicaciones de la ONU. Define estandares como G.984 (GPON para fibra optica hasta el hogar), G.711 (codificacion de voz digital) y H.264/H.265 (compresion de video).

**ISO/IEC 11801:** estandar internacional para infraestructura de cableado estructurado. Define las clases de rendimiento (Class C, D, E, F) que equivalen a las categorias TIA (Cat5e, Cat6, Cat6a, Cat7).

**IEEE 802.1Q:** estandar para VLANs (*Virtual LAN*), que permite segmentar logicamente una red fisica.

**IEEE 802.1D/1w/1s:** estandares para STP (*Spanning Tree Protocol*) y sus variantes rapidas, que previenen bucles en topologias con redundancia.

> Los estandares son documentos vivos que se actualizan para incorporar nuevas tecnologias. Un profesional de redes debe mantenerse informado sobre las revisiones de los estandares relevantes, ya que una especificacion obsoleta puede causar problemas de compatibilidad y rendimiento en instalaciones nuevas.

