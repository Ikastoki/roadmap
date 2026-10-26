# Inspeccion y diagnostico de red

El diagnostico de red en contenedores requiere herramientas para verificar conectividad, resolucion DNS y configuracion de red. Cuando un contenedor no puede comunicarse con otro, el problema suele estar en la red, no en la aplicacion.

## Verificar conectividad

```bash
# Verificar conectividad desde un contenedor
docker exec web ping -c 3 db
# PING db (172.18.0.2): 56 data bytes
# 64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.089 ms
# 64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.067 ms
# 64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.058 ms

# Si ping no funciona, verificar que los contenedores estan en la misma red
docker network inspect mi-app --format '{{range .Containers}}{{.Name}}:{{.IPv4Address}} {{end}}'
# db:172.18.0.2/16 web:172.18.0.3/16
```

## Verificar resolucion DNS

```bash
# Verificar que el nombre resuelve
docker exec web nslookup db
# Name:      db
# Address 1: 172.18.0.2

# Si nslookup no esta instalado, instalarlo
docker exec web apk add bind-tools
docker exec web nslookup db
```

## Herramientas de diagnostico

```bash
# Instalar herramientas de red en un contenedor
docker exec web apk add curl bind-tools net-tools iputils

# Verificar puertos abiertos
docker exec web curl -v http://db:5432
# Si la conexion es rechazada, el puerto no esta escuchando

# Ver interfaces de red
docker exec web ip addr show
# eth0: inet 172.18.0.3/16

# Ver tabla de enrutamiento
docker exec web ip route
# default via 172.18.0.1 dev eth0

# Ver conexiones activas
docker exec web netstat -tlnp
```

## Contenedor de diagnostico

```bash
# Ejecutar contenedor temporal con herramientas de red
docker run --rm -it --network mi-app nicolaka/netshoot
# netshoot incluye: ping, curl, wget, nslookup, dig, traceroute, tcpdump, etc.

# Dentro de netshoot
ping db
curl http://web:80
dig db
traceroute db
```

## Problemas comunes

```bash
# Problema 1: contenedores en diferentes redes
docker network inspect frontend --format '{{range .Containers}}{{.Name}} {{end}}'
# web api

docker network inspect backend --format '{{range .Containers}}{{.Name}} {{end}}'
# db

# Solucion: conectar el contenedor a la otra red
docker network connect backend web

# Problema 2: resolucion DNS no funciona
docker exec web ping db
# ping: bad address 'db'

# Solucion: crear red de usuario (bridge por defecto no tiene DNS)
docker network create mi-app
docker run --rm --network mi-app alpine ping db
```

## Inspeccionar configuracion de red del contenedor

```bash
# Ver configuracion de red completa
docker inspect web --format '{{json .NetworkSettings.Networks}}'
# {
#   "mi-app": {
#     "IPAMConfig": null,
#     "IPAddress": "172.18.0.3",
#     "Gateway": "172.18.0.1",
#     "DNS": ["127.0.0.11"]
#   }
# }

# Ver puertos mapeados
docker inspect web --format '{{json .NetworkSettings.Ports}}'
# {"80/tcp":[{"HostIp":"0.0.0.0","HostPort":"8080"}]}
```

> El diagnostico de red en contenedores sigue el mismo patron que en servidores tradicionales: verificar conectividad (ping), resolucion DNS (nslookup/dig) y puertos (curl/netstat). La diferencia es que los contenedores estan en redes virtuales, por lo que la inspeccion de la configuracion de red es el primer paso antes de asumir un problema de la aplicacion.

