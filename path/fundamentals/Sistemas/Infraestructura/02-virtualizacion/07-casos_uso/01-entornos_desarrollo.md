# Entornos de desarrollo y pruebas

La virtualizacion ha transformado la forma en que se desarrolla y prueba software. Antes, un desarrollador necesitaba multiples computadoras para probar su aplicacion en diferentes sistemas operativos. Hoy, un solo portatil con un hipervisor Tipo 2 puede ejecutar Windows, Linux y macOS en ventanas separadas, cada uno con sus propias herramientas y configuraciones.

## Laboratorio de desarrollo local

Un entorno de desarrollo completo puede constar de multiples VMs interconectadas:

```
Laboratorio de desarrollo:
├── VM 1: Ubuntu 22.04 (servidor de aplicacion)
│   ├── Node.js 20
│   ├── PostgreSQL 15
│   └── Redis
├── VM 2: Windows 11 (pruebas de frontend)
│   ├── Chrome, Firefox, Edge
│   └── Herramientas de accesibilidad
├── VM 3: macOS (pruebas de compatibilidad)
│   └── Safari
└── VM 4: Kali Linux (pruebas de seguridad)
    └── Herramientas de pentesting
```

```bash
# Crear laboratorio con Vagrant
mkdir laboratorio-dev
cd laboratorio-dev

cat > Vagrantfile << 'EOF'
Vagrant.configure("2") do |config|
  config.vm.define "app" do |app|
    app.vm.box = "ubuntu/jammy64"
    app.vm.hostname = "app-server"
    app.vm.network "private_network", ip: "192.168.56.10"
    app.vm.provider "virtualbox" do |vb|
      vb.memory = "4096"
      vb.cpus = 2
    end
    app.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y nodejs npm postgresql redis-server
    SHELL
  end

  config.vm.define "db" do |db|
    db.vm.box = "ubuntu/jammy64"
    db.vm.hostname = "db-server"
    db.vm.network "private_network", ip: "192.168.56.20"
    db.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 1
    end
  end
end
EOF

vagrant up
```

## Reproducibilidad

Una de las mayores ventajas de las VMs en desarrollo es la reproducibilidad. Cada desarrollador puede tener exactamente el mismo entorno, eliminando el problema clasico "funciona en mi maquina".

```bash
# Exportar VM reproducible
VBoxManage export "entorno-desarrollo" --output plantilla-dev.ova

# Cualquier miembro del equipo puede importarla
VBoxManage import plantilla-dev.ova
# Resultado: entorno identico en segundos
```

## Pruebas multiplataforma

Probar una aplicacion web en multiples navegadores y sistemas operativos:

```bash
# VM 1: Firefox en Linux
# VM 2: Chrome en Windows
# VM 3: Safari en macOS
# Cada VM ejecuta el navegador correspondiente
# El desarrollador verifica compatibilidad visual y funcional
```

## Pruebas de regresion

Crear una VM base con la aplicacion funcionando correctamente. Despues de cada cambio, clonar la VM y ejecutar las pruebas. Si algo falla, eliminar la clon y crear otra.

```bash
# Clonar VM base para pruebas
VBoxManage clonevm "base-app" --name "test-regresion-$(date +%Y%m%d)" --register

# Ejecutar suite de pruebas
# Si pasa: eliminar la clon
# Si falla: investigar en la clon sin afectar la base
```

> La virtualizacion en desarrollo no es un lujo: es una necesidad. La capacidad de crear, destruir y replicar entornos completos en minutos elimina la friccion del desarrollo multiplataforma y garantiza que las aplicaciones funcionan en todos los entornos donde seran utilizadas.

