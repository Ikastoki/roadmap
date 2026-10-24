# Redes por defecto en Docker

Cuando se instala Docker, el motor crea automaticamente tres redes preconfiguradas: `bridge`, `host` y `none`. De ellas, la red `bridge` es la que utiliza cualquier contenedor lanzado sin una red especifica, y es el escenario donde ocurre la mayoria de la comunicacion entre contenedores en un mismo host. Comprender como funciona esta red por defecto, que aislamiento proporciona y como se comunican los contenedores a traves de ella es fundamental para disenar arquitecturas de contenedores que no dependan de suposiciones erroneas sobre la conectividad.

## La red bridge por defecto

La red `bridge` por defecto (cuyo nombre tecnico es `docker0`) es un puente de red Linux que Docker crea en el host al iniciarse. Todos los contenedores que no especifican una red se conectan a este puente, obteniendo una direccion IP dentro del rango privado de la subred `172.17.0.0/16`. El puente `docker0` actua como un conmutador de nivel 2 entre los contenedores y, al mismo tiempo, como interfaz de enlace con la pila de red del host.

```bash
# Ver la red bridge por defecto
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# a1b2c3d4e5f6   bridge    bridge    local

# Ver los detalles de la red bridge
docker network inspect bridge
```

El comportamiento fundamental de esta red es el siguiente: los contenedores pueden comunicarse entre si a traves de sus direcciones IP asignadas por DHCP interno de Docker, pero no reciben un nombre DNS resoluble. El host puede alcanzar a los contenedores a traves del bridge (o de los puertos publicados), pero desde fuera del host no hay acceso directo a las IPs de los contenedores sin una publicacion explicita de puertos.

```bash
# La subred del bridge por defecto
ip addr show docker0
# 3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
#     inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0

# Las IPs de los contenedores caen dentro de esta subred
docker run -d --name contenedor_a alpine sleep 300
docker inspect contenedor_a --format '{{.NetworkSettings.IPAddress}}'
# 172.17.0.2
```

## Comunicacion entre contenedores por IP

En la red `bridge` por defecto, dos contenedores pueden intercambiar trafico siempre que conozcan sus direcciones IP mutuas. Docker no aplica ninguna politica de aislamiento entre contenedores en la misma red bridge, por lo que cualquier par de contenedores conectados a `docker0` pueden establecer conexiones TCP o UDP sin restricciones adicionales.

```bash
# Crear dos contenedores en la red bridge por defecto
docker run -d --name web --rm nginx:alpine
docker run -d --name cliente --rm alpine sleep 300

# Obtener la IP del contenedor web
WEB_IP=$(docker inspect web --format '{{.NetworkSettings.IPAddress}}')

# Desde el cliente, alcanzar el servidor web por IP
docker exec cliente wget -qO- http://$WEB_IP
# Funciona porque estan en la misma red bridge
```

Sin embargo, la comunicacion por IP presenta limitaciones practicas importantes. Las direcciones IP de los contenedores cambian cada vez que se recrean (al actualizar la imagen, al escalar servicios, al reiniciar el nodo), lo que hace que la configuracion estatica sea fragil y propensa a errores. Docker asigna las IPs en orden ascendente dentro de la subred, pero esta asignacion no es predecible cuando los contenedores se crean y destruyen constantemente.

```bash
# Demostracion: la IP cambia al recrear el contenedor
IP_ORIGINAL=$(docker inspect web --format '{{.NetworkSettings.IPAddress}}')
docker rm -f web
docker run -d --name web nginx:alpine
NUEVA_IP=$(docker inspect web --format '{{.NetworkSettings.IPAddress}}')
echo "Antes: $IP_ORIGINAL, Despues: $NUEVA_IP"
# Ambas IPs son diferentes
```

## NAT y acceso al exterior

La red `bridge` por defecto implementa NAT (Network Address Translation) a traves de `iptables` para permitir que los contenedores accedan a redes externas. Cuando un contenedor envía un paquete a una direccion fuera de la subred del bridge (por ejemplo, a internet), Docker enmascara la direccion origen del contenedor con la direccion IP del host. Esto permite que el contenedor tenga conectividad saliente sin necesidad de configurar rutas adicionales.

```bash
# El contenedor accede a internet a traves de NAT
docker run --rm alpine ping -c 2 8.8.8.8
# Funciona gracias al enmascaramiento NAT de iptables
```

Las reglas de iptables que Docker gestiona se encargan de:
- Permitir trafico entre contenedores en el bridge (reglas `FORWARD` aceptadas)
- Enmascarar el trafico saliente hacia el exterior (regla `MASQUERADE` en `POSTROUTING`)
- Redirigir el trafico entrante en puertos publicados hacia los contenedores (reglas `DNAT` en `PREROUTING`)

```bash
# Ver las reglas iptables que Docker ha creado
iptables -t nat -L -n | grep -i docker
iptables -L FORWARD -n | grep -i docker
```

> La red `bridge` por defecto es funcional para contenedores independientes o entornos de desarrollo simples, pero no ofrece las capacidades de descubrimiento de servicios y aislamiento que proporcionan las redes definidas por el usuario. Docker recomienda crear redes personalizadas para aplicaciones multicontenedor, incluso en entornos de un solo host.

## Limitaciones de la red bridge por defecto

La red `bridge` por defecto carece de resolucion DNS interna automatica. Dos contenedores en la misma red `docker0` no pueden referirse entre si por el nombre del contenedor; deben descubrir la IP por otros medios (inspeccion manual, variables de entorno, o un servicio externo de descubrimiento). Esta limitacion es la razon principal por la que las redes definidas por el usuario, que si incorporan DNS interno, son la opcion recomendada incluso para despliegues monohost.

```bash
# La red bridge por defecto NO resuelve nombres de contenedor
docker run -d --name server nginx:alpine
docker run --rm alpine ping server
# ping: bad address 'server'
```

Otra limitacion relevante es que el aislamiento en la red por defecto es todo o nada. Todos los contenedores conectados al bridge comparten el mismo segmento de red y pueden comunicarse libremente. No es posible crear grupos de contenedores aislados dentro del mismo bridge sin recurrir a tecnicas externas como iptables o sidecars proxy.

