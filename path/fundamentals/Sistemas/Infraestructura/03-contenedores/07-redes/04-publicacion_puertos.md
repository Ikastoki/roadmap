# Publicacion de puertos

La publicacion de puertos es el mecanismo que conecta el mundo exterior con los contenedores. Sin publicar puertos, los contenedores solo son accesibles desde otros contenedores en la misma red. La publicacion usa NAT para redirigir un puerto del host a un puerto del contenedor.

## Sintaxis basica

```bash
# Publicar puerto 80 del contenedor en el puerto 8080 del host
docker run -d --name web -p 8080:80 nginx:alpine
# http://localhost:8080 ──► contenedor:80

# Publicar multiples puertos
docker run -d --name web -p 8080:80 -p 8443:443 nginx:alpine

# Publicar todos los puertos expuestos (peligroso)
docker run -d --name web -P nginx:alpine
# Asigna puertos aleatorios del host a todos los puertos EXPOSE
```

## Publicar en IP especifica

```bash
# Solo en localhost (no accesible desde la red)
docker run -d --name web -p 127.0.0.1:8080:80 nginx:alpine

# En una IP especifica
docker run -d --name web -p 192.168.1.100:8080:80 nginx:alpine

# En todas las interfaces (default)
docker run -d --name web -p 8080:80 nginx:alpine
```

## Puertos TCP y UDP

```bash
# TCP (por defecto)
docker run -d --name web -p 8080:80 nginx:alpine

# UDP
docker run -d --name dns -p 5353:53/udp dns-server

# TCP y UDP en el mismo puerto
docker run -d --name dns -p 5353:53/tcp -p 5353:53/udp dns-server

# Usando la sintaxis explicita
docker run -d --name web \
  --publish published=8080,target=80,protocol=tcp \
  nginx:alpine
```

## Publicacion en Docker Compose

```yaml
version: '3.8'
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"        # host:contenedor
      - "8443:443"
      - "127.0.0.1:3000:3000"  # Solo localhost

  api:
    image: node:20
    ports:
      - "3001:3000"
```

## Publicacion vs. exposicion

```bash
# EXPOSE (en Dockerfile): documenta puertos, NO los publica
# Dockerfile:
# EXPOSE 80
# EXPOSE 443

# -p (al ejecutar): publica puertos del host
docker run -d -p 8080:80 nginx:alpine
# EXPOSE solo es documentacion; -p es la accion real
```

```bash
# Ver puertos publicados
docker port web
# 80/tcp -> 0.0.0.0:8080
# 443/tcp -> 0.0.0.0:8443
```

## Seguridad en publicacion de puertos

```bash
# MAL: expuesto a todas las interfaces
docker run -d -p 3306:3306 mysql

# BIEN: solo localhost
docker run -d -p 127.0.0.1:3306:3306 mysql

# Mejor: no publicar puertos, usar red de usuario
docker network create mi-app
docker run -d --network mi-app --name db mysql
docker run -d --network mi-app --name app mi-app
# db es accesible desde app pero NO desde el exterior
```

> La publicacion de puertos es el puente entre los contenedores y el mundo exterior. Publicar solo los puertos necesarios, en la IP correcta y con el protocolo adecuado, es una practica de seguridad fundamental. Para servicios internos como bases de datos, es preferible no publicar puertos y usar redes de usuario para la comunicacion entre contenedores.

