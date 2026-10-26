# Escenarios de uso inicial

La virtualizacion no es solo para centros de datos empresariales. Sus casos de uso mas comunes incluyen desarrollo de software, aprendizaje, pruebas y ejecucion de sistemas operativos alternativos, todos desde una maquina personal. Comprender estos escenarios ayuda a entender por que la virtualizacion se ha vuelto omnipresente.

## Probar sistemas operativos sin particionar el disco

Antes de la virtualizacion, instalar un segundo SO requeria particionar el disco duro, lo que era riesgoso y requeria reiniciar la maquina para cambiar de SO. Con un hipervisor Tipo 2, puedes ejecutar Linux, Windows o macOS como una ventana mas en tu escritorio.

```bash
# Descargar una ISO de Ubuntu
wget https://releases.ubuntu.com/22.04/ubuntu-22.04-desktop-amd64.iso

# Crear una VM en VirtualBox
VBoxManage createvm --name "ubuntu-test" --ostype Ubuntu_64 --register
VBoxManage modifyvm "ubuntu-test" --memory 4096 --cpus 2
```

## Ejecutar software legacy

Muchas empresas dependen de aplicaciones que solo funcionan en versiones antiguas de Windows o sistemas operativos obsoletos. En lugar de mantener un servidor fisico dedicado para una aplicacion legacy, se puede virtualizar el sistema completo y ejecutarlo en un servidor moderno.

```
Servidor moderno (Windows Server 2022 + Hyper-V)
└── VM: Windows XP (ejecutando aplicacion contable legacy de 2005)
```

## Entornos de laboratorio para aprendizaje

La virtualizacion es la herramienta ideal para aprender tecnologias de TI sin riesgo. Puedes crear un laboratorio completo con multiples VMs interconectadas: servidores web, bases de datos, firewalls, clientes. Si algo falla, eliminas la VM y la creas de nuevo.

```bash
# Laboratorio de red con Vagrant
mkdir laboratorio-red
cd laboratorio-red

# Crear Vagrantfile con 3 VMs
cat > Vagrantfile << 'EOF'
Vagrant.configure("2") do |config|
  config.vm.define "firewall" do |fw|
    fw.vm.box = "ubuntu/jammy64"
    fw.vm.network "private_network", ip: "192.168.1.1"
  end
  config.vm.define "web" do |web|
    web.vm.box = "ubuntu/jammy64"
    web.vm.network "private_network", ip: "192.168.1.10"
  end
  config.vm.define "db" do |db|
    db.vm.box = "ubuntu/jammy64"
    db.vm.network "private_network", ip: "192.168.1.20"
  end
end
EOF

vagrant up
```

## Pruebas de seguridad

Ejecutar software sospechoso o visitar sitios web no confiables en una VM es una practica de seguridad estandar. Si la VM se infecta con malware, simplemente se elimina y se crea una nueva. El sistema anfitrion permanece intacto.

## Desarrollo multiplataforma

Un desarrollador puede probar su aplicacion en multiples sistemas operativos desde un solo equipo:

```bash
# Probar una aplicacion web en diferentes SO
vagrant up ubuntu    # Linux
vagrant up windows   # Windows
vagrant up macos     # macOS
# Cada VM ejecuta el navegador para probar compatibilidad
```

## Creacion de imagenes y templates

La virtualizacion permite crear plantillas de VMs preconfiguradas que se pueden clonar rapidamente. Esto es util para provisionar nuevos entornos de forma consistente.

```bash
# Clonar una VM en VirtualBox
VBoxManage clonevm "plantilla-web" --name "servidor-web-01" --register
```

> Los escenarios de uso iniciales muestran que la virtualizacion no es exclusiva de la infraestructura empresarial. Cualquier persona con un portatil puede usar un hipervisor Tipo 2 para experimentar, aprender y trabajar con multiples sistemas operativos sin riesgo para su maquina principal.

