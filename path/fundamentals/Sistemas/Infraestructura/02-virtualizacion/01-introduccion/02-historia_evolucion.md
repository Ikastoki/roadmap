# Historia y evolucion de la virtualizacion

La virtualizacion no nacio con la nube computacional ni con VMware. Sus origenes se remontan a las decadas de 1960 y 1970, cuando los mainframes de IBM ya permitian partitionar recursos para ejecutar multiples sistemas simultaneamente. La evolucion de esta tecnologia refleja la busqueda constante de eficiencia en el uso del hardware informatico.

## Mainframes y los origenes (1960-1970)

IBM fue pionera con el CP-40 (1964) y el VM/370 (1972), sistemas que permitian crear multiples maquinas virtuales sobre un mainframe. Cada VM ejecutaba su propio sistema operativo (CMS) de forma aislada. La motivacion era simple: los mainframes eran enormemente caros y no se podian permitir que un solo usuario o aplicacion ocupara todo el hardware.

```
Mainframe IBM VM/370
├── VM 1: CMS (desarrollo)
├── VM 2: CMS (produccion)
├── VM 3: OS/360 (lote)
└── VM 4: OS/360 (investigacion)
```

Estos sistemas ya resolvian los problemas fundamentales: aislamiento entre entornos, partitionamiento de recursos y ejecucion simultanea. La diferencia con la virtualizacion moderna era la escala y el costo.

## Estancamiento y renacimiento (1980-1990)

Con la llegada de los PC y los servidores x86 baratos, los mainframes perdieron terreno. La filosofia de "un servidor, un sistema operativo" se impuso. Los servidores x86 corrian un unico SO y ejecutaban una o pocas aplicaciones. La virtualizacion se relego a nichos de mainframe.

Durante esta epoca, surgieron tecnologias que anticiparon el renacimiento: el sistema de archivos UFS de BSD permitia cierto aislamiento, y los *jails* de FreeBSD (1999) introdujeron el concepto de virtualizacion a nivel de sistema operativo.

## El renacimiento x86 (2000-2005)

La virtualizacion en x86 necesito ayuda del hardware porque las arquitecturas x86 originales no estaban disenadas para ser virtualizadas de forma segura. Las extensiones Intel VT-x (2005) y AMD-V (2006) resolvieron este problema al anadir instrucciones especificas para virtualizacion.

En paralelo, VMware (fundada en 1998) lanzo el primer hipervisor comercial para x86. VirtualBox (Sun Microsystems, 2007) y Xen (2003) abrieron la virtualizacion al codigo abierto.

```
Evolucion del hardware para virtualizacion:
1998: VMware Workstation (sin soporte de hardware)
2003: Xen (requiere parches al kernel)
2005: Intel VT-x (virtualizacion por hardware)
2006: AMD-V (virtualizacion por hardware)
2008: KVM (integrado en Linux como modulo del kernel)
```

## Consolidacion y nube (2005-2015)

KVM (Kernel-based Virtual Machine) se integro en el kernel de Linux en 2008, convirtiendo al propio Linux en hipervisor. Esto fue revolucionario: el sistema operativo mas utilizado del mundo se convirtio en la base de la virtualizacion empresarial.

Proxmox VE combino KVM con tecnologias de contenedores y una interfaz web. VMware ESXi se consolido como el hipervisor empresarial de referencia. La virtualizacion se convirtio en la base de los centros de datos modernos.

## La nube y la abstraccion total (2010-presente)

Amazon Web Services (AWS), lanzado en 2006, llevo la virtualizacion a su nivel mas alto de abstraccion: el usuario ya no piensa en servidores fisicos ni hipervisores, solo en instancias bajo demanda. La virtualizacion se convirtio en invisible.

```
Evolucion de la abstraccion:
1960: Hardware fisico (1 SO por servidor)
1970: VMs en mainframes
2000: VMs en x86
2010: IaaS (instancias en la nube)
2015: Contenedores (Docker, Kubernetes)
2020: Serverless (funciones bajo demanda)
```

> La historia de la virtualizacion es la historia de como la informatica aprendio a hacer mas con menos. De un mainframe que servia a una organizacion entera, a un servidor fisico que ejecuta 20 VMs, a un contenedor que ejecuta una funcion en milisegundos. Cada paso aumento la eficiencia y redujo el acoplamiento entre hardware y software.

