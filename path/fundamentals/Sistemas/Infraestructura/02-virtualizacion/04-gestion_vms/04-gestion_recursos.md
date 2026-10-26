# Gestion de recursos

La gestion eficiente de recursos es el corazon de la virtualizacion. Asignar los recursos correctos a cada VM equilibra rendimiento y eficiencia: demasiados recursos desperdician hardware; demasiado pocos degradan la experiencia. Los hipervisores ofrecen multiples mecanismos para controlar, limitar y monitorear el consumo de CPU, memoria, almacenamiento y red.

## Gestion de CPU

**vCPUs (CPU virtuales):**
Cada vCPU corresponde a un hilo de ejecucion del CPU fisico. Si un servidor fisico tiene 16 nucleos y 32 hilos (hyper-threading), puede ejecutar hasta 32 vCPUs.

```bash
# VirtualBox: asignar 2 vCPUs
VBoxManage modifyvm "mi-vm" --cpus 2

# Verificar uso de CPU en la VM
top
# O con htop:
htop
```

**Limites y reservas:**
- **Limite:** CPU maxima que la VM puede usar (ej. 80% de un nucleo).
- **Reserva:** CPU minima garantizada (ej. 0.5 nucleos).
- **Peso (*weight*):** prioridad relativa cuando hay contention de CPU.

## Gestion de memoria

La memoria es el recurso mas critico. La VM necesita suficiente RAM para su SO y aplicaciones. Asignar demasiada memoria a una VM reduce la disponible para otras VMs.

```bash
# VirtualBox: asignar 4GB de RAM
VBoxManage modifyvm "mi-vm" --memory 4096

# VirtualBox: configurar memoria dinamica (ballooning)
VBoxManage modifyvm "mi-vm" --memory 4096
VBoxManage modifyvm "mi-vm" --vram 128
```

**Memoria dinamica (ballooning):**
El driver de ballooning dentro de la VM puede devolver memoria no utilizada al hipervisor. Esto permite sobreviaasignar memoria (asignar mas memoria total de la fisicamente disponible) siempre que no todas las VMs la usen al maximo simultaneamente.

**Diferencia entre memoria asignada y uso real:**
Una VM con 4 GB asignados puede estar usando solo 1.5 GB si el SO y las aplicaciones no necesitan mas. Los GB no utilizados estan disponibles para otras VMs en el servidor.

## Gestion de almacenamiento

**Discos dinamicos vs. fijos:**

| Tipo     | Ventaja                         | Desventaja                            |
| -------- | ------------------------------- | ------------------------------------- |
| Dinamico | Ocupa solo lo que usa           | Rendimiento menor al crecer           |
| Fijo     | Mejor rendimiento (preasignado) | Ocupa todo el espacio desde el inicio |

```bash
# VirtualBox: crear disco dinamico (default)
VBoxManage createmedium disk --filename mi-disco.vdi --size 50000

# VirtualBox: crear disco fijo
VBoxManage createmedium disk --filename mi-disco.vdi --size 50000 --variant Fixed
```

**Snapshots y espacio:**
Cada snapshot crea un disco delta que almacena los cambios. Si tienes muchos snapshots, el consumo de disco puede crecer significativamente.

## Gestion de red

```bash
# VirtualBox: configurar multiples interfaces
VBoxManage modifyvm "mi-vm" --nic1 nat             # Internet
VBoxManage modifyvm "mi-vm" --nic2 intnet           # Red interna
VBoxManage modifyvm "mi-vm" --nic3 hostonly         # Solo host

# Limitar ancho de banda (KVM)
tc qdisc add dev vnet0 root tbf rate 100mbit burst 32kbit latency 50ms
```

## Sobre asignación

La sobre asignación permite asignar mas recursos virtuales de los que fisicamente existen. Es una practica comun pero requiere monitoreo cuidadoso.

```
Servidor fisico: 32 GB RAM
VM1: 8 GB asignados (usa 3 GB)
VM2: 8 GB asignados (usa 4 GB)
VM3: 8 GB asignados (usa 2 GB)
VM4: 8 GB asignados (usa 5 GB)
Total asignado: 32 GB (100%)
Total usado: 14 GB (44%)
Sobreviaasignacion: segura porque el uso real es menor
```

> La gestion de recursos es donde la virtualizacion demuestra su verdadero poder. Un servidor fisico con 64 GB de RAM puede ejecutar 16 VMs de 4 GB cada una, incluso si solo 10 GB estan siendo utilizados realmente. Esta eficiencia es imposible sin virtualizacion.

