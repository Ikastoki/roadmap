# Arquitectura interna de los hipervisores Tipo 1

La arquitectura de un hipervisor de Tipo 1 esta disenada para un unico proposito: ejecutar multiples maquinas virtuales con el maximo rendimiento y aislamiento posible. A diferencia de un sistema operativo generalista, el hipervisor solo implementa las funcionalidades necesarias para gestionar VMs, lo que reduce su superficie de ataque y su consumo de recursos.

## Control directo del hardware

El hipervisor tiene acceso directo al CPU, la memoria, los discos y las interfaces de red. No existe un SO intermedio que filtre las operaciones, lo que minimiza la latencia en cada operacion de E/S.

```
VM (Ubuntu)
  │
  │  syscall de disco
  │
  ▼
Hipervisor Tipo 1
  │
  │  acceso directo al hardware
  │
  ▼
Controlador NVMe del servidor
```

Esta ausencia de capas intermedias es la razon principal del alto rendimiento de los hipervisores bare-metal.

## Extensiones de virtualizacion del hardware

Los procesadores modernos incluyen extensiones especificas para virtualizacion:

- **Intel VT-x:** permite que el CPU ejecute instrucciones de VM en modo separado (VMX root y VMX non-root).
- **AMD-V (AMD Virtualization):** equivalente de AMD a VT-x.
- **ARM VHE (Virtualization Host Extensions):** extensiones de virtualizacion para arquitectura ARM.
- **Intel VT-d / AMD-Vi:** virtualizacion de E/S (DMA e interrupciones), permite asignar dispositivos fisicos directamente a VMs.

```bash
# Verificar si el CPU soporta virtualizacion
grep -E "vmx|svm" /proc/cpuinfo
# vmx: Intel VT-x
# svm: AMD-V
```

## Modos de ejecucion

```
┌──────────────────────────────────────┐
│  Modo invitado (VMX non-root)        │
│  ┌────────────┐  ┌────────────┐      │
│  │    VM 1    │  │    VM 2    │      │
│  │ (Linux)    │  │ (Windows)  │      │
│  └────────────┘  └────────────┘      │
├──────────────────────────────────────┤
│  Modo host (VMX root)                │
│  ┌──────────────────────────────┐    │
│  │      Hipervisor              │    │
│  └──────────────────────────────┘    │
├──────────────────────────────────────┤
│         Hardware (CPU, RAM)          │
└──────────────────────────────────────┘
```

El hipervisor ejecuta en modo root (nivel de privilegio maximo). Las VMs ejecutan en modo non-root, lo que significa que incluso si un SO invitado "cree" tener control total del CPU, en realidad esta ejecutando en un entorno controlado por el hipervisor.

## Gestion de memoria

El hipervisor gestiona la memoria fisica y la distribuye entre las VMs. Cada VM tiene su propio espacio de direcciones virtual, aislado del de las demas.

- **Asignacion estatica:** se asigna un bloque fijo de RAM a cada VM.
- **Ballooning:** un driver dentro de la VM devuelve memoria no utilizada al hipervisor.
- **KSM (Kernel Same-page Merging):** deduplica paginas de memoria identicas entre VMs (reduccion de consumo).

## Gestion de E/S

El hipervisor gestiona el acceso a dispositivos de E/S (discos, red, USB) de forma centralizada:

- **VirtIO:** drivers paravirtualizados que optimizan la comunicacion entre VM e hipervisor.
- **SR-IOV (Single Root I/O Virtualization):** permite asignar interfaces de red fisicas directamente a VMs para maximo rendimiento.
- **Passthrough de dispositivos:** asignar GPUs, tarjetas NVMe o otros dispositivos fisicos directamente a una VM.

> La arquitectura del hipervisor Tipo 1 esta optimizada para eliminar cualquier innecesario. No hay demonios de fondo, no hay servidores de audio, no hay actualizaciones automaticas: solo la maquinaria minima para ejecutar VMs con eficiencia y aislamiento maximos.

