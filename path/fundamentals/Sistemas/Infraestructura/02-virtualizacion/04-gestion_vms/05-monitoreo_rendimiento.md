# Monitoreo y rendimiento

El monitoreo de VMs es esencial para detectar cuellos de botella, planificar扩容, evitar la degradacion del servicio y optimizar el uso de recursos. Un servidor virtualizado con 20 VMs puede tener problemas de rendimiento que no son evidentes sin herramientas de monitoreo adecuadas.

## Metricas clave

**CPU:**
- **Uso de CPU:** porcentaje de tiempo que la CPU esta ocupada. Valores sostenidos por encima del 80% indican sobrecarga.
- **Ready time:** tiempo que una VM espera porque el CPU no esta disponible. Valores altos indican sobreviaasignacion de CPU.
- **Steal time:** tiempo que el hipervisor roba ciclos de CPU a la VM para atender a otras VMs.

**Memoria:**
- **Uso de memoria:** porcentaje de la RAM asignada que esta siendo utilizada.
- **Ballooning:** memoria recuperada por el hipervisor a traves del driver de ballooning. Si hay mucho ballooning, la VM necesita mas memoria.
- **Swap:** cuando la VM usa disco como memoria virtual. Si hay swap, la VM necesita mas RAM urgentemente.

**Almacenamiento:**
- **IOPS:** operaciones de entrada/salida por segundo. Indica la carga de E/S.
- **Latencia:** tiempo de respuesta del disco. Valores por encima de 20ms indican problemas.
- **Throughput:** volumen de datos leidos/escritos por segundo.

**Red:**
- **Ancho de banda:** datos transmitidos/recibidos por segundo.
- **Paquetes丢弃:** paquetes descartados por congestion de red.

## Herramientas de monitoreo

**Desde el hipervisor:**

```bash
# VirtualBox: ver info de una VM
VBoxManage showvminfo "mi-vm" --machinereadable | grep -E "CPU|Memory|Network"

# VMware ESXi: CLI
esxtop
# Teclas: c (CPU), m (memoria), n (red), d (disco)

# Proxmox: interfaz web incluye graficos de uso
# CLI:
qm status 100
pvesh get /nodes/$(hostname)/status
```

**Agentes en el invitado:**

```bash
# Dentro de la VM Linux:
# Monitoreo basico
top                    # Uso de CPU y memoria en tiempo real
htop                   # Version mejorada de top
iostat -x 1            # Estadisticas de disco
sar -n DEV 1           # Estadisticas de red
vmstat 1               # Resumen de actividad del sistema
free -h                # Uso de memoria
```

**Soluciones externas (multi-VM):**

```bash
# Prometheus + Grafana: monitoreo centralizado de multiples VMs
# Instalar node_exporter en cada VM:
docker run -d --name node-exporter \
  -p 9100:9100 \
  prom/node-exporter

# Grafana: dashboards visuales con graficos de uso de recursos
```

## Identificacion de cuellos de botella

**VM lenta pero CPU baja:** probablemente瓶颈 en disco (I/O) o memoria (swap).
**VM lenta y CPU alta:** necesita mas vCPUs o la aplicacion no es eficiente.
**VM con mucho swap:** necesita mas RAM.
**VM con latencia de disco alta:** el almacenamiento es el cuello de botella.
**VM con paquetes丢弃:** la red esta congestionada.

## Optimizacion

```bash
# Si una VM tiene swap, aumentar la memoria
VBoxManage modifyvm "mi-vm" --memory 8192

# Si una VM tiene CPU al 100% constante, añadir vCPUs
VBoxManage modifyvm "mi-vm" --cpus 4

# Si el disco es lento, migrar a SSD o NVMe
# (en VirtualBox: copiar el VDI a un SSD y actualizar la ruta)
```

> Monitorear no es solo detectar problemas: es anticiparse. Un servidor que opera al 70% de su capacidad es saludable; uno al 95% esta a punto de fallar. Las metricas de monitoreo proporcionan la visibilidad necesaria para tomar decisiones de扩容 antes de que el servicio se degraden.

