# Creacion de una maquina virtual

Crear una VM es el primer paso para ejecutar un sistema operativo virtualizado. La creacion implica definir los recursos que tendra (CPU, memoria, almacenamiento, red), seleccionar el SO invitado y configurar el medio de instalacion. Las decisiones tomadas en este paso determinan el rendimiento y la eficiencia de la VM a lo largo de su vida util.

## Decisiones clave antes de crear

**Recursos a asignar:**
- **vCPUs:** depende de la carga de trabajo. Un servidor web basico necesita 1-2 vCPUs; una base de datos transaccional puede necesitar 4-8.
- **RAM:** el SO invitado necesita un minimo para arrancar (2 GB para Linux, 4 GB para Windows). Las aplicaciones que ejecuta determinan el total.
- **Disco:** dinamico (crece segun uso) o fijo (tamaño preasignado). Dinamico es mas eficiente en espacio; fijo es mas rapido en E/S.

**Medio de instalacion:**
- Archivo ISO del SO que se va a instalar.
- Plantilla preconfigurada (OVA/OVF) que incluye el SO ya instalado.

## Creacion desde interfaz grafica (VirtualBox)

1. **Nombre y SO:** escribir el nombre, seleccionar tipo (Linux/Windows) y version.
2. **Memoria:** asignar la cantidad de RAM (recomendado: 2-4 GB para desarrollo).
3. **Disco virtual:** crear nuevo disco virtual (VDI, VMDK o VHD), dinamico o fijo.
4. **Configuracion avanzada:** ajustar CPU, video, red, puertos serie.
5. **ISO de instalacion:** en Settings > Storage, adjuntar la ISO al controlador IDE.

## Creacion desde linea de comandos

```bash
# Paso 1: Crear la VM
VBoxManage createvm --name "web-server" --ostype Ubuntu_64 --register

# Paso 2: Configurar recursos
VBoxManage modifyvm "web-server" \
  --memory 2048 \
  --cpus 2 \
  --vram 16 \
  --graphicscontroller vmsvga \
  --boot1 disk \
  --boot2 dvd \
  --nic1 nat

# Paso 3: Crear disco virtual
VBoxManage createmedium disk \
  --filename ~/VirtualBox\ VMs/web-server/web-server.vdi \
  --size 20000 \
  --format VDI

# Paso 4: Agregar controladores y adjuntar disco
VBoxManage storagectl "web-server" --add sata --controller IntelAhci
VBoxManage storageattach "web-server" --storagectl sata \
  --port 0 --device 0 \
  --type hdd \
  --medium ~/VirtualBox\ VMs/web-server/web-server.vdi

# Paso 5: Adjuntar ISO de instalacion
VBoxManage storagectl "web-server" --add ide --controller PIIX4
VBoxManage storageattach "web-server" --storagectl ide \
  --port 0 --device 0 \
  --type dvddrive \
  --medium ~/Descargas/ubuntu-22.04-live-server-amd64.iso

# Paso 6: Iniciar la VM para instalar el SO
VBoxManage startvm "web-server"
```

## Plantillas y OVA

En lugar de instalar el SO manualmente, se pueden importar plantillas preconfiguradas:

```bash
# Importar plantilla OVA
VBoxManage import ubuntu-server-22.04.ova

# La VM se crea con el SO ya instalado y configurado
# Solo queda ajustar recursos si es necesario
VBoxManage modifyvm "ubuntu-server-22.04" --memory 4096 --cpus 4
```

## Configuracion post-instalacion

Despues de instalar el SO, es recomendable:

1. Instalar Guest Additions / VMware Tools.
2. Configurar red estatica si es un servidor.
3. Configurar usuario sin root para uso diario.
4. Tomar un snapshot "estado base" como punto de restauracion.

```bash
# Tomar snapshot del estado limpio post-instalacion
VBoxManage snapshot "web-server" take "estado-base-post-instalacion"
```

> Crear una VM correctamente evita problemas de rendimiento y mantenimiento a futuro. Asignar demasiados recursos desperdicia hardware; asignar muy pocos limita el rendimiento. La clave esta en ajustar los recursos a la carga de trabajo real, no al maximo teorico.

