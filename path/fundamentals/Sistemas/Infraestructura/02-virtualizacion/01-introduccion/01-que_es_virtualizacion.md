# Que es la virtualizacion

La virtualizacion es la creacion de versiones abstractas de recursos fisicos —CPU, memoria, almacenamiento, red— mediante una capa de software que permite ejecutar multiples sistemas operativos o entornos aislados sobre un mismo hardware fisico. En lugar de asignar un servidor fisico a una unica tarea, la virtualizacion permite partitionar ese servidor en multiples unidades logicas independientes, cada una con sus propios recursos asignados y su propio sistema operativo.

Esta tecnologia elimino la dependencia directa entre hardware y software que definio la informatica durante las primeras decadas de su existencia. Un servidor fisico ya no es un sistema operativo: es una plataforma sobre la que pueden ejecutarse muchos sistemas operativos simultaneamente.

## Analogia practica

Un servidor fisico sin virtualizar es como un apartamento entero ocupado por una sola persona: desperdicia espacio y recursos. Virtualizar ese servidor es como dividir el apartamento en varios pisos independientes, cada uno con su propia cocina, bano y entrada. Cada inquilino vive de forma aislada, pero todos comparten la estructura del edificio (el hardware).

## Componentes fundamentales

- **Hardware fisico:** el servidor real con CPU, RAM, discos y tarjetas de red.
- **Hipervisor:** el software que crea y gestiona las maquinas virtuales sobre el hardware.
- **Maquina virtual (VM):** una instancia aislada que emula un servidor completo con su propio SO.
- **Sistema operativo invitado (*guest*):** el SO que se ejecuta dentro de la VM.
- **Sistema operativo anfitrion (*host*):** el SO que ejecuta el hipervisor (en virtualizacion Tipo 2).

## Ejemplo basico

```bash
# Instalar VirtualBox (hipervisor Tipo 2) en Ubuntu
sudo apt update
sudo apt install virtualbox

# Crear una VM desde linea de comandos
VBoxManage createvm --name "ubuntu-server" --ostype Ubuntu_64 --register
VBoxManage modifyvm "ubuntu-server" --memory 2048 --cpus 2
VBoxManage createmedium disk --filename ~/ubuntu-server.vdi --size 20000
VBoxManage storagectl "ubuntu-server" --add sata --controller IntelAhci
VBoxManage storageattach "ubuntu-server" --storagectl sata --port 0 --device 0 --type hdd --medium ~/ubuntu-server.vdi
VBoxManage storagectl "ubuntu-server" --add ide --controller PIIX4
VBoxManage storageattach "ubuntu-server" --storagectl ide --port 0 --device 0 --type dvddrive --medium /home/usuario/ubuntu-22.04.iso
VBoxManage startvm "ubuntu-server" --type headless
```

> La virtualizacion no es solo una tecnologia de ahorro de costos: es un cambio de paradigma. Antes, el software dependia del hardware; despues, el hardware sirve a multiples software independientes. Esta inversion permitio la nube computacional, los contenedores y todo el ecosistema moderno de infraestructura.

