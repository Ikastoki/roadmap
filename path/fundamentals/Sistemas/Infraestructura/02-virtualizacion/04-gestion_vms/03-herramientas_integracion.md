# Herramientas de integracion

Las herramientas de integracion (Guest Additions en VirtualBox, VMware Tools en VMware, spice-vdagent en KVM) son controladores y agentes que se instalan dentro del SO invitado para mejorar significativamente la experiencia de virtualizacion. Sin ellas, la VM funciona pero de forma limitada: resolucion fija, sin portapapeles compartido, sin carpetas compartidas, sin sincronizacion de tiempo.

## Que proporcionan

- **Controladores de video:** resolucion adaptativa (la pantalla se ajusta al tamano de la ventana), aceleracion 2D/3D basica.
- **Sincronizacion de tiempo:** la VM mantiene la hora exacta del host, evitando desviaciones.
- **Portapapeles compartido:** copiar y pegar texto e archivos entre host y VM.
- **Carpetas compartidas:** acceder a carpetas del host desde la VM sin configurar red.
- **Arrastrar y soltar:** mover archivos del host a la VM con el mouse.
- **Mouse integrado:** el cursor se mueve libremente entre host y VM sin atajos de teclado.
- **Control de energi:** apagado limpio desde el SO invitado (ACPI).
- **Redireccion de USB:** conectar dispositivos USB fisicos a la VM.

## Instalacion en VirtualBox

```bash
# Dentro de la VM Linux:
# 1. Actualizar el sistema
sudo apt update && sudo apt upgrade -y

# 2. Instalar dependencias de compilacion
sudo apt install -y build-essential dkms linux-headers-$(uname -r)

# 3. Montar el CD de Guest Additions
# Desde VirtualBox: Devices > Insert Guest Additions CD image
sudo mount /dev/cdrom /mnt

# 4. Ejecutar el instalador
cd /mnt
sudo ./VBoxLinuxAdditions.run

# 5. Reiniciar
sudo reboot
```

## Instalacion en VMware

```bash
# Desde VMware: VM > Install VMware Tools
# Se monta un CD virtual con el instalador

# Dentro de la VM Linux:
sudo mount /dev/cdrom /mnt
cd /tmp
sudo tar xzf /mnt/VMwareTools-*.tar.gz
cd vmware-tools-distrib
sudo ./vmware-install.pl

# Seguir el asistente (aceptar valores por defecto)
sudo reboot
```

## Instalacion en KVM/QEMU

```bash
# Instalar spice-vdagent (para redireccion de display)
sudo apt install spice-vdagent

# Instalar virtio drivers (para mejor rendimiento de E/S)
sudo apt install virtio-drivers
```

## Verificacion

```bash
# Verificar que las Guest Additions estan instaladas
VBoxManage guestproperty get "mi-vm" "/VirtualBox/GuestAdd/Version"
# 7.0.14r161095

# En la VM, verificar el servicio
systemctl status vboxadd-service
```

## Solucion de problemas

- **Resolucion no se ajusta:** verificar que las Guest Additions estan instaladas y el servicio esta corriendo.
- **Portapapeles no funciona:** reiniciar la VM despues de instalar las herramientas.
- **Carpetas compartidas no aparecen:** verificar que el usuario esta en el grupo `vboxsf`:
```bash
sudo usermod -aG vboxsf $USER
# Cerrar sesion y volver a entrar
```

> Las herramientas de integracion son la diferencia entre una VM que funciona y una VM que se integra de forma fluida con el entorno de trabajo. Sin ellas, la experiencia es incomoda y limitada. Con ellas, la VM se comporta como una extension natural del sistema anfitrion.

