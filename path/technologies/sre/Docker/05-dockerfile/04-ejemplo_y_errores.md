# Ejemplo practico y errores comunes

Un Dockerfile bien escrito es el resultado de aplicar los principios de optimizacion, seguridad y claridad en cada instruccion. Este capitulo presenta un ejemplo completo de una aplicacion Python con Flask, y analiza los errores mas frecuentes que se cometen al escribir Dockerfiles en entornos profesionales.

## Ejemplo practico: aplicacion Flask

La aplicacion es un servicio web simple que responde en el puerto 8000. El proyecto tiene la siguiente estructura:

```
mi-app/
├── app.py
├── requirements.txt
└── Dockerfile
```

Archivo `requirements.txt`:

```
flask==3.0.0
gunicorn==21.2.0
```

Archivo `app.py`:

```python
from flask import Flask
import os

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hola desde Docker"

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8000))
    app.run(host="0.0.0.0", port=port)
```

Dockerfile completo:

```dockerfile
FROM python:3.11-slim

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8000

ENV PORT=8000

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "app:app"]
```

Construccion y ejecucion:

```bash
docker build -t flask-app:1.0 .
docker run -d -p 8000:8000 flask-app:1.0
curl http://localhost:8000
```

### Decisiones de diseno en el ejemplo

- **Imagen base slim**: Python 3.11-slim proporciona glibc y pip sin el peso de las herramientas de desarrollo. Es el equilibrio optimo entre tamano y compatibilidad para aplicaciones Python.
- **Usuario no root**: Se crea un grupo y un usuario dedicados (`appuser`) y se cambia a ese usuario antes de ejecutar la aplicacion. Esto limita el impacto de una posible vulnerabilidad: un atacante que comprometa el contenedor no tendra permisos de superusuario.
- **Gunicorn como entrypoint**: Flask incluye un servidor de desarrollo, pero no es adecuado para produccion. Gunicorn es un servidor WSGI preparado para concurrentes y manejo de senales.
- **Orden de capas optimizado**: `requirements.txt` se copia antes que el resto del codigo. Esto permite cachear la instalacion de dependencias mientras no cambien.
- **`--no-cache-dir`**: Evita que pip almacene el cache de paquetes descargados, que ocuparia espacio innecesario en la capa.
- **`ENV PORT`**: La variable de entorno se define para que la aplicacion pueda leerla en tiempo de ejecucion, pero el valor en `CMD` esta hardcodeado. En un escenario real se usaria `CMD` sin hardcodear y la configuracion se delegaria a la variable de entorno.

## Errores comunes

### Hardcoding de credenciales

Incluir claves API, contrasenas o tokens directamente en el Dockerfile es el error de seguridad mas grave que se puede cometer. Cualquier persona con acceso a la imagen (incluso a traves de un registro publico o privado) puede extraer las credenciales de los metadatos de la imagen.

```dockerfile
# ERROR GRAVE: credenciales en la imagen
ENV DB_PASSWORD=supersecreto123
ENV API_KEY=sk-abc123def456
```

Las credenciales deben inyectarse en tiempo de ejecucion mediante variables de entorno externas, archivos `.env` o sistemas de gestion de secretos.

```bash
# Forma correcta: pasar credenciales al ejecutar
docker run -e DB_PASSWORD=$(cat db_password.txt) mi-app:1.0
```

Para valores necesarios durante el build (como tokens de repositorios privados), se usa `ARG`, que no persiste en la imagen final a menos que se copie explicitamente a `ENV`.

### Usar latest como imagen base

`latest` es una etiqueta movil que apunta a la version mas reciente de una imagen. Confiar en `latest` produce builds no deterministas: la misma build puede generar imagenes diferentes en momentos distintos, y una actualizacion de la imagen base puede introducir cambios incompatibles o vulnerabilidades sin previo aviso.

```dockerfile
# Mal: imagen base indeterminada
FROM python:latest

# Bien: version fija
FROM python:3.11-slim
```

La unica excepcion aceptable es en entornos de desarrollo local o prototipado rapido. En cualquier pipeline de CI/CD o produccion, la imagen base debe estar fijada a un tag concreto y actualizarse de forma controlada mediante un proceso de revision.

### No especificar CMD o ENTRYPOINT

Una imagen sin `CMD` ni `ENTRYPOINT` no puede ejecutarse con `docker run` a menos que se proporcione un comando explicito. Esto produce errores confusos y obliga a quien consuma la imagen a conocer los detalles internos del contenedor.

```bash
# Error: image has no command
docker run mi-imagen-incompleta
```

Siempre debe definirse al menos `CMD` con un comando por defecto significativo. La combinacion `ENTRYPOINT` + `CMD` es la mas flexible porque permite sobrescribir argumentos sin cambiar el ejecutable principal.

### Exponer puertos sin documentarlos

Aunque `EXPOSE` es una instruccion documental, omitirla hace que la imagen sea menos autodescriptiva. Herramientas como `docker ps` o interfaces graficas no mostraran los puertos que la aplicacion espera utilizar.

```dockerfile
# Falta EXPOSE: quien use la imagen no sabe que puerto abrir
```

### No limpiar la cache del gestor de paquetes

Las descargas de paquetes y sus metadatos ocupan espacio en la imagen final si no se eliminan en la misma instruccion `RUN`. Este es el error mas comun en imagenes basadas en Debian/Ubuntu.

```dockerfile
# Mal: las listas de paquetes ocupan ~30 MB en la imagen
RUN apt-get update && apt-get install -y curl

# Bien: se limpian en el mismo RUN
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

Ejemplos equivalentes para otros gestores:

```dockerfile
# apk (Alpine)
RUN apk add --no-cache curl

# pip (Python)
RUN pip install --no-cache-dir -r requirements.txt

# npm (Node.js)
RUN npm install --production && npm cache clean --force
```

### Multiples capas innecesarias

Cada instruccion `RUN`, `COPY` y `ADD` crea una capa. Aunque las capas se comparten entre imagenes, un numero excesivo de capas pequenas puede degradar el rendimiento de las operaciones de UnionFS y aumentar el tiempo de construccion.

```dockerfile
# Evitar: demasiadas capas sin motivo
RUN mkdir -p /app
RUN mkdir -p /app/data
RUN mkdir -p /app/logs
RUN chown -R appuser:appuser /app
```

```dockerfile
# Mejor: una sola capa para operaciones relacionadas
RUN mkdir -p /app /app/data /app/logs && \
    chown -R appuser:appuser /app
```

### No gestionar los permisos correctamente

Ejecutar como root dentro del contenedor o copiar archivos con permisos incorrectos son problemas recurrentes. Un contenedor que corre como root otorga al proceso dentro del contenedor los mismos privilegios que el usuario root del host si se produce un escape de contenedor.

```dockerfile
# Mal: root por defecto
FROM python:3.11-slim
COPY . /app
CMD ["python", "app.py"]
```

```dockerfile
# Bien: usuario no root
FROM python:3.11-slim
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["python", "app.py"]
```

El cambio de usuario debe ocurrir lo mas tarde posible, justo antes del comando de ejecucion, para que las instrucciones anteriores (como `RUN pip install`) puedan ejecutarse con los permisos necesarios.

