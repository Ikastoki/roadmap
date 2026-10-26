# Arquitectura interna de los hipervisores Tipo 2

La arquitectura de un hipervisor de Tipo 2 esta definida por su dependencia del sistema operativo anfitrion. A diferencia del Tipo 1, que se ejecuta directamente sobre el hardware, el Tipo 2 opera como un proceso del SO,利用ando sus APIs y controladores para acceder a los recursos fisicos. Esta capa intermedia tiene implicaciones directas en el rendimiento, el aislamiento y la facilidad de uso.

## Capas de abstraccion

```
Aplicacion en VM (navegador, servidor, etc.)
    │
    │ syscall de red
    ▼
SO invitado (Windows/Linux dentro de la VM)
    │
    │ drivers paravirtualizados
    ▼
Hipervisor Tipo 2 (VirtualBox/VMware)
    │
    │ llamadas al SO anfitrion
    ▼
SO anfitrion (Windows 11/Ubuntu)
    │
    │ controladores del kernel
    ▼
Hardware fisico (CPU, RAM, disco, red)
```

Cada operacion que realiza la VM debe atravesar cuatro capas antes de llegar al hardware. En el Tipo 1, solo hay dos capas (hipervisor y hardware). Esta diferencia se traduce en mayor latencia, especialmente en operaciones de E/S.

## Gestion de recursos

El hipervisor Tipo 2 solicita memoria, tiempo de CPU y acceso a disco al SO anfitrion, que luego distribuye entre las VMs.

**Memoria:**
```bash
# VirtualBox: asignar 4GB de RAM a una VM
VBoxManage modifyvm "mi-vm" --memory 4096

# La VM puede usar hasta 4GB, pero el SO anfitrión necesita RAM para si
# Regla: el SO anfitrión necesita al menos 2-4GB para funcionar bien
# Un PC con 16GB puede ejecutar 2-3 VMs con 4GB cada una
```

**CPU:**
```bash
# VirtualBox: asignar 2 nucleos virtuales
VBoxManage modifyvm "mi-vm" --cpus 2

# VMware: configurar 2 CPUs virtuales
# Editar VM > Hardware > Processors > Number of processors: 2
```

**Almacenamiento:**
Los discos virtuales se almacenan como archivos en el sistema de archivos del SO anfitrion.

```
C:\Users\usuario\VirtualBox VMs\mi-vm\mi-vm.vdi  (VirtualBox)
C:\Users\usuario\VirtualBox VMs\mi-vm\mi-vm.vmdk  (VMware)
```

**Red:**
El hipervisor crea interfaces de red virtuales que se conectan a los adaptadores del SO anfitrion.

```bash
# VirtualBox: modos de red
VBoxManage modifyvm "mi-vm" --nic1 nat          # NAT (acceso a Internet)
VBoxManage modifyvm "mi-vm" --nic2 bridged      # Bridge (mismo segmento que host)
VBoxManage modifyvm "mi-vm" --nic3 intnet       # Red interna (solo VMs)
VBoxManage modifyvm "mi-vm" --nic4 hostonly     # Solo host (comunicacion host-VM)
```

## Aceleracion por hardware

Los hipervisores Tipo 2 pueden aprovechar las extensiones de virtualizacion del CPU (VT-x, AMD-V) para reducir la sobrecarga. Sin embargo, la aceleracion completa es dificil de alcanzar porque el SO anfitrion mantiene el control del hardware.

```bash
# Verificar si VirtualBox usa VT-x/AMD-V
VBoxManage showvminfo "mi-vm" | grep "Hardware Virtualization"
# VT-x/AMD-V: Enabled
```

## Integracion con el host

Los hipervisores Tipo 2 ofrecen funcionalidades que el Tipo 1 no puede proporcionar facilmente:

- **Carpetas compartidas:** acceder a archivos del SO anfitrion desde la VM.
- **Portapapeles compartido:** copiar y pegar entre host y VM.
- **Redireccion de USB:** conectar dispositivos USB fisicos a la VM.
- **Resolucion adaptativa:** la ventana de la VM se ajusta al tamano de la ventana.
- **Arrastrar y soltar:** mover archivos entre host y VM con el mouse.

```bash
# VirtualBox: compartir carpeta
VBoxManage sharedfolder add "mi-vm" --name "compartida" --hostpath "/home/usuario/compartida"
```

> La arquitectura del hipervisor Tipo 2 refleja un compromiso: sacrifican rendimiento y aislamiento a cambio de facilidad de uso e integracion con el entorno de trabajo diario. Para un desarrollador que necesita probar su aplicacion en Linux y Windows, esta compensacion es perfectamente aceptable.

