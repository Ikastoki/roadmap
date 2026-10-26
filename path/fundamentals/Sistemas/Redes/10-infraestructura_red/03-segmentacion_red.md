# Segmentacion de red

La segmentacion de red es el proceso de dividir una red en segmentos o subredes mas pequenas, cada una con politicas de seguridad y configuracion independientes. La segmentacion reduce la superficie de ataque, mejora el rendimiento al reducir el trafico broadcast y permite aplicar controles de acceso granulares.

Sin segmentacion, toda la red esta en el mismo dominio de broadcast: cualquier dispositivo puede intentar acceder a cualquier otro, y un compromiso en un dispositivo puede afectar a toda la red.

## VLANs

Las VLANs (*Virtual LANs*) son la herramienta fundamental de segmentacion en redes cableadas. Permiten crear redes logicas independientes dentro de un mismo switch fisico, sin necesidad de cables adicionales ni switches separados.

```
Switch fisico (24 puertos):
┌─────────────────────────────────────┐
│ VLAN 10 (Admin): Puertos 1-8       │
│ VLAN 20 (Desarrollo): Puertos 9-16 │
│ VLAN 30 (Invitados): Puertos 17-24 │
└─────────────────────────────────────┘

Dispositivos en VLAN 10 NO pueden comunicarse
directamente con dispositivos en VLAN 20 o 30.
```

Las VLANs se implementan con etiquetado 802.1Q: cuando un paquete cruza de un switch a otro, se le anade una etiqueta VLAN en la cabecera de la trama que identifica a que VLAN pertenece.

## Zonas DMZ

Una DMZ (*Demilitarized Zone*) es un segmento de red que esta parcialmente aislado de la red interna y parcialmente expuesto a internet. Se utiliza para alojar servicios que deben ser accesibles desde internet (web, correo, DNS) sin comprometer la seguridad de la red interna.

```
Internet
   │
   ▼
[Firewall exterior]
   │
   ├── DMZ: Servidor web, servidor de correo, DNS
   │
   [Firewall interior]
   │
   ├── Red interna: PCs, servidores internos, NAS
```

Los servidores de la DMZ pueden ser accesibles desde internet pero tienen acceso limitado (o nulo) a la red interna. Si un servidor de la DMZ es comprometido, el atacante no puede acceder directamente a los recursos internos.

## Segmentacion logica vs fisica

**Segmentacion fisica:** utilizar switches y cables separados para cada segmento. Ofrece el mayor aislamiento pero es costosa y dificil de gestionar.

**Segmentacion logica (VLANs):** crear segmentos logicos en el mismo hardware. Es flexible, economica y facil de reconfigurar pero depende de la correcta configuracion del switch.

**Segmentacion por subredes:** combinar VLANs con subredes IP diferentes, permitiendo que el router aplique filtros entre segmentos. Es la combinacion mas comun y efectiva.

## Beneficios de la segmentacion

- **Seguridad:** un atacante que compromete un dispositivo en un segmento no puede acceder directamente a los demas segmentos.
- **Rendimiento:** el trafico broadcast se limita a cada segmento, reduciendo el ruido en la red.
- **Gestion:** cada segmento puede tener politicas de QoS, firewall y monitoreo independientes.
- **Cumplimiento normativo:** los requisitos PCI DSS, HIPAA y RGPD a menudo exigen segmentacion para proteger datos sensibles.

> La segmentacion es una de las medidas de seguridad mas efectivas y subestimadas. Una red plana (sin segmentacion) es como un edificio sin puertas internas: si un intruso entra, puede acceder a todas las habitaciones. La segmentacion crea compartimentos que limitan el movimiento lateral del atacante.

