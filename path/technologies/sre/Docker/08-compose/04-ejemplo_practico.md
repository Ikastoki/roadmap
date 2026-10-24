# Ejemplo practico: Flask + PostgreSQL + Nginx

Un ejemplo real permite ver como todos los conceptos de Compose se integran en una aplicacion funcional. El escenario es una aplicacion web escrita en Flask que expone una API REST, PostgreSQL como base de datos relacional y Nginx como proxy inverso que sirve contenido estatico y redirige las peticiones de API al backend. Cada componente se ejecuta en su propio contenedor, orquestado por un unico archivo `docker-compose.yml`.

## Estructura del proyecto

```
mi-app/
├── docker-compose.yml
├── .env
├── nginx/
│   └── default.conf
├── web/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
└── db/
    └── init.sql
```

## Archivo .env

Separa la configuracion sensible y los parametros del entorno:

```
POSTGRES_DB=mi_app
POSTGRES_USER=admin
POSTGRES_PASSWORD=secreto_cambiame
FLASK_ENV=production
SECRET_KEY=clave_secreta_segura
```

## Archivo docker-compose.yml

```yaml
services:
  db:
    image: postgres:15-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    env_file:
      - .env
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 15s
    networks:
      - backend

  web:
    build: ./web
    ports:
      - "5000:5000"
    env_file:
      - .env
    environment:
      - DB_HOST=db
      - DB_PORT=5432
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./web:/app
    networks:
      - backend

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - web
    networks:
      - backend

volumes:
  pgdata:

networks:
  backend:
    driver: bridge
```

## Dockerfile de la aplicacion Flask

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

```txt
# requirements.txt
Flask==3.0.0
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

## Aplicacion Flask

```python
import os
import psycopg2
from flask import Flask, jsonify

app = Flask(__name__)

def get_db_connection():
    conn = psycopg2.connect(
        host=os.environ["DB_HOST"],
        port=os.environ["DB_PORT"],
        dbname=os.environ["POSTGRES_DB"],
        user=os.environ["POSTGRES_USER"],
        password=os.environ["POSTGRES_PASSWORD"]
    )
    return conn

@app.route("/health")
def health():
    return jsonify({"status": "ok"})

@app.route("/items")
def get_items():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("SELECT id, nombre FROM items LIMIT 10")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify([{"id": r[0], "nombre": r[1]} for r in rows])

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

## Script de inicializacion de la base de datos

```sql
-- db/init.sql
CREATE TABLE IF NOT EXISTS items (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL
);

INSERT INTO items (nombre) VALUES
    ('Elemento 1'),
    ('Elemento 2'),
    ('Elemento 3');
```

PostgreSQL ejecuta automaticamente cualquier archivo `.sql` o `.sh` ubicado en `/docker-entrypoint-initdb.d/` la primera vez que se inicia el contenedor con un volumen vacio.

## Configuracion de Nginx

```nginx
# nginx/default.conf
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://web:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /static/ {
        alias /usr/share/nginx/html/static/;
    }
}
```

Nginx actua como unico punto de entrada: recibe peticiones en el puerto 80 y las redirige al servicio `web` utilizando el nombre del servicio como direccion DNS, que Compose resuelve automaticamente dentro de la red `backend`.

## Flujo de trabajo

```bash
# Clonar o situarse en el directorio del proyecto
cd mi-app/

# Construir y levantar todos los servicios
docker compose up -d

# Verificar el estado
docker compose ps

# Consultar la API a traves de Nginx
curl http://localhost/health
curl http://localhost/items

# Ver logs de todos los servicios
docker compose logs -f

# Ejecutar un comando en el contenedor web
docker compose exec web python -c "import flask; print(flask.__version__)"

# Detener y limpiar (conserva volumen de BD)
docker compose down

# Detener y eliminar tambien el volumen de datos
docker compose down -v
```

## Consideraciones importantes

La seccion `depends_on` con `condition: service_healthy` en el servicio `web` garantiza que la base de datos este aceptando conexiones antes de que Flask intente conectarse. Sin esta condicion, la aplicacion podria iniciar antes que PostgreSQL y lanzar errores de conexion.

El volumen `pgdata` es un volumen nombrado que persiste los datos de la base de datos entre ejecuciones. Si se ejecuta `docker compose down` sin `-v`, la base de datos conserva sus datos. Esto es fundamental en cualquier entorno que no sea de desarrollo.

El bind mount `./web:/app` en el servicio `web` monta el codigo local dentro del contenedor, lo que permite que los cambios en el codigo se reflejen sin reconstruir la imagen. Esta practica es habitual en desarrollo pero no debe usarse en produccion; alli la imagen debe contener el codigo completo.

> En produccion, eliminar los bind mounts y usar exclusivamente imagenes construidas con `docker compose build`. El bind mount de desarrollo permite hot-reload pero expone el sistema de archivos del host al contenedor, lo que reduce el aislamiento.

