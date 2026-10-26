# Inspeccion de imagenes

Inspeccionar una imagen permite entender exactamente que contiene: que SO base usa, que paquetes tiene instalados, que variables de entorno define, que puertos expone y que comando ejecuta al iniciar. Esta informacion es critica para diagnosticar problemas, verificar seguridad y planificar la optimizacion de imagenes.

## Inspeccion basica

```bash
# Ver metadatos completos de una imagen
docker inspect nginx:alpine
# Retorna un JSON con toda la informacion de la imagen

# Filtrar informacion especifica
docker inspect --format='{{.Os}}/{{.Architecture}}' nginx:alpine
# linux/amd64

docker inspect --format='{{.Config.Env}}' nginx:alpine
# [PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin NGINX_VERSION=1.25.4]

docker inspect --format='{{.Config.ExposedPorts}}' nginx:alpine
# map[80/tcp:{}]

docker inspect --format='{{.Config.Cmd}}' nginx:alpine
# [nginx -g daemon off;]

docker inspect --format='{{.Config.WorkingDir}}' nginx:alpine
# (vacio = directorio raiz)
```

## Ver capas de una imagen

```bash
# Historial de capas
docker history nginx:alpine
# IMAGE          CREATED      CREATED BY                                      SIZE
# a1b2c3d4e5f6   2 weeks ago  /bin/sh -c #(nop) CMD ["nginx" "-g" "daemon…   0B
# f6e5d4c3b2a1   2 weeks ago  /bin/sh -c #(nop) COPY file:…                   2.58kB
# 89d71d5a5c4b   2 weeks ago  /bin/sh -c #(nop) STOPSIGNAL SIGQUIT            0B
# ...
# 567890abcdef   3 months ago /bin/sh -c #(nop) ADD file:… in /              7.74MB

# Ver tamano de cada capa
docker history --format "{{.Size}}" nginx:alpine
# 0B
# 2.58kB
# 0B
# 1.41kB
# ...
# 7.74MB
```

## Herramientas de analisis

**dive:** herramienta para explorar el contenido de cada capa de una imagen.

```bash
# Instalar dive
# Linux
wget https://github.com/wagoodman/dive/releases/download/v0.12.0/dive_0.12.0_linux_amd64.deb
sudo dpkg -i dive_0.12.0_linux_amd64.deb

# Usar dive para analizar una imagen
dive nginx:alpine
# Muestra una vista dual:
# Izquierda: arbol de archivos de cada capa
# Derecha: tamano, eficiencia y archivos modificados

# Analizar eficiencia de la imagen
dive nginx:alpine --ci
# Image efficiency score: 85.3%
# Size: 42.5MB
# Optimizable: 6.3MB (15% de ahorro potencial)
```

**docker scout:** analisis de vulnerabilidades integrado en Docker.

```bash
# Escaneo de vulnerabilidades
docker scout cves nginx:alpine
# Vulnerabilities
# ┌──────────┬───────────┬──────────┬──────────────────┐
# │ Severity │ Package   │ Version  │ Fix              │
# ├──────────┼───────────┼──────────┼──────────────────┤
# │ HIGH     │ openssl   │ 3.1.4-r0 │ 3.1.4-r1         │
# │ MEDIUM   │ libcrypto │ 3.1.4-r0 │ 3.1.4-r1         │
# └──────────┴───────────┴──────────┴──────────────────┘
```

## Inspeccionar una imagen en ejecucion

```bash
# Ejecutar contenedor y explorar su interior
docker run -it --rm nginx:alpine sh
# Dentro del contenedor:
cat /etc/os-release        # Ver SO base
ls -la /etc/nginx/         # Ver configuracion nginx
ps aux                     # Ver procesos
netstat -tlnp              # Ver puertos abiertos
exit
```

## Obtener informacion sin ejecutar

```bash
# Ver el comando por defecto
docker inspect --format='{{json .Config.Cmd}}' nginx:alpine
# ["nginx","-g","daemon off;"]

# Ver las capas como JSON
docker inspect --format='{{json .RootFS.Layers}}' nginx:alpine
# ["sha256:567890abcdef...","sha256:abcdef123456..."]

# Ver los labels (etiquetas de metadatos)
docker inspect --format='{{json .Config.Labels}}' nginx:alpine
# null
```

> Inspeccionar imagenes antes de desplegarlas es una practica de seguridad y eficiencia. Saber que hay dentro de una imagen, que version de software ejecuta y que puertos expone permite tomar decisiones informadas sobre su uso en produccion.

