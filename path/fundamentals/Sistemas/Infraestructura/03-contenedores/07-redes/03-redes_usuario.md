# Redes definidas por el usuario

Las redes de usuario resuelven las limitaciones de la red bridge por defecto: ofrecen resolucion DNS automatica entre contenedores, aislamiento personalizado y control total sobre la configuracion de red. Son la base de la comunicacion entre microservicios en Docker.

## Crear red de usuario

```bash
# Crear red bridge personalizada
docker network create mi-app

# Crear con opciones especificas
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  --ip-range 172.20.5.0/24 \
  mi-app-segura

# Listar redes
docker network ls
# NETWORK ID     NAME            DRIVER    SCOPE
# abc123         mi-app          bridge    local
# def456         mi-app-segura   bridge    local
```

## Resolucion DNS por nombre

```bash
# Crear red y contenedores
docker network create mi-app
docker run -d --name db --network mi-app postgres:16-alpine
docker run -d --name redis --network mi-app redis:7-alpine
docker run -d --name web --network mi-app nginx:alpine

# Ahora los contenedores se resuelven por nombre
docker exec web ping db
# PING db (172.18.0.2): 56 data bytes
# 64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.089 ms

docker exec web ping redis
# PING redis (172.18.0.3): 56 data bytes
# 64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.067 ms
```

## Aislamiento entre redes

```bash
# Dos redes aisladas
docker network create frontend
docker network create backend

# Servicios en cada red
docker run -d --name web --network frontend nginx:alpine
docker run -d --name api --network frontend --network backend node:20
docker run -d --name db --network backend postgres:16

# web NO puede acceder a db (estan en redes diferentes)
# api puede acceder a web y db (esta en ambas redes)

docker exec web ping db
# ping: bad address 'db'

docker exec api ping db
# PING db (172.19.0.2): 56 data bytes
# Funciona porque api esta en la red backend
```

## Conectar y desconectar contenedores

```bash
# Conectar contenedor a red adicional
docker network connect backend web
# web ahora esta en frontend Y backend

# Desconectar contenedor de red
docker network disconnect backend web
# web vuelve a estar solo en frontend
```

## Inspeccionar red

```bash
# Ver configuracion de la red
docker network inspect mi-app
# [
#   {
#     "Name": "mi-app",
#     "Driver": "bridge",
#     "IPAM": {
#       "Config": [
#         {
#           "Subnet": "172.18.0.0/16",
#           "Gateway": "172.18.0.1"
#         }
#       ]
#     },
#     "Containers": {
#       "abc123": { "Name": "db", "IPv4Address": "172.18.0.2/16" },
#       "def456": { "Name": "web", "IPv4Address": "172.18.0.3/16" }
#     }
#   }
# ]

# Ver contenedores en una red
docker network inspect mi-app --format '{{range .Containers}}{{.Name}} {{end}}'
# db web
```

## Eliminar red

```bash
# Eliminar red (solo si no hay contenedores conectados)
docker network rm mi-app

# Eliminar todas las redes sin usar
docker network prune
```

> Las redes de usuario son la base de la comunicacion entre microservicios en Docker. La resolucion DNS por nombre elimina la necesidad de gestionar IPs manualmente, y el aislamiento entre redes permite crear topologias de seguridad que separan los servicios criticos de los menos sensibles.

