# Maquinas virtuales vs. entornos sandbox

Las sandbox (cajas de arena) y las maquinas virtuales son ambos mecanismos de aislamiento, pero operan en niveles diferentes de la pila de tecnologia. Una sandbox aísla procesos o aplicaciones dentro del mismo sistema operativo; una VM aísla sistemas operativos completos. La eleccion entre ambas depende del nivel de aislamiento necesario, el rendimiento requerido y la amenaza que se intenta mitigar.

## Nivel de aislamiento

```
Aislamiento por nivel (de menor a mayor):
├── Sandbox de proceso: aislamiento a nivel de syscall
├── Contenedor: aislamiento a nivel de namespaces/cgroups
├── Sandbox de navegador: aislamiento a nivel de pestaña/proceso
├── Maquina virtual: aislamiento a nivel de SO completo
└── Hardware fisico: aislamiento fisico absoluto
```

**Sandbox de proceso:**
- Un proceso se ejecuta en un entorno restringido.
- No puede acceder a archivos, red o memoria fuera de su sandbox.
- Ejemplos: seccomp, Landlock (Linux), AppArmor en modo sandbox.

**Sandbox de contenedor:**
- Un conjunto de procesos se ejecuta aislado del resto del sistema.
- Comparte el kernel del host pero tiene su propio namespace de archivos, red y usuarios.
- Ejemplos: Docker, Podman, Firejail.

**Sandbox de VM:**
- Un sistema operativo completo se ejecuta aislado en una VM.
- Tiene su propio kernel, su propia memoria y su propia red.
- Ejemplos: VirtualBox, VMware, QEMU.

## Comparacion de seguridad

| Aspecto | Sandbox | VM |
|---|---|---|
| Aislamiento | Por proceso/contenedor | Por SO completo |
| Escape | Posible si hay vulnerabilidad en el kernel | Difícil (requiere escape del hipervisor) |
| Superficie de ataque | Kernel del host compartido | Kernel separado |
| Rendimiento | Casi nativo | Sobrecarga del 5-15% |
| Recursos | Minimos | GB de RAM, GB de disco |

## Casos de uso

**Sandbox para:**
- Ejecutar codigo de fuentes no confiables (evaluar scripts, code runners).
- Navegacion web en entornos no confiables.
- Ejecutar pruebas automatizadas sin afectar el sistema.
- Aislar procesos criticos de otros procesos.

**VM para:**
- Analisis de malware que pueda comprometer el kernel.
- Pruebas de actualizaciones del SO en entornos controlados.
- Ejecutar software de otra arquitectura (con emulacion).
- Entornos multi-tenancy donde el aislamiento es critico.

## Ejemplo practico: Firejail vs. VirtualBox

```bash
# Firejail: sandbox para Firefox
firejail firefox
# Firefox se ejecuta en un entorno restringido
# No puede acceder a archivos fuera de su directorio permitido
# Rendimiento: casi nativo

# VirtualBox: ejecutar Firefox en una VM
VBoxManage startvm "ubuntu-vm"
# Firefox se ejecuta en un SO completo aislado
# No puede acceder a nada del host (ni siquiera ve el host)
# Rendimiento: 80-90% del nativo
```

## Trade-off

La Sandbox es rapida y ligera pero menos segura: si el atacante encuentra una vulnerabilidad en el kernel del host, puede escapar de la sandbox. La VM es mas lenta y pesada pero mas segura: incluso si el atacante compromete el SO invitado, no puede acceder al host.

> La Sandbox y la VM no son alternativas excluyentes. En un entorno de seguridad robusto, se usan ambas: sandbox para aislar procesos individuales y VM para aislar sistemas completos. La clave es entender el nivel de amenaza y elegir el mecanismo de aislamiento adecuado.

