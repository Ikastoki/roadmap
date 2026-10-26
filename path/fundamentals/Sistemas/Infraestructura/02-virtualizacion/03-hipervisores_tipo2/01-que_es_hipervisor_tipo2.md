# Que es un hipervisor de Tipo 2

Un hipervisor de Tipo 2, tambien llamado alojado o hosted, es un software de virtualizacion que se ejecuta como una aplicacion dentro de un sistema operativo anfitrion existente (Windows, macOS o Linux). A diferencia del Tipo 1, que se instala directamente sobre el hardware, el Tipo 2 funciona como un proceso mas del SO, solicitando recursos a traves de las APIs del sistema operativo subyacente.

Esta arquitectura lo hace ideal para uso personal, desarrollo y aprendizaje, donde no se necesita el rendimiento maximo del hardware pero si la flexibilidad de ejecutar multiples sistemas operativos en un solo equipo.

## Arquitectura

```
┌──────────────────────────────────────┐
│       VM 1 (Linux)  │  VM 2 (Win)   │
├──────────────────────────────────────┤
│     Hipervisor Tipo 2 (VirtualBox)   │
├──────────────────────────────────────┤
│  SO Anfitrion (Windows 11 / Linux)   │
├──────────────────────────────────────┤
│   CPU  │  RAM  │  Disco  │   Red    │
│         Hardware del PC              │
└──────────────────────────────────────┘
```

Las solicitudes de las VMs pasan por el hipervisor y luego por el SO anfitrion antes de llegar al hardware. Esta capa intermedia introduce algo de latencia, pero para la mayoria de usos es despreciable.

## Ejemplos principales

- **VirtualBox:** gratuito, multiplataforma, la opcion mas popular para uso personal.
- **VMware Workstation Pro/Player:** funcionalidades avanzadas, gratuito para uso personal desde 2024.
- **Parallels Desktop:** optimizado para macOS, excelente integracion con Apple Silicon.
- **QEMU:** emulador y virtualizador de codigo abierto, muy potente pero con curva de aprendizaje alta.

## Diferencia fundamental con Tipo 1

El hipervisor Tipo 2 no tiene control directo sobre el hardware. Cada operacion de E/S (disco, red, video) debe pasar por el SO anfitrion, lo que aumenta la latencia. Para tareas que no requieren alto rendimiento (navegacion, desarrollo web, pruebas), esta diferencia es insignificante. Para cargas de trabajo pesadas (bases de datos transaccionales, servidores de produccion), el Tipo 1 es superior.

> Un hipervisor Tipo 2 es la puerta de entrada mas accesible a la virtualizacion. Con VirtualBox o VMware Workstation, cualquier persona con un portatil puede ejecutar Linux, Windows y macOS en ventanas separadas, aprender administracion de sistemas, probar software y experimentar sin riesgo para su equipo principal.

