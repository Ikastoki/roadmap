# Virtualizacion vs. contenedores

La comparacion mas relevante en la infraestructura moderna. Las maquinas virtuales y los contenedores son mecanismos de aislamiento y ejecucion de aplicaciones, pero operan en niveles fundamentales diferentes de la pila de tecnologia. VMs virtualizan hardware; contenedores virtualizan el sistema operativo. Esta diferencia tiene implicaciones directas en rendimiento, portabilidad, seguridad y complejidad operativa.

## Arquitectura comparada

```
VIRTUALIZACION (VMs):
┌────────┐ ┌────────┐
│  VM 1  │ │  VM 2  │
│ Linux  │ │Windows │
├────────┤ ├────────┤
│  Libs  │ │  Libs  │
│  App   │ │  App   │
├────────┴─┴────────┤
│  Hipervisor       │
├───────────────────┤
│  Host OS          │
├───────────────────┤
│  Hardware         │
└───────────────────┘

CONTENEDORES:
┌────────┐ ┌────────┐
│Cont. 1 │ │Cont. 2 │
│ Linux  │ │ Linux  │
├────────┤ ├────────┤
│  Libs  │ │  Libs  │
│  App   │ │  App   │
├────────┴─┴────────┤
│  Motor de cont.   │
│  (Docker/Podman)  │
├───────────────────┤
│  Host OS (kernel) │
├───────────────────┤
│  Hardware         │
└───────────────────┘
```

## Diferencias fundamentales

| Aspecto | Maquina virtual | Contenedor |
|---|---|---|
| Aislamiento | Completo (SO separado) | Por proceso (comparte kernel) |
| Arranque | Minutos | Milisegundos |
| Tamano | Gigabytes | Megabytes |
| Sobrecarga | Alta (SO completo) | Minima (solo libs y app) |
| Densidad | 10-50 por servidor | 100-1000 por servidor |
| Portabilidad | Depende del formato | Altamente portable (imagen) |
| Seguridad | Mayor (kernel separado) | Menor (kernel compartido) |
| SO invitado | Cualquier SO | Mismo SO que el host |

## Rendimiento

Los contenedores tienen practicamente el mismo rendimiento que el codigo ejecutando nativamente. Las VMs tienen una capa de sobrecarga por el hipervisor y el SO invitado.

```bash
# Benchmark conceptual
# Ejecutar nginx nativo:     50,000 req/s
# Ejecutar nginx en cont.:   49,500 req/s  (1% overhead)
# Ejecutar nginx en VM:      42,000 req/s  (16% overhead)
```

## Portabilidad

Los contenedores son nativos de la nube: una imagen Docker corre igual en un portatil, en un servidor local y en cualquier proveedor de nube. Las VMs requieren conversion de formato y pueden tener diferencias de drivers.

```bash
# Contenedor: funciona igual en cualquier lugar
docker run -d -p 80:80 nginx:latest

# VM: requiere configurar el SO, instalar nginx, configurar red...
# Y el formato del disco puede no ser compatible entre hipervisores
```

## Seguridad

Las VMs proporcionan mayor aislamiento porque cada una tiene su propio kernel. Si un atacante compromete el kernel de una VM, no afecta al host ni a otras VMs. Los contenedores comparten el kernel del host, lo que significa que una vulnerabilidad en el kernel puede comprometer todos los contenedores.

## Caso ideal de uso

**VMs para:**
- Sistemas operativos heterogeneos (Linux y Windows en el mismo servidor).
- Software legacy que requiere un SO especifico.
- Cargas de trabajo que necesitan maximo aislamiento (multi-tenancy).
- Entornos donde la seguridad es la maxima prioridad.

**Contenedores para:**
- Microservicios que se despliegan frecuentemente.
- CI/CD y entornos de prueba efimeros.
- Aplicaciones cloud-native diseñadas para contenedores.
- Maximizar densidad de aplicaciones en un servidor.

> La eleccion entre VMs y contenedores no es mutuamente excluyente. En la mayoria de entornos modernos, ambos coexisten: VMs proporcionan la capa de infraestructura base y los contenedores ejecutan las aplicaciones dentro de ellas. Kubernetes, el orquestador de contenedores mas popular, frecuentemente se despliega sobre VMs en la nube.

