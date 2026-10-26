# Seguridad de red

La seguridad de red en contenedores se basa en el principio de minimo privilegio: cada contenedor solo debe poder comunicarse con los servicios que necesita. Las redes de usuario, el aislamiento y la no exposicion de puertos innecesarios son las herramientas fundamentales.

## Aislamiento por defecto

```bash
# En bridge por defecto, todos los contenedores se comunican
docker run -d --name web nginx
docker run -d --name db mysql
# web puede acceder a db (no hay aislamiento)

# Con redes de usuario, el aislamiento es automatico
docker network create frontend
docker network create backend
docker run -d --name web --network frontend nginx
docker run -d --name db --network backend mysql
# web NO puede acceder a db
```

## Patron de microservicios seguros

```yaml
# docker-compose.yml
version: '3.8'
services:
  proxy:
    image: nginx:alpine
    ports:
      - "443:443"
    networks:
      - frontend

  web:
    image: nginx:alpine
    networks:
      - frontend
      - backend

  api:
    image: node:20
    networks:
      - backend
    # No expone puertos al exterior

  db:
    image: postgres:16
    networks:
      - backend
    # No expone puertos al exterior
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

networks:
  frontend:
  backend:
    internal: true  # Sin acceso a Internet

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

```
Internet ──► proxy (frontend) ──► web (frontend+backend) ──► api (backend) ──► db (backend)

- proxy: expuesto al exterior (443)
- web: interno, expone puertos internamente
- api: sin exposicion, solo accesible desde web
- db: sin exposicion, solo accesible desde api
```

## Redes internas

```yaml
# Red sin acceso a Internet
networks:
  backend:
    internal: true
# Los contenedores en esta red NO pueden acceder a Internet
# Ideal para bases de datos que no necesitan actualizaciones externas
```

```bash
# Crear red interna
docker network create --internal backend-secure

# Verificar que no hay acceso a Internet
docker run --rm --network backend-secure alpine ping 8.8.8.8
# ping: sendto: Network is unreachable
```

## No exponer puertos innecesarios

```bash
# PELIGROSO: base de datos expuesta al exterior
docker run -d -p 3306:3306 mysql
# Cualquier persona en la red puede acceder a MySQL

# SEGURO: base de datos sin publicar
docker run -d --network mi-app --name db mysql
# Solo accesible desde contenedores en mi-app
```

## Segmentacion de red

```bash
# Tres niveles de red
docker network create public     # Acceso a Internet
docker network create internal   # Solo entre contenedores
docker network create secure     # Solo entre contenedores criticos

# proxy: public + internal
# web: internal
# api: internal + secure
# db: secure
```

> La seguridad de red en contenedores no es diferente a la seguridad de red en servidores tradicionales: segmentacion, minimo privilegio y no exposicion de servicios innecesarios. La diferencia es que las redes de usuario facilitan enormemente la implementacion de estos principios de forma declarativa.

