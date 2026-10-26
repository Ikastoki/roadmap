# Ciclo de vida de una VM

Una maquina virtual sigue un ciclo de vida bien definido: desde su creacion hasta su eliminacion, pasando por diferentes estados que determinan que puede hacer el sistema y que recursos consume. Comprender este ciclo es fundamental para gestionar eficientemente los recursos del servidor y mantener el entorno virtualizado ordenado.

## Estados de una VM

```
┌──────────┐    start     ┌──────────┐
│  Creada  │─────────────▶│ Ejecutando│
└──────────┘              └──────────┘
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Suspendida│ │ Pausada  │ │ Apagada  │
              └──────────┘ └──────────┘ └──────────┘
                    │                     │
                    │    resume/start      │
                    └─────────────────────┘
                               │
                          ┌────▼────┐
                          │Clonada  │
                          └─────────┘
                               │
                          ┌────▼────┐
                          │Eliminada│
                          └─────────┘
```

**Creada:** la VM existe en el inventario pero no esta encendida. Los archivos de disco estan en el almacenamiento pero no consumen CPU ni memoria activa.

**Ejecutando (*running*):** la VM esta encendida y ejecutando el SO invitado. Consume los recursos asignados (vCPU, RAM, disco, red).

**Suspendida (*saved*):** el estado de la memoria se guarda en disco. Al reanudar, la VM continua exactamente donde la dejaste, como si no se hubiera apagado. No consume CPU ni memoria pero mantiene el estado en disco.

**Pausada (*paused*):** la VM se congela temporalmente sin guardar estado a disco. Util para liberar CPU momentaneamente sin perder el estado de memoria.

**Apagada (*powered off*):** la VM esta apagada completamente. No consume recursos ni mantiene estado.

**Clonada:** se ha creado una copia de la VM. La VM original sigue existiendo y la clon es una entidad independiente.

**Eliminada (*destroyed*):** los archivos de la VM se eliminan permanentemente. Esta accion es irreversible (a menos que existan snapshots).

## Comandos del ciclo de vida

```bash
# VirtualBox: gestion del ciclo de vida
VBoxManage startvm "mi-vm"                        # Encender
VBoxManage controlvm "mi-vm" acpipowerbutton      # Apagar (ACPI)
VBoxManage controlvm "mi-vm" poweroff             # Apagar forzado
VBoxManage controlvm "mi-vm" suspend               # Suspender (guardar estado)
VBoxManage controlvm "mi-vm" pause                 # Pausar
VBoxManage controlvm "mi-vm" resume                # Reanudar desde pausa
VBoxManage controlvm "mi-vm" reset                 # Reiniciar (like boton reset)
VBoxManage clonevm "mi-vm" --name "copia" --register  # Clonar
VBoxManage unregistervm "mi-vm" --delete           # Eliminar
```

## Consideraciones importantes

- **Suspender vs. apagar:** suspender es rapido pero requiere espacio en disco para la memoria. Apagar es limpio pero pierde el estado de la sesion.
- **Pausar vs. suspender:** pausar es instantaneo pero la VM sigue consumiendo disco. Suspender libera memoria pero tarda mas.
- **Eliminar:** antes de eliminar una VM, verifica que no necesitas sus discos o snapshots. Algunos hipervisores eliminan los archivos pero otros los mueven a papelera.
- **Snapshots:** no son backups. Son puntos de restauracion que dependen de la VM original.

> El ciclo de vida de una VM es similar al de un servidor fisico pero con mucho mas control. Puedes pausar, suspender, clonar y eliminar VMs en segundos, algo imposible con hardware fisico. Esta flexibilidad es una de las ventajas mas significativas de la virtualizacion.

