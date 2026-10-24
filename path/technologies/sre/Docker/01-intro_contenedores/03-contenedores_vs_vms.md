# Contenedores frente a maquinas virtuales

La diferencia fundamental entre contenedores y maquinas virtuales no es de grado sino de naturaleza: mientras que las maquinas virtuales virtualizan el hardware para ejecutar sistemas operativos completos e independientes sobre un hipervisor, los contenedores virtualizan el sistema operativo para ejecutar procesos aislados directamente sobre el kernel del anfitrion. Esta distincion tiene implicaciones profundas en el rendimiento, la densidad, la seguridad y la gestion de los entornos de ejecucion. Comprender cuando usar cada tecnologia requiere analizar sus arquitecturas, sus fortalezas y sus limitaciones inherentes.

## Arquitectura de las maquinas virtuales

Una maquina virtual incluye un sistema operativo completo: su propio kernel, sus propios drivers, su propio init system y todas las bibliotecas del sistema. El hipervisor (tipo 1 como VMware ESXi, Hyper-V o KVM; o tipo 2 como VirtualBox o VMware Workstation) abstrae el hardware fisico y gestiona la multiplexacion de los recursos entre las VMs.

```
+--------------------------------------------------+
|                    Anfitrion                       |
|  +-------------------+  +-------------------+     |
|  | VM 1              |  | VM 2              |     |
|  | +---------------+ |  | +---------------+ |     |
|  | | Aplicacion    | |  | | Aplicacion    | |     |
|  | +---------------+ |  | +---------------+ |     |
|  | | Bibliotecas   | |  | | Bibliotecas   | |     |
|  | +---------------+ |  | +---------------+ |     |
|  | | Sistema       | |  | | Sistema       | |     |
|  | | Operativo     | |  | | Operativo     | |     |
|  | +---------------+ |  | +---------------+ |     |
|  +-------------------+  +-------------------+     |
|  +---------------------------------------------+ |
|  | Hipervisor                                   | |
|  +---------------------------------------------+ |
+--------------------------------------------------+
```

Cada VM consume recursos reservados: memoria RAM asignada, espacio en disco para el sistema operativo invitado y ciclos de CPU para mantener el kernel independiente. Iniciar una VM implica arrancar un sistema operativo completo, lo que puede llevar minutos. El consumo de recursos por VM es significativo, incluso si la aplicacion que ejecuta es minima.

## Arquitectura de los contenedores

Un contenedor comparte el kernel del anfitrion y solo incluye la aplicacion y sus dependencias inmediatas. No existe un sistema operativo separado; los procesos del contenedor se ejecutan directamente sobre el kernel del anfitrion, aislados mediante namespaces y limitados mediante cgroups.

```
+--------------------------------------------------+
|                    Anfitrion                       |
|  +-------------------+  +-------------------+     |
|  | Contenedor 1      |  | Contenedor 2      |     |
|  | +---------------+ |  | +---------------+ |     |
|  | | Aplicacion    | |  | | Aplicacion    | |     |
|  | +---------------+ |  | +---------------+ |     |
|  | | Bibliotecas   | |  | | Bibliotecas   | |     |
|  | +---------------+ |  | +---------------+ |     |
|  +-------------------+  +-------------------+     |
|  +---------------------------------------------+ |
|  | Kernel del anfitrion (Linux)                 | |
|  +---------------------------------------------+ |
|  | Hardware                                     | |
|  +---------------------------------------------+ |
+--------------------------------------------------+
```

La ausencia de un kernel propio hace que los contenedores sean drasticamente mas ligeros. Una imagen de contenedor puede tener unos pocos megabytes, mientras que una VM requiere gigabytes para el sistema operativo. El arranque de un contenedor es casi instantaneo, ya que solo necesita inicializar el proceso y configurar los namespaces.

## Comparativa de rendimiento

En terminos de rendimiento puro, los contenedores tienen una ventaja significativa sobre las VMs porque no introducen la sobrecarga del hipervisor ni del kernel invitado. Las llamadas al sistema realizadas por procesos en un contenedor llegan directamente al kernel del anfitrion, mientras que en una VM deben atravesar el hipervisor y el kernel invitado.

```bash
# Prueba de rendimiento: sysbench en contenedor vs VM
# En un contenedor:
docker run --rm ubuntu bash -c "apt-get update && apt-get install -y sysbench && sysbench cpu run"

# En una VM:
ssh vm "sysbench cpu run"
```

La penalizacion de rendimiento de las VMs tipicamente se situa entre el 5 % y el 20 % dependiendo de la carga de trabajo, mientras que los contenedores tienen una penalizacion cercana a cero para la mayoria de las operaciones. Las operaciones de E/S de disco y red pueden mostrar diferencias mas notables debido a la virtualizacion del almacenamiento y la red en las VMs.

> Para cargas de trabajo intensivas en computacion, los contenedores ofrecen rendimiento nativo. Sin embargo, para escenarios que requieren aislamiento estricto del kernel (como ejecutar modulos del kernel o sistemas operativos diferentes), las VMs son la unica opcion viable.

## Densidad de despliegue

La densidad se refiere al numero de unidades de ejecucion que pueden desplegarse en un mismo servidor fisico. Los contenedores permiten densidades mucho mayores que las VMs porque cada contenedor no necesita su propio sistema operativo. En un servidor con 16 GB de RAM, se podrian ejecutar decenas o cientos de contenedores, mientras que el mismo servidor probablemente no soportaria mas de 5 a 10 VMs con sistemas operativos completos.

```bash
# En un servidor de 16 GB RAM, 4 vCPU
# VMs tipicas: 4 VMs con 2 GB RAM y 1 vCPU cada una -> 8 GB RAM usados solo para SOs
# Contenedores tipicos: 20 contenedores con 256 MB RAM cada uno -> 5 GB RAM total
```

La alta densidad de los contenedores los hace ideales para entornos donde se ejecutan muchos servicios independientes, como microservicios o plataformas de SaaS. Cada servicio puede tener su propio contenedor con sus recursos minimos necesarios, maximizando la utilizacion del hardware.

## Seguridad y aislamiento

El modelo de seguridad de las VMs es inherentemente mas fuerte que el de los contenedores. Cada VM ejecuta su propio kernel, lo que proporciona una barrera de seguridad solida: incluso si un atacante compromete el kernel de una VM, no puede acceder al anfitrion ni a otras VMs sin explotar vulnerabilidades del hipervisor.

```bash
# En una VM comprometida, el atacante tiene control solo sobre esa VM
# El hipervisor aísla el acceso al hardware y a otras VMs

# En un contenedor, el kernel es compartido
# Una vulnerabilidad de escalado de privilegios en el kernel puede comprometer el anfitrion
```

Los contenedores comparten el kernel del anfitrion, lo que significa que una vulnerabilidad que permita escalar privilegios desde un contenedor al anfitrion afecta a todos los contenedores. Docker implementa varias medidas de seguridad para mitigar este riesgo: capacidad de eliminar privilegios (capabilities), modos de seguridad como seccomp y AppArmor, y la ejecucion como usuario no root dentro del contenedor.

> Las VMs ofrecen un aislamiento de seguridad superior. Para cargas de trabajo multitenant donde los inquilinos no son de confianza, las VMs siguen siendo la opcion recomendada. Los contenedores son adecuados cuando el modelo de confianza es mas permisivo (mismo equipo, misma organizacion).

## Portabilidad

Ambas tecnologias ofrecen portabilidad, pero en diferentes niveles. Una VM es portatil en forma de imagen de disco (OVA, QCOW2, VHD) que puede importarse en cualquier hipervisor compatible. Un contenedor es portatil en forma de imagen Docker que puede ejecutarse en cualquier sistema con Docker instalado, independientemente del hipervisor o del proveedor de nube.

```bash
# Una imagen Docker se ejecuta igual en:
docker run mi-app
# - En una laptop con Docker Desktop
# - En un servidor Linux con Docker Engine
# - En AWS ECS
# - En Google Cloud Run
# - En Azure Container Instances
```

La portabilidad de los contenedores es mas granular y practica para aplicaciones modernas. Mientras que mover una VM entre hiperciscopes puede requerir conversion de formatos y ajustes de drivers, una imagen Docker se ejecuta sin cambios en cualquier lugar donde haya Docker.

## Casos de uso ideal

Cada tecnologia tiene escenarios donde brilla. Las VMs son la opcion correcta cuando se necesita ejecutar sistemas operativos diferentes al del anfitrion (por ejemplo, una VM con Windows en un servidor Linux), cuando se requiere aislamiento de seguridad estricto (entornos multitenant no confiables) o cuando se necesita emular hardware especifico.

Los contenedores son la opcion preferida para aplicaciones nativas de la nube, microservicios, pipelines de CI/CD, entornos de desarrollo reproducibles y cualquier escenario donde la densidad, la velocidad de arranque y la eficiencia de recursos sean prioritarias.

```yaml
# Caso tipico de uso mixto: VMs para nodos, contenedores para aplicaciones
# Kubernetes sobre VMs en un cluster
nodes:
  - vm-1: 192.168.1.10
    pods:
      - app-web:1.0
      - api-gateway:2.1
  - vm-2: 192.168.1.11
    pods:
      - app-worker:1.0
      - redis:7.0
```

> No se trata de elegir entre contenedores o VMs, sino de entender como combinarlos. La mayoria de las arquitecturas modernas ejecutan contenedores dentro de VMs, aprovechando el aislamiento de las VMs a nivel de nodo y la eficiencia de los contenedores a nivel de aplicacion.

