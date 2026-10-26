# Redes por defecto

Docker crea automaticamente tres redes cuando se instala el motor: `bridge`, `none` y `host`. Cada contenedor se conecta a una red por defecto si no se especifica otra. La red `bridge` es la mas usada: aísla los contenedores del host y entre si, y usa NAT para permitir el acceso a Internet.

## Red bridge por defecto

```bash
# Listar redes
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# abc123def456   bridge    bridge    local
# def789ghi012   host      host      local
# ghi345jkl678   none      null      local

# Inspeccionar bridge
docker network inspect bridge
```

## Como funciona bridge

```
Host (192.168.1.100)
├── bridge: 172.17.0.1
├── contenedor-a: 172.17.0.2
│   └── Puerto 8080 del host ──► NAT ──► contenedor:80
└── contenedor-b: 172.17.0.3
    └── No expuesto al exterior

Internet ◄── NAT ◄── bridge ◄── contenedores
```

```bash
# Ejecutar contenedor en bridge por defecto
docker run -d --name web -p 8080:80 nginx:alpine
# El contenedor obtiene IP 172.17.0.x en la red bridge

# Ver IP del contenedor
docker inspect web --format '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# 172.17.0.2

# El host puede acceder via NAT
curl http://localhost:8080
```

## Red host

```bash
# El contenedor comparte la red del host
docker run -d --name web --network host nginx:alpine
# No hay aislamiento de red
# El contenedor usa directamente 192.168.1.100:80

# Ventaja: rendimiento maximo (sin NAT)
# Desventaja: no hay aislamiento
```

## Red none

```bash
# Sin interfaz de red
docker run -d --name aislado --network none alpine
# El contenedor no tiene IP ni conectividad
# Ideal para procesos que no necesitan red
```

## Limitaciones de bridge por defecto

```bash
# Los contenedores en bridge por defecto NO se resuelven por nombre
docker run --rm alpine ping web
# ping: bad address 'web'

# Solo se resuelven por IP
docker run --rm alpine ping 172.17.0.2

# Para resolucion por nombre, crear una red de usuario
docker network create mi-red
docker run --rm --network mi-red alpine ping web  # Funciona
```

> La red bridge por defecto es el punto de partida para la comunicacion de contenedores, pero tiene limitaciones importantes: no hay resolucion DNS entre contenedores y todos comparten el mismo espacio de direcciones. Para aplicaciones con multiples servicios, es recomendable crear redes de usuario que ofrezcan aislamiento y resolucion por nombre.

