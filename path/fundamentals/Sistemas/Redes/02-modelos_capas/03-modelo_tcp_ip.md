# El modelo TCP/IP

El modelo TCP/IP es el marco de referencia que define la arquitectura de internet. A diferencia de OSI, que fue disefado como modelo teorico antes de su implementacion, TCP/IP se desarrollo a partir de los protocolos reales que componen la red de redes. Es el estandar de facto para las comunicaciones en redes modernas.

TCP/IP fue desarrollado originalmente por DARPA (Defense Advanced Research Projects Agency) en la decada de 1970 para ARPANET, y evoluciono hasta convertirse en la base de internet. Su diseno practico y orientado a la implementacion lo convirtio en el modelo dominante, mientras que OSI se mantuvo como referencia educativa.

## Estructura de 4 capas

El modelo TCP/IP se compone de 4 capas (algunas fuentes lo presentan como 5, separando la capa de acceso a la red):

**Capa de enlace (Network Interface):** corresponde a las capas 1 y 2 de OSI. Incluye la transmision fisica y el enlace de datos local. Ethernet, Wi-Fi y PPP operan en esta capa.

**Capa de red (Internet):** corresponde a la capa 3 de OSI. Maneja el direccionamiento y enrutamiento de paquetes entre redes. IPv4 e IPv6 son los protocolos principales de esta capa.

**Capa de transporte (Transport):** corresponde a la capa 4 de OSI. Proporciona comunicacion de extremo a extremo entre procesos. TCP y UDP son los protocolos fundamentales.

**Capa de aplicacion (Application):** combina las capas 5, 6 y 7 de OSI. Incluye HTTP, SMTP, FTP, DNS, SSH y todos los protocolos que las aplicaciones utilizan para comunicarse.

## Comparacion con OSI

```
      OSI              TCP/IP
┌──────────────┐   ┌──────────────┐
│  Aplicacion  │   │              │
├──────────────┤   │  Aplicacion  │
│ Presentacion │   │              │
├──────────────┤   │              │
│   Sesion     │   ├──────────────┤
├──────────────┤   │  Transporte  │
│  Transporte  │   ├──────────────┤
├──────────────┤   │     Red      │
│     Red      │   ├──────────────┤
├──────────────┤   │ Enlace datos │
│Enlace datos  │   │   Fisica     │
├──────────────┤   └──────────────┘
│   Fisica     │
└──────────────┘
```

Las diferencias clave son:

- OSI tiene 7 capas; TCP/IP tiene 4 (o 5).
- OSI es un modelo teorico; TCP/IP es un modelo pratico basado en implementacion real.
- OSI separa sesion, presentacion y aplicacion; TCP/IP las fusiona en una sola capa.
- TCP/IP se desarrolla primero y se estandariza despues; OSI se estandariza primero y se implementa despues.

## Por que TCP/IP es el estandar de internet

TCP/IP gano la batalla contra OSI por varias razones:

- **Simplicidad:** menos capas significa menos complejidad en la implementacion y menor overhead en la comunicacion.
- **Implementacion temprana:** TCP/IP ya existia en ARPANET cuando OSI era solo un borrador. La base instalada era inmensa.
- **Aprovision de BSD Unix:** la inclusion de TCP/IP en BSD Unix en los anos 80 lo difundio rapidamente en universidades e investigadores.
- **Flexibilidad:** TCP/IP no impone restricciones estrictas sobre las capas superiores, permitiendo la innovacion rapida en protocolos de aplicacion.
- **Escalabilidad:** el modeloTCP/IP se adapto naturalmente al crecimiento de internet desde una red academica hasta la red global actual.

> TCP/IP no reemplaza a OSI: OSI sigue siendo valioso como herramienta educativa y de diagnostico. La terminologia OSI (capa 3, capa 7, etc.) es universalmente entendida por los profesionales de redes, incluso cuando trabajan con protocolos TCP/IP. Comprender ambos modelos permite comunicarse eficazmente con profesionales de diferentes contextos y abordar problemas desde multiples perspectivas.

