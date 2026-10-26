# Tipos de redes

Docker ofrece cinco tipos de redes para diferentes necesidades de aislamiento, rendimiento y comunicacion. Cada tipo resuelve un problema especifico: desde el aislamiento total hasta la comunicacion entre hosts en un cluster.

## Bridge

```bash
# Red bridge por defecto (NAT)
docker run -d --name web -p 8080:80 nginx:alpine

# Red bridge de usuario (resolucion DNS)
docker network create mi-app
docker run -d --name web --network mi-app nginx:alpine
docker run -d --name app --network mi-app node:20
# web y app se comunican por nombre: "web" resuelve a la IP del contenedor
```

**Caracteristicas:**
- Aislamiento entre contenedores.
- Resolucion DNS en redes de usuario.
- NAT para acceso a Internet.
- Rendimiento bueno (overhead de NAT).
- Ideal para la mayoria de aplicaciones.

## Host

```bash
# Comparte la red del host
docker run -d --name web --network host nginx:alpine
# Sin NAT, sin aislamiento
# El contenedor escucha directamente en 192.168.1.100:80

# Ventaja: rendimiento maximo (sin overhead de red)
# Desventaja: no hay aislamiento, todos los contenedores comparten puertos
```

**Caracteristicas:**
- Sin aislamiento de red.
- Sin overhead de NAT (rendimiento maximo).
- Los puertos del contenedor estan en el host.
- Ideal para: monitoreo, proxy inverso de alto rendimiento.

## None

```bash
# Sin interfaz de red
docker run -d --name aislado --network none alpine
# Solo loopback (127.0.0.1)
# Sin acceso a Internet ni a otros contenedores
```

**Caracteristicas:**
- Sin conectividad de red.
- Solo interfaz loopback.
- Ideal para: procesos que no necesitan red, tareas de procesamiento aislado.

## Overlay

```bash
# Comunicacion entre hosts (requiere swarm o Kubernetes)
docker swarm init
docker network create --driver overlay mi-overlay

# Los contenedores en diferentes hosts se comunican
# como si estuvieran en la misma red local
```

**Caracteristicas:**
- Comunicacion entre contenedores en diferentes hosts.
- Encriptacion automatica (con --opt encrypted).
- Ideal para: Docker Swarm, aplicaciones multi-host.

## Macvlan

```bash
# Asigna MAC real al contenedor
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  mi-macvlan

docker run -d --network mi-macvlan --ip 192.168.1.100 nginx:alpine
# El contenedor aparece como un dispositivo fisico en la red
```

**Caracteristicas:**
- MAC real, IP real.
- Aparece como dispositivo fisico en la red.
- Ideal para: migraciones de aplicaciones legacy, redes que requieren MAC real.

## Comparacion

| Tipo | Aislamiento | DNS | Rendimiento | Uso |
|---|---|---|---|---|
| bridge | Si | Si (usuario) | Bueno | General |
| host | No | N/A | Maximo | Proxy alto rendimiento |
| none | Total | No | N/A | Procesos aislados |
| overlay | Si | Si | Bueno | Multi-host |
| macvlan | Si | Si | Alto | Dispositivos fisicos |

> La eleccion del tipo de red depende del nivel de aislamiento, la necesidad de resolucion DNS y el rendimiento requerido. Para la mayoria de aplicaciones, la red bridge de usuario es suficiente. Para clusters multi-host, overlay es la opcion natural.

