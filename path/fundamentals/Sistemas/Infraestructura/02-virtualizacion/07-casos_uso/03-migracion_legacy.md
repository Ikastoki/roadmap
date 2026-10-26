# Migracion de sistemas legacy

Miles de organizaciones dependen de software legacy: aplicaciones criticas que fueron escritas hace decadas y que solo funcionan en versiones antiguas de Windows, sistemas operativos obsoletos o hardware que ya no se fabrica. La virtualizacion ofrece una salida a este problema: en lugar de mantener servidores fisicos antiguos dedicados a estas aplicaciones, se virtualiza el sistema completo y se ejecuta en hardware moderno.

## El problema del software legacy

El software legacy presenta desafios unicos:
- Solo funciona en SO antiguos (Windows XP, Windows Server 2003, CentOS 5).
- Los fabricantes dejaron de soportar el hardware original.
- No hay actualizaciones de seguridad disponibles.
- El codigo fuente puede no existir o no ser mantenible.
- La migracion a una plataforma moderna es demasiado costosa o riesgosa.

## Estrategia de migracion por virtualizacion

**Paso 1: Documentar el entorno actual**

```bash
# Documentar el servidor fisico que se va a virtualizar
# - SO y version exacta
# - Aplicaciones instaladas
# - Configuracion de red
# - Dependencias de hardware (tarjetas ISA, dispositivos serie)
# - Usuarios y permisos
# - Horarios de backup
```

**Paso 2: Preparar el hipervisor**

```bash
# Instalar ESXi o Proxmox en un servidor moderno
# Asegurar compatibilidad de hardware
# Configurar almacenamiento y red
```

**Paso 3: Convertir el servidor fisico a VM**

```bash
# Usar P2V (Physical-to-Virtual) tools
# VMware Converter: convierte servidores fisicos a VMs VMware
# PlateSpin: convierte entre diferentes plataformas

# O: instalar el SO desde cero en la VM
# y reinstallar la aplicacion legacy
```

**Paso 4: Configurar y probar**

```bash
# Verificar que la aplicacion funciona en la VM
# Configurar red (puede necesitar IPs estaticas)
# Probar impresoras, dispositivos USB,etc.
# Verificar rendimiento
```

**Paso 5: Desplegar y monitorear**

```bash
# Apagar el servidor fisico original
# Encender la VM en su lugar
# Monitorear rendimiento y estabilidad
# Mantener backups de la VM
```

## Ejemplo practico

```
Antes:
  Servidor Dell PowerEdge 2950 (2008)
  - Windows Server 2003 R2
  - Aplicacion contable VB6
  - Base de datos Access 2003
  - Riesgo: hardware fallando, sin soporte de seguridad

Despues:
  Servidor moderno (Proxmox VE)
  └── VM: Windows Server 2003 R2
      - Misma aplicacion contable
      - Misma base de datos
      - Snapshots para backup
      - Migracion facil a otro host
      - Hardware moderno y fiable
```

## Herramientas de virtualizacion legacy

- **VMware Converter:** P2V nativo para VMware.
- **VirtualBox:** gratuito, soporta SO antiguos.
- **QEMU:** emulacion de hardware antiguo (ISA, PCI legacy).
- **86Box/PCem:** emuladores de PC antiguos para software muy legacy.

## Riesgos y mitigaciones

- **Licencias:** el SO legacy puede tener licencias atadas al hardware original. Consultar con el proveedor.
- **Drivers:** algunos dispositivos legacy no tienen drivers para VMs. Buscar alternativas virtuales.
- **Rendimiento:** el software legacy puede no ser compatible con virtualizacion. Probar antes de migrar.
- **Seguridad:** el SO legacy sigue sin parches. Aislar la VM en una red dedicada.

> La migracion de sistemas legacy por virtualizacion es una de las aplicaciones mas valiosas de esta tecnologia. Permite preservar software critico sin depender de hardware obsoleto, proporcionando estabilidad, portabilidad y la capacidad de backups que los servidores fisicos antiguos no pueden ofrecer.

