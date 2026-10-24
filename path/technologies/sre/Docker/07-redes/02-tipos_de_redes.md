# Tipos de redes en Docker

Docker abstrae la complejidad de la configuracion de red subyacente mediante un modelo de drivers que se seleccionan al crear una red. Cada driver implementa un comportamiento de red distinto, desde el aislamiento completo hasta la integraacion directa con la interfaz fisica del host. La eleccion del driver determina como se asignan las direcciones IP, como se enruta el trafico y que grado de aislamiento existe entre contenedores y entre estos y el host. Conocer las caracteristicas de cada driver es indispensable para disenar la topologia de red que mejor se adapte a los requisitos de seguridad, rendimiento y escalabilidad de cada aplicacion.

## Driver bridge

El driver `bridge` es el predeterminado y el mas utilizado en entornos de un solo host. Crea un puente de red Linux (`linux bridge`) al que se conectan los contenedores, formando una red privada aislada del host y del exterior. Docker gestiona las reglas de iptables necesarias para el enrutamiento y el enmascaramiento NAT.

```bash
# Crear una red bridge personalizada
docker network create --driver bridge mi-red-bridge

# Ejecutar contenedores conectados a ella
docker run -d --name app1 --network mi-red-bridge nginx:alpine
docker run -d --name app2 --network mi-red-bridge alpine sleep 300
```

A diferencia de la red `bridge` por defecto, las redes bridge creadas por el usuario ofrecen resolucion DNS interna basada en el nombre del contenedor, lo que elimina la dependencia de direcciones IP estaticas. Cada red bridge crea una subred aislada; dos contenedores en redes bridge diferentes no pueden comunicarse directamente a menos que se conecten a ambas.

```bash
# Verificar el aislamiento entre redes bridge distintas
docker network create red-a
docker network create red-b
docker run -d --name c-a --network red-a alpine sleep 300
docker run -d --name c-b --network red-b alpine sleep 300
# c-a no puede alcanzar a c-b aunque esten en el mismo host
```

## Driver host

El driver `host` elimina el aislamiento de red entre el contenedor y el host. El contenedor utiliza directamente la pila de red del host: comparte la misma interfaz, las mismas direcciones IP y el mismo espacio de puertos. No hay NAT, no hay puente, no hay asignacion de IP propia.

```bash
# Ejecutar un contenedor con red host
docker run -d --name web-host --network host nginx:alpine
# Nginx escucha directamente en el puerto 80 del host
# No es necesario publicar puertos con -p
```

Este driver ofrece el maximo rendimiento de red, ya que elimina la traduccion de direcciones y el paso a traves del bridge. Sin embargo, sacrifica completamente el aislamiento: cualquier proceso en el contenedor puede escuchar en cualquier puerto del host, y las aplicaciones en el contenedor ven las interfaces de red del host como propias. Es util para aplicaciones que requieren rendimiento de red critico o que necesitan monitorizar trafico de red a nivel de host.

```bash
# El contenedor ve las mismas interfaces que el host
docker run --rm --network host alpine ip addr
# Muestra las interfaces del host, no interfaces virtuales
```

> Usar `--network host` en un Dockerfile con `EXPOSE` no publica puertos automaticamente. `EXPOSE` es meramente documental; con host network, el contenedor ya tiene acceso directo a los puertos del host.

## Driver none

El driver `none` desconecta completamente el contenedor de cualquier red. El contenedor solo dispone de la interfaz loopback (`lo`), sin acceso a otros contenedores, al host ni a redes externas. Es el nivel maximo de aislamiento de red que Docker puede proporcionar.

```bash
# Contenedor sin red
docker run --rm --network none alpine ip addr
# Solo muestra la interfaz loopback: 1: lo: <LOOPBACK,UP,LOWER_UP>
```

Este driver se utiliza para contenedores que procesan datos completamente offline, ejecutan tareas batch que no requieren conectividad, o como medida de seguridad para contenedores que solo necesitan acceso a volumenes locales. Un contenedor con red `none` no puede recibir conexiones ni iniciarlas; esta funcionalmente aislado.

## Driver overlay

El driver `overlay` permite conectar contenedores que se ejecutan en diferentes nodos de un cluster Docker Swarm, creando una red multiplano que abarca multiples hosts. Internamente utiliza túneles VXLAN (Virtual Extensible LAN) para encapsular trafico entre nodos, de modo que los contenedores ven una red plana unificada independientemente del nodo fisico donde residan.

```bash
# Crear una red overlay (requiere Swarm activo o dockerd con swarm habilitado)
docker network create --driver overlay --attachable mi-red-overlay

# Ejecutar contenedores en diferentes nodos del cluster
docker service create --name api --network mi-red-overlay --replicas 3 mi-api:latest
```

El driver overlay implementa cifrado del trafico entre nodos mediante IPSec cuando se habilita la opcion `--opt encrypted`. El rendimiento es inferior al de bridge o host debido a la sobrecarga de encapsulacion VXLAN, pero es la unica opcion nativa de Docker para redes multicluster sin recurrir a soluciones externas como Flannel, Calico o Weave.

```bash
# Crear una red overlay con cifrado
docker network create --driver overlay --opt encrypted red-segura
```

## Driver macvlan

El driver `macvlan` asigna a cada contenedor una direccion MAC real, haciendo que aparezca en la red fisica como un dispositivo independiente. El contenedor recibe una IP del mismo rango que el host y puede ser alcanzado directamente desde la red local sin NAT ni publicacion de puertos.

```bash
# Crear una red macvlan vinculada a la interfaz fisica eth0
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  mi-red-macvlan

# Ejecutar un contenedor que aparece como dispositivo independiente
docker run -d --name web-mac --network mi-red-macvlan nginx:alpine
```

Este driver es util para aplicaciones legacy que esperan conectarse directamente a dispositivos de red fisicos, o para integrar contenedores en redes corporativas existentes donde cada servicio debe tener su propia identidad de red. La desventaja principal es que el host no puede comunicarse directamente con los contenedores en una red macvlan a menos que se configure una subinterfaz adicional, y muchos entornos virtualizados o en la nube publica no permiten direcciones MAC adicionales.

```bash
# El contenedor tiene una MAC unica en la red fisica
docker inspect web-mac --format '{{.NetworkSettings.Networks.mi-red-macvlan.MacAddress}}'
# 02:42:c0:a8:01:02 (ejemplo)
```

## Comparativa de drivers

| Driver | Aislamiento | Caso de uso tipico |
|--------|-------------|-------------------|
| bridge | Alto (NAT) | Aplicaciones monohost |
| host | Ninguno | Rendimiento critico, monitorizacion |
| none | Total | Procesamiento offline, seguridad |
| overlay | Medio (VXLAN) | Clusters multi-nodo |
| macvlan | Bajo (MAC real) | Integracion redes fisicas legacy |

La eleccion del driver de red no es permanente. Un contenedor puede conectarse a multiples redes simultaneamente, combinando, por ejemplo, una red bridge para la comunicacion interna entre microservicios y una red macvlan para la exposicion al exterior sin NAT, o una red overlay para la sincronizacion entre nodos de un cluster.

