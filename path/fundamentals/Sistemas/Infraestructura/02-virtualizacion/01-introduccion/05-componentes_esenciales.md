# Componentes esenciales de la virtualizacion

La virtualizacion de hardware se sustenta en cuatro componentes fundamentales: el hipervisor, la maquina anfitriona, la maquina invitada y los recursos compartidos. Comprender como interactuan es clave para diseñar, administrar y solucionar problemas en entornos virtualizados.

## El hipervisor

El hipervisor es el nucleo de la virtualizacion. Es el software (o firmware) que crea, ejecuta y gestiona las maquinas virtuales sobre el hardware fisico. Controla el acceso de cada VM al CPU, memoria, almacenamiento y red, garantizando el aislamiento entre ellas.

**Funciones principales:**
- Crear y destruir VMs.
- Asignar y recuperar recursos (CPU, RAM, disco, red).
- Aislar las VMs entre si y del hardware fisico.
- Gestionar la ejecucion concurrente de multiples VMs.
- Facilitar snapshots, migracion y clonacion.

**Clasificacion:**
- **Tipo 1 (bare-metal):** se ejecuta directamente sobre el hardware, sin SO anfitrion intermedio.
- **Tipo 2 (alojado):** se ejecuta como aplicacion sobre un SO anfitrion (Windows, Linux, macOS).

## Maquina anfitriona (host)

Es el servidor fisico real que ejecuta el hipervisor. Proporciona el hardware subyacente: CPU, memoria RAM, discos duros o SSD, tarjetas de red y demas componentes fisicos.

El host no necesita ser un servidor de gama alta para virtualizar. Un servidor con 32 GB de RAM puede ejecutar comodamente 8-10 VMs con 2-4 GB cada una, dependiendo de la carga de trabajo.

**Especificaciones tipicas de un host:**
- **CPU:** multiples nucleos con extensiones de virtualizacion (VT-x, AMD-V).
- **RAM:** suficiente para todas las VMs mas el hipervisor (el hipervisor consume poco).
- **Almacenamiento:** SSD recomendado para VMs con E/S intensiva; HDD para VMs pasivas.
- **Red:** multiples interfaces de red para segmentar trafico.

## Maquina invitada (guest)

Es la VM en si: un sistema operativo completo que se ejecuta de forma aislada sobre el hipervisor. Cada guest cree ser un servidor fisico independiente.

El guest incluye:
- Su propio kernel y sistema operativo.
- Drivers paravirtualizados (que optimizan la comunicacion con el hipervisor).
- Herramientas de integracion (VMware Tools, VirtualBox Guest Additions, spice-vdagent).
- Aplicaciones y servicios que ejecuta.

## Recursos compartidos

El hipervisor distribuye los recursos fisicos del host entre las VMs. Esta distribucion puede ser estatica (asignada fija) o dinamica (se ajusta segun demanda).

**CPU virtual:**
- Cada vCPU corresponde a un hilo de ejecucion del CPU fisico.
- El hipervisor alterna los ciclos de CPU entre las VMs.
- No es lo mismo que un nucleo fisico: varias vCPUs pueden compartir un nucleo fisico.

**Memoria:**
- Se asigna un bloque fijo o dinamico de RAM a cada VM.
- La memoria no se comparte por defecto entre VMs (aislamiento).
- Technologies como KSM (Kernel Same-page Merging) pueden deduplicar paginas iguales.

**Almacenamiento:**
- Discos virtuales: archivos que emulan discos fisicos (VMDK, VDI, qcow2).
- Pueden ser dinamicos (crecen segun uso) o preasignados (tamaño fijo).

**Red:**
- Interfaces de red virtuales conectadas a switches virtuales.
- Cada VM puede tener multiples interfaces virtuales.
- Las VMs pueden conectarse a la red fisica, a otras VMs o estar completamente aisladas.

> El hipervisor es el administrador de un edificio de apartamentos: asigna pisos (VMs), gestiona servicios compartidos (electricidad, agua = CPU, memoria) y garantiza que lo que ocurre en un apartamento no afecte a los demas. Sin el hipervisor, la virtualizacion simplemente no existe.

