# Que es un hipervisor de Tipo 1

Un hipervisor de Tipo 1, tambien llamado nativo o bare-metal, es un software de virtualizacion que se ejecuta directamente sobre el hardware fisico sin necesidad de un sistema operativo anfitrion intermedio. A diferencia de los hipervisores Tipo 2, que se ejecutan como una aplicacion dentro de un SO (como VirtualBox en Windows), el hipervisor Tipo 1 ocupa el lugar del sistema operativo: es el primer software que arranca en el servidor y gestiona directamente el CPU, la memoria, el almacenamiento y la red.

Esta arquitectura elimina la capa intermedia del SO anfitrion, lo que se traduce en menor latencia, mejor rendimiento y un control mas directo sobre el hardware.

## Arquitectura

```
┌─────────────────────────────────────────┐
│            VM1    │    VM2    │    VM3  │
│          (Linux)  │ (Windows) │ (Linux) │
├─────────────────────────────────────────┤
│         Hipervisor Tipo 1 (ESXi)       │
├─────────────────────────────────────────┤
│   CPU  │  RAM  │  Disco  │    Red     │
│       Hardware fisico del servidor      │
└─────────────────────────────────────────┘
```

El hipervisor se ejecuta en el nivel mas bajo del software, justo encima del hardware. Las VMs se ejecutan sobre el hipervisor, cada una con su propio SO invitado.

## Ejemplos principales

- **VMware ESXi:** el hipervisor empresarial mas utilizado. Interfaz web y CLI (ESXCLI). Requiere hardware certificado.
- **Microsoft Hyper-V:** integrado en Windows Server. Utiliza la arquitectura de particion de Windows.
- **KVM (Kernel-based Virtual Machine):** modulo del kernel de Linux que convierte al propio Linux en hipervisor Tipo 1. Es la base de Proxmox, oVirt y la nube de Amazon (EC2).
- **Xen:** hipervisor de codigo abierto utilizado por AWS en sus primeros años. Soporta paravirtualizacion y virtualizacion por hardware.

## Diferencia con Tipo 2

| Aspecto | Tipo 1 (bare-metal) | Tipo 2 (alojado) |
|---|---|---|
| Ejecucion | Directamente sobre hardware | Sobre un SO anfitrion |
| Rendimiento | Alto (sin capa intermedia) | Menor (atraviesa el SO) |
| Aislamiento | Completo (el hipervisor es el SO) | Parcial (depende del SO anfitrion) |
| Uso ideal | Centros de datos, produccion | Escritorio, desarrollo, pruebas |
| Complejidad | Mayor (requiere configuracion dedicada) | Menor (se instala como aplicacion) |

> Un hipervisor Tipo 1 es esencialmente un sistema operativo disenado exclusivamente para ejecutar otros sistemas operativos. No tiene escritorio, no tiene navegador, no tiene aplicaciones: su unico proposito es gestionar VMs con la maxima eficiencia posible.

