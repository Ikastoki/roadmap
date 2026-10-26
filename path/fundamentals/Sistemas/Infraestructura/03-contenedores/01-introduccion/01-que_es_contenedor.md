# Que es un contenedor

Un contenedor es un entorno de ejecucion aislado, ligero y portable que empaqueta una aplicacion con todas sus dependencias, bibliotecas y configuraciones necesarias para funcionar en cualquier maquina que tenga un motor de contenedores instalado. A diferencia de una maquina virtual, el contenedor no incluye un sistema operativo completo: comparte el kernel del SO anfitrion con los demas contenedores, lo que lo hace mucho mas rapido y eficiente en el uso de recursos.

Esta arquitectura significa que un contenedor puede arrancar en milisegundos, consumir apenas megabytes de RAM y ejecutarse en cualquier lugar: en el portatil de un desarrollador, en un servidor de produccion en la nube o en un dispositivo embebido.

## Arquitectura

```
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Cont. 1  │ │ Cont. 2  │ │ Cont. 3  │
│ (nginx)  │ │ (node.js)│ │ (redis)  │
├──────────┤ ├──────────┤ ├──────────┤
│ Libs/App │ │ Libs/App │ │ Libs/App │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │
     └────────────┼────────────┘
                  ▼
        Motor de contenedores
         (Docker/Podman)
                  │
                  ▼
          Kernel de Linux
          (compartido)
                  │
                  ▼
            Hardware fisico
```

Cada contenedor tiene su propio sistema de archivos aislado, su propia red virtual y sus propios procesos, pero todos comparten el mismo kernel de Linux. Esta es la diferencia clave con las VMs: donde una VM necesita un SO completo (que ocupa GB), un contenedor necesita solo las bibliotecas y archivos de la aplicacion (que ocupan MB).

## Ejemplo basico

```bash
# Ejecutar un servidor web nginx en un contenedor
docker run -d -p 80:80 --name mi-web nginx:alpine

# En segundos, tienes un servidor web funcionando
# Sin instalar nginx en el host
# Sin configurar el SO
# Sin configurar la red manualmente
```

```bash
# Ejecutar una base de datos PostgreSQL
docker run -d \
  --name mi-db \
  -e POSTGRES_PASSWORD=secret \
  -v mi-datos:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16-alpine
```

## Analogia practica

Un contenedor es como un modulo de Lego: cada pieza (contenedor) es autosuficiente, encaja con otras y se puede combinar de diferentes formas. Si una pieza se rompe, la reemplazas sin afectar a las demas. Si necesitas mas piezas iguales, las duplicas en segundos. Y todas funcionan igual sin importar donde las coloques.

> Los contenedores no reemplazan a las VMs: las complementan. Las VMs virtualizan hardware; los contenedores virtualizan el sistema operativo. En la mayoria de entornos modernos, ambos coexisten: VMs como base de infraestructura y contenedores ejecutando las aplicaciones dentro de ellas.

