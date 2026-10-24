# Optimizacion del Dockerfile

Un Dockerfile que funciona correctamente no es necesariamente un buen Dockerfile. La eficiencia en la construccion de imagenes tiene un impacto directo en el tiempo de desarrollo, el almacenamiento en disco, el ancho de banda en las descargas y la superficie de ataque de la imagen final. Optimizar un Dockerfile es una habilidad que distingue a un usuario avanzado de Docker.

## Cacheo de capas

Docker mantiene una cache local de las capas generadas durante las construcciones. Cuando se ejecuta `docker build`, Docker compara el contenido de cada instruccion con el hash de las capas existentes. Si coincide, reutiliza la capa cacheada en lugar de ejecutar la instruccion. Este mecanismo acelera drasticamente las reconstrucciones, pero solo si las instrucciones estan ordenadas correctamente.

```dockerfile
# Eficiente: cache de dependencias hasta que cambien
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

En este orden, mientras `requirements.txt` no cambie, Docker reutiliza las capas hasta `RUN pip install`. El codigo fuente, que cambia en cada commit, se copia al final y solo reconstruye la ultima capa. El tiempo de build baja de minutos a segundos.

```dockerfile
# Ineficiente: cualquier cambio en . invalida toda la cache
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
```

Aqui, al copiar todo el directorio antes de instalar las dependencias, cualquier cambio en cualquier archivo invalida la cache de la capa `COPY` y obliga a reinstalar todas las dependencias en cada build.

> La cache se invalida por completo si cambia la instruccion `FROM`. Por eso conviene fijar una version concreta de la imagen base (`python:3.11-slim` en lugar de `python:latest`) para evitar invalidaciones imprevistas cuando se publica una nueva version de la imagen base.

## Combinar comandos RUN

Cada `RUN` genera una nueva capa. Si se ejecutan varios comandos independientes en `RUN` separados, cada uno produce una capa que persiste en la imagen final, incluso si los archivos temporales creados se eliminan despues.

```dockerfile
# Mal: cada RUN crea una capa que aumenta el tamano final
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/*
```

```dockerfile
# Bien: un solo RUN, una sola capa, limpia los temporales
RUN apt-get update && apt-get install -y curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

La regla general es agrupar operaciones relacionadas en un mismo `RUN`, especialmente cuando implican instalacion de paquetes, compilacion o descarga de archivos temporales. El operador `&&` asegura que si un comando falla, toda la instruccion falla, lo que evita construir imagenes en un estado inconsistente.

## El archivo .dockerignore

El contexto de build se envia completo al demonio de Docker. Si el directorio del proyecto contiene archivos innecesarios (modulos de node, cache de Python, archivos de entorno, directorios `.git`), estos se empaquetan y transmiten en cada build, aumentando el tiempo de envio y pudiendo sobrescribir archivos dentro del contenedor.

`.dockerignore` funciona de forma analoga a `.gitignore`: excluye archivos y directorios del contexto de build.

```
.env
.git/
node_modules/
__pycache__/
*.pyc
.env.local
dist/
*.log
```

Un `.dockerignore` bien configurado reduce el contexto de megas a kilobytes, acelera el envio al demonio y evita que archivos sensibles (como `.env` con credenciales) queden accidentalmente incluidos en la imagen.

> `.dockerignore` tambien afecta a que archivos estan disponibles para `COPY` y `ADD`. Si un archivo aparece en `.dockerignore`, no puede copiarse al contenedor. Es la primera linea de defensa contra la inclusion involuntaria de secretos.

## Multi-stage builds

Los multi-stage builds permiten usar multiples instrucciones `FROM` en un mismo Dockerfile. Cada `FROM` inicia una nueva etapa independiente, y se pueden copiar archivos entre etapas con `COPY --from`. Esto es especialmente util para compilar binarios en una etapa con herramientas de build y copiar solo el binario resultante a una etapa final ligera.

```dockerfile
# Etapa 1: compilacion
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o mi-app

# Etapa 2: imagen final minima
FROM alpine:3.18
WORKDIR /app
COPY --from=builder /app/mi-app .
EXPOSE 8080
CMD ["./mi-app"]
```

La imagen final solo contiene el binario compilado y el sistema Alpine, sin el compilador de Go ni las bibliotecas de desarrollo. El resultado es una imagen drasticamente mas pequena y con menor superficie de ataque.

```bash
# Comparar tamanos
docker images mi-app --format "{{.Size}}"
# Sin multi-stage: ~850 MB
# Con multi-stage: ~15 MB
```

## Imagenes base ligeras

La eleccion de la imagen base es el factor que mas influye en el tamano final. Una imagen `python:3.11` pesa ~330 MB, mientras que `python:3.11-slim` pesa ~120 MB y `python:3.11-alpine` ~50 MB.

| Imagen base | Tamano aprox. | Observaciones |
|-------------|--------------|---------------|
| `ubuntu:22.04` | ~77 MB | Base general, glibc |
| `debian:bookworm-slim` | ~80 MB | Compatible con la mayoria de binarios |
| `alpine:3.18` | ~5 MB | musl, requiere compilacion estatica en algunos casos |
| `gcr.io/distroless/base` | ~20 MB | Sin gestor de paquetes ni shell |

> Las imagenes distroless eliminan incluso el gestor de paquetes y la shell, lo que mejora la seguridad al reducir drasticamente la superficie de ataque, pero complica la depuracion en produccion.

## Orden de instrucciones como estrategia de CI

En pipelines de integracion continua, la cache de capas se pierde porque cada build suele ejecutarse en un entorno limpio. Para mitigar esto, se pueden usar registros intermedios que almacenen las capas o utilizar herramientas como `docker buildx` con cache remota.

```bash
# Construir con cache desde un registro
docker buildx build --cache-from=type=registry,ref=usuario/mi-app:cache \
                    --cache-to=type=registry,ref=usuario/mi-app:cache \
                    -t mi-app:latest .
```

Esto permite que los builds en CI sean casi tan rapidos como los builds locales, reutilizando las capas compartidas a traves del registro de Docker.

