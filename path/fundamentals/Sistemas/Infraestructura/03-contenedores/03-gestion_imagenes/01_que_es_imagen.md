# Que es una imagen de contenedor

Una imagen de contenedor es una plantilla inmutable, de solo lectura y portable que contiene todo lo necesario para ejecutar una aplicacion: el sistema de archivos, las dependencias, las configuraciones y el codigo. Las imagenes son el molde a partir del cual se crean contenedores en ejecucion. Son inmutables, lo que significa que una vez construidas, no se pueden modificar; cualquier cambio genera una nueva imagen.

Esta inmutabilidad es una de las propiedades mas valiosas de las imagenes: garantiza que la imagen que se probo es exactamente la misma que se despliega en produccion.

## Estructura de una imagen

```
Imagen nginx:alpine
├── Capa 1 (base): Alpine Linux (~7.74 MB)
│   └── Sistema de archivos base minimo
├── Capa 2: Dependencias del sistema (~2.3 MB)
│   └── libnginx, libpcre, openssl
├── Capa 3: Configuracion nginx (~5.8 KB)
│   └── nginx.conf, mime.types
├── Capa 4: Metadatos
│   └── CMD ["nginx", "-g", "daemon off;"]
│   └── EXPOSE 80
│   └── ENV NGINX_VERSION=1.25.4
└── Total: ~10 MB
```

Cada capa es de solo lectura y se apila sobre la anterior. Cuando un contenedor se crea desde la imagen, se añade una capa de escritura fina encima de todas las capas de solo lectura.

## Imagen vs. contenedor

| Aspecto | Imagen | Contenedor |
|---|---|---|
| Estado | Inmutable, de solo lectura | Mutable, en ejecucion |
| Almacenamiento | En disco o registry | En memoria y disco |
| Ubicacion | Registry o local | Host en ejecucion |
| Modificacion | No se puede modificar | Se puede escribir |
| Crear | `docker build` | `docker run` |
| Versionar | Tags y digest | IDs generados |

## Imagenes oficiales vs. comunitarias

- **Oficiales:** mantenidas por los autores del software o por Docker. Verificadas, seguras y actualizadas regularmente.
- **Comunitarias:** creadas por usuarios de la comunidad. Pueden ser excelentes o inseguras. Siempre verificar el autor y las actualizaciones.

```bash
# Imagen oficial de nginx
docker pull nginx              # Equivalente a docker.io/library/nginx:latest

# Imagen comunitaria
docker pull usuario/mi-app     # Imagen de un usuario especifico
```

## Almacenamiento local

Las imagenes se almacenan localmente en el directorio del motor de contenedores:

```bash
# Ver imagenes locales
docker images
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# nginx        alpine    a1b2c3d4e5f6   2 weeks ago    42.5MB
# postgres     16        b2c3d4e5f6a1   3 weeks ago    379MB
# alpine       latest    c3d4e5f6a1b2   1 month ago    7.74MB

# Ver espacio ocupado por imagenes
docker system df
```

> Una imagen de contenedor es el artefacto de despliegue definitivo: empaqueta todo lo que la aplicacion necesita en un paquete inmutable y portable. La construccion una vez y ejecucion en cualquier lugar es la promesa fundamental de la contenerizacion.

