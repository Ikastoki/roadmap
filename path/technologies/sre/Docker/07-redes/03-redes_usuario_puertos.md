# Redes definidas por el usuario y publicacion de puertos

Las redes creadas por el usuario resuelven las limitaciones fundamentales de la red `bridge` por defecto: la falta de descubrimiento de servicios por nombre y el aislamiento excesivamente permisivo entre todos los contenedores. En combinacion con la publicacion de puertos, proporcionan el mecanismo preciso para controlar que contenedores se comunican entre si y como se exponen sus servicios al exterior. Estas dos capacidades, usadas conjuntamente, constituyen la base de la topologia de red en despliegues Docker de cualquier tamano.

## Creacion de redes con docker network create

El comando `docker network create` permite instanciar redes con control total sobre el driver, la subred, la puerta de enlace y otras opciones especificas de cada driver. Cada red creada es un objeto gestionable de forma independiente: se puede listar, inspeccionar, conectar y desconectar contenedores, y eliminar sin afectar a otras redes.

```bash
# Crear una red bridge personalizada con parametros explicitos
docker network create \
  --driver bridge \
  --subnet 10.20.0.0/16 \
  --gateway 10.20.0.1 \
  --ip-range 10.20.1.0/24 \
  red-backend

# Listar las redes disponibles
docker network ls
# NETWORK ID     NAME          DRIVER    SCOPE
# b2c3d4e5f6a7   red-backend   bridge    local
```

Las redes definidas por el usuario ofrecen dos ventajas criticas sobre la red bridge por defecto. La primera es el **aislamiento logico de grupos de contenedores**: se pueden crear tantas redes como sea necesario y conectar solo los contenedores que deben comunicarse. Un contenedor de base de datos puede estar en una red interna sin acceso a internet, mientras que el frontend se conecta a una red diferente.

```bash
# Redes aisladas para diferentes capas de la aplicacion
docker network create red-frontend
docker network create red-backend
docker network create red-base-datos

# Cada contenedor se conecta solo a las redes necesarias
docker run -d --name api --network red-backend mi-api
docker network connect red-frontend api  # La API tambien expone frontend
```

La segunda ventaja es que un contenedor puede estar conectado a **multiples redes simultaneamente**, actuando como puente entre ellas. Esto es esencial para patrones como el de proxy reverso o API gateway, donde un contenedor necesita estar en la red publica y en la red privada simultaneamente.

## DNS interno y resolucion por nombre de contenedor

Las redes definidas por el usuario incorporan un servidor DNS interno integrado en el demonio de Docker. Cuando un contenedor intenta resolver un nombre de host, Docker consulta su registro interno y devuelve la direccion IP del contenedor destino si ambos comparten al menos una red definida por el usuario. Este DNS resuelve nombres de contenedor, nombres de servicio (en Swarm) y alias de red.

```bash
# Crear una red y dos contenedores
docker network create red-app

docker run -d --name db --network red-app -e POSTGRES_PASSWORD=secret postgres:16-alpine
docker run -d --name web --network red-app nginx:alpine

# Resolver por nombre de contenedor
docker exec web nslookup db
# Server:    127.0.0.11
# Address:   127.0.0.11#53
# Name:      db
# Address:   10.20.1.2
```

El servidor DNS interno escucha en `127.0.0.11` dentro del contenedor y actua como forwarder: las consultas que no corresponden a contenedores en la misma red se delegan a los DNS configurados en el host. Esto significa que los contenedores pueden seguir resolviendo nombres de internet sin configuracion adicional.

```bash
# El DNS interno tambien resuelve nombres externos
docker exec web nslookup google.com
# Server:    127.0.0.11
# Address:   127.0.0.11#53
# Non-authoritative answer:
# Name:      google.com
# Address:   142.250.184.46
```

La resolucion por nombre de contenedor funciona exclusivamente dentro de redes definidas por el usuario. En la red `bridge` por defecto, el DNS interno no esta disponible, lo que obliga a usar direcciones IP o el flag `--link` (deprecado). Esta es la razon principal por la que incluso en entornos de desarrollo local se recomienda crear una red personalizada.

```bash
# Un contenedor en la red por defecto NO resuelve nombres
docker run -d --name server nginx:alpine
docker run --rm alpine ping server
# ping: bad address 'server'

# En una red creada por el usuario, funciona
docker network create red-ejemplo
docker run -d --name server --network red-ejemplo nginx:alpine
docker run --rm --network red-ejemplo alpine ping server
# PING server (10.20.0.2): 56 data bytes
```

## Publicacion de puertos con -p

La publicacion de puertos es el mecanismo que permite que el trafico procedente de fuera del host (desde otras maquinas, desde internet, desde el propio host pero fuera del contexto de Docker) alcance a un contenedor. Se configura con el flag `-p` (o `--publish`) de `docker run` y crea una regla DNAT en iptables que redirige el trafico de un puerto del host al puerto del contenedor.

```bash
# Sintaxis: -p <puerto_host>:<puerto_contenedor>
docker run -d --name web -p 8080:80 nginx:alpine

# Acceso externo: http://localhost:8080 redirige a nginx en el puerto 80
curl localhost:8080
# Respuesta de nginx
```

El mapeo puede ser especifico o ephemeral. Si solo se indica el puerto del contenedor, Docker asigna un puerto aleatorio en el host, visible con `docker ps` o `docker port`.

```bash
# Puerto ephemeral: Docker asigna un puerto libre aleatorio
docker run -d -p 80 nginx:alpine
docker port $(docker ps -lq)
# 80/tcp -> 0.0.0.0:32768

# Mapeo a una IP especifica del host
docker run -d --name web -p 127.0.0.1:8080:80 nginx:alpine
# Solo accesible desde el propio host
```

Es importante entender que la publicacion de puertos solo es necesaria para el **acceso externo** a los contenedores. La comunicacion entre contenedores en la misma red Docker no requiere publicacion de puertos: dos contenedores en la misma red pueden alcanzarse directamente por IP o por nombre de contenedor sin pasar por el host.

```bash
# Comunicacion interna sin publicacion de puertos
docker network create backend
docker run -d --name api --network backend -p 3000:3000 mi-api
docker run -d --name worker --network backend alpine sleep 300

# El worker accede a la API directamente por el puerto 3000 del contenedor
docker exec worker wget -qO- http://api:3000/health
# No hace falta publicar el puerto para que worker acceda
```

El flag `-p` tambien acepta el rango de puertos y el protocolo (TCP/UDP), permitiendo mapeos complejos para aplicaciones que utilizan multiples puertos o protocolos.

```bash
# Mapeo de rangos de puertos
docker run -d --name app -p 3000-3005:3000-3005 mi-app

# Mapeo especifico por protocolo
docker run -d --name dns -p 53:53/udp -p 53:53/tcp coredns/coredns
```

> La publicacion de puertos no debe confundirse con la instruccion `EXPOSE` del Dockerfile. `EXPOSE` es meramente documental: indica que el contenedor escucha en un puerto, pero no lo publica. El `-p` en `docker run` es quien realmente crea la regla de redireccion.

