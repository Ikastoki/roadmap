# Por que necesitamos orquestacion

Un contenedor aislado en un unico servidor funciona correctamente mientras la carga sea predecible y el numero de servicios sea reducido. En ese escenario, herramientas como Docker CLI o Podman ofrecen todo lo necesario: construir imagenes, iniciar contenedores, gestionar redes y voluepnes. Sin embargo, cuando una aplicacion crece y requiere multiples servicios distribuidos en varios servidores, la gestion manual con comandos individuales se convierte en un cuello de botella critico. Cada actualizacion implica detener un contenedor, eliminarlo, descargar la nueva imagen y relanzarlo, lo que genera tiempo de inactividad no aceptable en entornos productivos. Ademas, si un servidor falla, no existe mecanismo automatico para reubicar los contenedores afectados en otro nodo, lo que obliga al equipo de operaciones a intervenir de forma manual y reactiva.

La orquestacion resuelve estos problemas al proporcionar una capa de abstraccion que gestiona el ciclo de vida completo de los contenedores de forma declarativa e automatica. En lugar de instruir al sistema paso a paso, se define el estado deseado y el orquestador se encarga de alcanzarlo y mantenerlo. Esto incluye programacion automatica de contenedores en los nodos adecuados, deteccion y recuperacion ante fallos, escalado dinamico segun la demanda, balanceo de carga entre instancias y despliegues sin tiempo de inactividad. A continuacion se presentan los problemas concretos que surgen al gestionar contenedores manualmente y como la orquestacion los resuelve.

## El dolor de la gestion manual

Consideremos una aplicacion web basica con cuatro componentes: un servidor web, una API, una base de datos y un cache Redis. Desplegar y mantener estos servicios con Docker CLI en un solo servidor es factible, pero incluso ahi surgen problemas inmediatos.

```bash
# Crear una red para los contenedores
docker network create app-network

# Lanzar la base de datos
docker run -d --name db --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Lanzar el cache
docker run -d --name redis --network app-network redis:7-alpine

# Lanzar la API
docker run -d --name api --network app-network \
  -e DB_HOST=db -e REDIS_HOST=redis \
  myapp-api:2.1.0

# Lanzar el servidor web
docker run -d --name web --network app-network \
  -p 80:8080 \
  myapp-web:2.1.0
```

Cada vez que se actualiza una imagen, es necesario ejecutar `docker stop`, `docker rm` y `docker run` en el orden correcto, con los parametros correctos y con tiempo de inactividad. Si el servidor se reinicia, hay que re-lanzar manualmente todos los contenedores o crear scripts de inicio. Si se desea ejecutar dos instancias de la API para cargar el trafico, hay que levantar un contenedor nginx o haproxy como balanceador y configurar las rutas manualmente. Cada una de estas tareas escala linealmente con el numero de servicios y nodos.

```bash
# Escenario con dos nodos: hay que gestionar cada contenedor en cada nodo
# Nodo 1
ssh nodo1 "docker run -d --name api1 --network app-network myapp-api:2.1.0"
ssh nodo1 "docker run -d --name web1 --network app-network -p 80:8080 myapp-web:2.1.0"

# Nodo 2
ssh nodo2 "docker run -d --name api2 --network app-network myapp-api:2.1.0"
ssh nodo2 "docker run -d --name web2 --network app-network -p 80:8080 myapp-web:2.1.0"

# Si nodo1 falla, hay que detectarlo y reubicar manualmente
ssh nodo2 "docker run -d --name api1_failover --network app-network myapp-api:2.1.0"
```

Este enfoque es frágil, no escalable y propenso a errores humanos. No hay tolerancia a fallos automatica, no hay balanceo de carga nativo y cada operacion requiere intervencion manual.

## Que resuelve la orquestacion

La orquestacion aborda estos problemas a traves de un conjunto de capacidades fundamentales que transforman la gestion de contenedores de un proceso operativo manual a un sistema declarativo y resiliente.

| Capacidad | Gestion manual | Con orquestador |
|---|---|---|
| Recuperacion ante fallos | Deteccion manual, re-lanzamiento manual | Deteccion automatica, re-ubicacion automatica |
| Escalado | Agregar contenedores manualmente, configurar load balancer | Escalado declarativo con reglas automaticas |
| Balanceo de carga | Configurar proxy inverso por separado | Integrado en la capa de red del orquestador |
| Despliegues | Stop + rm + run con downtime | Rolling updates, blue-green, canary automaticos |
| Gestion multi-nodo | SSH a cada servidor individualmente | Panel centralizado con vista del cluster completo |
| Descubrimiento de servicios | IPs hardcodeadas o archivos /etc/hosts | DNS interno automatico por nombre de servicio |
| Configuracion y secretos | Variables de entorno en scripts o .env | Gestor centralizado de configs y secretos |

> En entornos de produccion con multiples servicios interdependientes, un solo contenedor caido puede generar un efecto dominó que afecta a toda la plataforma. La orquestacion garantiza que el sistema converja hacia el estado deseado sin intervencion humana.

La diferencia fundamental entre gestionar contenedores manualmente y utilizar un orquestador es similar a la diferencia entre administrar servidores fisicos uno por uno y utilizar una plataforma de computacion en la nube. En ambos casos, la abstraccion permite que la infraestructura se convierta en un recurso programable y resiliente, en lugar de un conjunto de maquinas que requieren atencion constante.

