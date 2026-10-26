# Virtualizacion vs. emulacion

Virtualizacion y emulacion son conceptos frecuentemente confundidos pero fundamentalmente diferentes. La virtualizacion ejecuta codigo nativo sobre hardware emulado; la emulacion simula una arquitectura de hardware completamente diferente. Comprender esta distincion es clave para elegir la herramienta adecuada para cada tarea.

## Diferencia fundamental

**Virtualizacion:** ejecuta el SO invitado sobre una CPU de la misma arquitectura que el host. El codigo de la VM se ejecuta directamente en el CPU fisico, con aceleracion por hardware (VT-x, AMD-V). Solo se emulan los dispositivos perifericos (disco, red, video).

**Emulacion:** simula una CPU completamente diferente. El codigo de la VM se traduce en tiempo real a la arquitectura del host, lo que requiere interpretacion o compilacion dinamica.

```
VIRTUALIZACION:
Código de la VM (x86)  ──►  CPU host (x86)
  Se ejecuta directamente, con aceleracion por hardware

EMULACION:
Código ARM  ──►  Emulador QEMU  ──►  CPU host (x86)
  Se traduce en tiempo real, mucho mas lento
```

## Rendimiento

La diferencia de rendimiento es abismal. La virtualizacion tiene una sobrecarga del 5-15%; la emulacion puede ser 10-100 veces mas lenta.

| Operacion | Virtualizacion | Emulacion |
|---|---|---|
| Arranque de SO | 30-60 segundos | 5-30 minutos |
| Rendimiento de CPU | 85-95% del nativo | 1-10% del nativo |
| Consumo de RAM | GB | MB-GB |

## QEMU: el caso especial

QEMU es tanto un emulador como un virtualizador. Sin aceleracion por hardware, QEMU emula una CPU completa (lento). Con KVM habilitado, QEMU se convierte en un virtualizador de alto rendimiento.

```bash
# QEMU como emulador (sin KVM) - lento
qemu-system-x86_64 -hda disco.qcow2 -m 2048

# QEMU como virtualizador (con KVM) - rapido
qemu-system-x86_64 -enable-kvm -hda disco.qcow2 -m 2048

# QEMU emulando ARM en x86 - muy lento
qemu-system-aarch64 -M virt -cpu cortex-a57 -hda disco.qcow2
```

## Cuándo usar cada uno

**Usar virtualizacion cuando:**
- La arquitectura de la VM es la misma que el host (x86 en x86, ARM en ARM).
- Necesitas alto rendimiento.
- Ejecutas sistemas operativos de produccion.
- Quieres arrancar en segundos.

**Usar emulacion cuando:**
- Necesitas ejecutar software de otra arquitectura (ARM en x86, x86 en ARM).
- Desarrollas para dispositivos embebidos (Raspberry Pi desde un PC x86).
- Pruebas de compatibilidad cross-platform.
- El rendimiento no es critico.

```bash
# Ejemplo: desarrollar para Raspberry Pi desde un PC
# Usar QEMU para emular ARM
qemu-system-aarch64 \
  -M virt \
  -cpu cortex-a72 \
  -m 2048 \
  -kernel kernel8.img \
  -drive file=sd-card.img,if=sd \
  -nographic
```

## Herramientas populares

- **Virtualizacion:** VMware, VirtualBox, KVM, Hyper-V, Xen.
- **Emulacion:** QEMU, Bochs, DOSBox, RPCS3 (PlayStation 3), Yuzu (Nintendo Switch).
- **Hibrido:** QEMU+KVM (emula dispositivos + acelera CPU).

> La virtualizacion optimiza la ejecucion de software sobre hardware nativo; la emulacion hace posible ejecutar software sobre hardware incompatible. Para la mayoria de usos empresariales, la virtualizacion es la herramienta correcta. La emulacion tiene su lugar en el desarrollo de software embebido, la investigacion y la preservacion de software legacy.

