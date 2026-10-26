# Contenedores vs. maquinas virtuales

La comparacion mas relevante en infraestructura moderna. Contenedores y VMs son mecanismos de aislamiento y ejecucion, pero operan en niveles fundamentalmente diferentes de la pila de tecnologia. Comprender esta diferencia es clave para elegir la herramienta adecuada para cada escenario.

## Diferencia de arquitectura

```
MAQUINA VIRTUAL:
┌─────────┐ ┌─────────┐
│   VM 1  │ │   VM 2  │
│ Windows │ │  Linux  │
├─────────┤ ├─────────┤
│ Kernel  │ │ Kernel  │  ← Cada VM tiene su propio kernel
│  SO     │ │  SO     │
├─────────┴─┴─────────┤
│    Hipervisor        │
├─────────────────────┤
│    Host OS           │
├─────────────────────┤
│    Hardware          │
└─────────────────────┘

CONTENEDOR:
┌─────────┐ ┌─────────┐
│ Cont. 1 │ │ Cont. 2 │
│ nginx   │ │ node.js │
├─────────┤ ├─────────┤
│ Libs    │ │ Libs    │
└────┬────┘ └────┬────┘
     └─────┬─────┘
           ▼
    Motor de contenedores
           │
           ▼
      Kernel de Linux     ← Kernel compartido
           │
           ▼
       Hardware
```

## Comparacion detallada

| Aspecto | Maquina virtual | Contenedor |
|---|---|---|
| Aislamiento | Completo (kernel separado) | Por proceso (kernel compartido) |
| Arranque | 30-60 segundos | Milisegundos |
| Tamano | 1-10 GB | 10-500 MB |
| Sobrecarga | 15-30% del hardware | 1-5% del hardware |
| Densidad | 10-50 por servidor | 100-1000+ por servidor |
| SO invitado | Cualquier SO | Mismo SO que el host |
| Portabilidad | Depende del formato de disco | Altamente portable |
| Seguridad | Mayor (kernel separado) | Menor (kernel compartido) |
| Gestion de estado | Persistente | Efímero por defecto |

## Rendimiento

```bash
# Benchmark conceptual: ejecutar nginx
# Nativo en el host:       50,000 req/s (100%)
# En contenedor Docker:    49,500 req/s (99%)
# En VM con KVM:           43,000 req/s (86%)
# En VM con VirtualBox:    38,000 req/s (76%)
```

Los contenedores tienen practicamente el mismo rendimiento que el codigo nativo. Las VMs tienen una capa de sobrecarga significativa por el hipervisor y el SO invitado.

## Cuándo usar VMs

- Necesitas ejecutar Windows y Linux en el mismo servidor.
- Software legacy que requiere un SO especifico.
- Multi-tenancy con aislamiento completo entre clientes.
- Cargas de trabajo que necesitan control total del SO.
- Seguridad critica donde el aislamiento de kernel es imperativo.

## Cuándo usar contenedores

- Microservicios que se despliegan frecuentemente.
- CI/CD con entornos efimeros.
- Aplicaciones cloud-native disenadas para contenedores.
- Maximizar densidad de aplicaciones en un servidor.
- Desarrollo local replicable.

## Complementarios, no excluyentes

En la practica, ambos coexisten:

```
Servidor fisico o instancia en nube (VM)
  └── Motor de contenedores (Docker/Podman)
        ├── Contenedor: API REST
        ├── Contenedor: Base de datos
        ├── Contenedor: Cache
        └── Contenedor: Proxy inverso
```

Kubernetes, el orquestador mas popular, frecuentemente se despliega sobre VMs en la nube, y las aplicaciones corren en contenedores dentro de esas VMs.

> La pregunta no es "VMs o contenedores?" sino "que nivel de aislamiento y que nivel de eficiencia necesito para cada carga de trabajo?". Un entorno bien disenado usa VMs para la base de infraestructura y contenedores para las aplicaciones.

