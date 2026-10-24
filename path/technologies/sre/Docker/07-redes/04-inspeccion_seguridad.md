# Inspeccion de redes y seguridad

La gestion de redes en Docker no termina con la creacion de contenedores y la publicacion de puertos. Una vez que los contenedores estan en ejecucion, es necesario verificar la topologia de red real, diagnosticar problemas de conectividad y, sobre todo, garantizar que el aislamiento configurado se corresponde con el comportamiento observado. Docker proporciona herramientas de inspeccion que revelan el estado interno de las redes, mientras que el modelo de seguridad de red se construye sobre la premisa de que el aislamiento debe ser intencional, no accidental.

## Inspeccion con docker network inspect

El comando `docker network inspect` devuelve en formato JSON la configuracion completa de una red, incluyendo los contenedores conectados, sus direcciones IP, la subred, la puerta de enlace y las opciones del driver. Es la herramienta principal para auditar la topologia de red en cualquier momento.

```bash
docker network inspect red-backend
```

La salida incluye, entre otros campos:

- `Name`: nombre de la red.
- `Driver`: driver utilizado (`bridge`, `overlay`, etc.).
- `IPAM.Config`: configuracion de subred, gateway y rango de IPs.
- `Containers`: mapa de contenedores conectados, con sus direcciones IP y MAC.
- `Options`: opciones especificas del driver (como `com.docker.network.bridge.name`).

```json
[
    {
        "Name": "red-backend",
        "Id": "b2c3d4e5f6a7...",
        "Driver": "bridge",
        "IPAM": {
            "Config": [
                {
                    "Subnet": "10.20.0.0/16",
                    "Gateway": "10.20.0.1"
                }
            ]
        },
        "Containers": {
            "abc123...": {
                "Name": "api",
                "IPv4Address": "10.20.1.2/16",
                "MacAddress": "02:42:0a:14:01:02"
            }
        }
    }
]
```

Para consultas rapidas sin procesar todo el JSON, el flag `--format` permite extraer campos especificos:

```bash
# Listar las IPs de todos los contenedores en una red
docker network inspect red-backend \
  --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'

# Obtener la IP de un contenedor especifico
docker inspect api --format '{{.NetworkSettings.Networks.red-backend.IPAddress}}'
```

## Resolucion de problemas de conectividad

El diagnostico de red entre contenedores se realiza con las herramientas clasicas de red, ejecutadas dentro de los propios contenedores. Para ello, los contenedores deben incluir utilidades como `ping`, `nslookup`, `dig`, `telnet` o `curl`, o bien usar una imagen diseñada para diagnostico como `nicolaka/netshoot` o `alpine` con los paquetes necesarios.

```bash
# Usar una imagen de diagnostico
docker run -it --network red-backend nicolaka/netshoot

# Dentro del contenedor de diagnostico
ping api          # Prueba de conectividad ICMP
nslookup db       # Resolucion DNS interna
curl -v http://api:3000/health  # Prueba HTTP
telnet db 5432    # Prueba de puerto TCP
```

La causa mas frecuente de problemas de conectividad es que los contenedores no esten en la misma red. Un contenedor solo puede resolver por nombre y comunicarse con aquellos contenedores que compartan al menos una red definida por el usuario.

```bash
# Verificar las redes a las que esta conectado un contenedor
docker inspect web --format '{{range $net, $v := .NetworkSettings.Networks}}{{$net}}{{"\n"}}{{end}}'
# red-frontend
# red-backend
```

Otra causa comun es la existencia de reglas de firewall en el host que bloquean el trafico entre contenedores. Docker inserta reglas en iptables para la red bridge, pero firewalls como `ufw` pueden interferir con estas reglas si no estan configurados para permitir trafico en el bridge `docker0`.

```bash
# Verificar las reglas de iptables de Docker
iptables -L DOCKER -n -v
iptables -t nat -L DOCKER -n -v

# Permitir trafico en el bridge docker0 (si ufw esta activo)
ufw allow in on docker0
```

## Seguridad de red en Docker

El modelo de seguridad de red de Docker se basa en el principio de aislamiento progresivo. Por defecto, un contenedor tiene acceso saliente a internet (a traves de NAT) y puede comunicarse con otros contenedores en la misma red, pero no recibe trafico entrante del exterior a menos que se publique explicitamente un puerto.

Sin embargo, el comportamiento por defecto no es necesariamente seguro para todos los escenarios. Un contenedor comprometido en la red `bridge` por defecto puede escanear y atacar a cualquier otro contenedor en esa misma red. La estrategia de seguridad consiste en aplicar el principio de minimo privilegio tambien a la red: cada grupo de contenedores debe tener solo la conectividad estrictamente necesaria.

```bash
# Estrategia de redes aisladas para microservicios sensibles
docker network create red-publica
docker network create red-privada

# El frontend solo ve la red publica
docker run -d --name frontend --network red-publica nginx:alpine

# La base de datos solo ve la red privada
docker run -d --name db --network red-privada postgres:16-alpine

# La API actua como puente entre ambas (conectada a las dos)
docker run -d --name api --network red-publica mi-api
docker network connect red-privada api
```

En este esquema, la base de datos es inalcanzable desde el frontend y desde el exterior. Solo la API, que esta conectada a ambas redes, puede comunicarse con la base de datos. Si el frontend es comprometido, el atacante no tiene ruta de red hacia la base de datos.

## Restriccion de conectividad saliente

Por defecto, los contenedores pueden iniciar conexiones salientes a cualquier direccion. Para restringir esta capacidad, es necesario intervenir a nivel de iptables o utilizar redes con driver `none` y conectar solo lo necesario.

```bash
# Bloquear trafico saliente de un contenedor especifico via iptables
iptables -I DOCKER 1 -s 172.17.0.2 -j DROP
# Esto bloquea TODO el trafico del contenedor con IP 172.17.0.2

# Alternativa mas selectiva: bloquear solo trafico saliente a internet
iptables -I FORWARD -s 172.17.0.2 ! -d 10.0.0.0/8 -j DROP
```

Para entornos que requieren politicas de red mas sofisticadas (segmentacion fine-grained, cifrado este-oeste, politicas basadas en identidad), Docker Swarm ofrece redes overlay con cifrado IPSec, y en Kubernetes se implementan Network Policies. En Docker monohost, la combinacion de redes definidas por el usuario y reglas de iptables manuales es la via para lograr un aislamiento completo.

## Buenas practicas de seguridad de red

1. **No usar la red bridge por defecto para aplicaciones multicontenedor**: Crear redes especificas para cada grupo de servicios que deba comunicarse.
2. **Minimizar la superficie de ataque**: Exponer solo los puertos estrictamente necesarios mediante `-p`, y limitar la publicacion a `127.0.0.1` cuando el servicio solo deba ser accesible localmente.
3. **Separar capas en redes distintas**: Base de datos en una red sin acceso a internet, API en una red intermedia, frontend en una red con acceso controlado.
4. **No ejecutar contenedores como root**: Un contenedor comprometido con privilegios de root tiene capacidad de manipular las reglas de iptables y comprometer la red del host.
5. **Auditar periodicamente**: Usar `docker network inspect` y herramientas de escaneo como `docker scout` para verificar que no hay contenedores con conectividad no intencionada.

```bash
# Verificar contenedores expuestos al exterior (con puertos publicados)
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep -v "0.0.0.0"

# Verificar contenedores conectados a la red bridge por defecto
docker network inspect bridge --format '{{range .Containers}}{{.Name}}{{"\n"}}{{end}}'
```

> La seguridad de red en Docker no es automatica; es el resultado de decisiones de diseno conscientes. El hecho de que dos contenedores puedan comunicarse no significa que deban hacerlo. Cada conexion entre contenedores debe ser una decision explicita, no una consecuencia del comportamiento por defecto.

