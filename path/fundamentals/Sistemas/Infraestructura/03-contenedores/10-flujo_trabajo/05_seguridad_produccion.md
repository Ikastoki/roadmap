# Seguridad en produccion

La seguridad de contenedores en produccion abarca la imagen base, la ejecucion del contenedor, la gestion de secretos y los permisos del sistema operativo. Cada capa de la pila de contenedores tiene vulnerabilidades potenciales que deben ser mitigadas con practicas especificas.

## Imagenes seguras

```dockerfile
# Usar imagenes minimas
FROM node:20-alpine  # En lugar de node:20 (45MB vs 1GB)

# O mejor: Distroless (sin shell, sin gestor de paquetes)
FROM gcr.io/distroless/nodejs20-debian12

# O incluso: scratch (vacia)
FROM scratch
COPY mi-app /mi-app
CMD ["/mi-app"]
```

```bash
# Escanear imagenes antes de desplegar
trivy image --severity HIGH,CRITICAL miusuario/mi-app:v1.2.3
```

## Ejecucion sin privilegios

```dockerfile
# Crear usuario no root
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Cambiar propiedad de directorios
RUN chown -R appuser:appgroup /app

# Ejecutar como usuario no root
USER appuser
```

```yaml
# En Docker Compose
services:
  app:
    image: miusuario/mi-app:v1.2.3
    user: "1001:1001"
    read_only: true  # Filesystem de solo lectura
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

## Gestion de secretos

```yaml
# Docker Compose secrets
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

```yaml
# Kubernetes secrets
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: c3VwZXJzZWNyZXQ=

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
        - name: app
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
```

## Permisos del SO

```yaml
services:
  app:
    image: mi-app
    security_opt:
      - no-new-privileges:true  # No escalada de privilegios
      - apparmor:docker-default  # Perfil AppArmor
    cap_drop:
      - ALL                    # Elimina todas las capacidades
    cap_add:
      - NET_BIND_SERVICE       # Solo la necesaria
    read_only: true            # Filesystem solo lectura
    tmpfs:
      - /tmp:size=100m         # /tmp en memoria
```

## Red segura

```yaml
services:
  db:
    image: postgres:16-alpine
    networks:
      - backend
    # No publica puertos al exterior

  app:
    image: mi-app
    networks:
      - frontend
      - backend
    ports:
      - "443:3000"

networks:
  frontend:
  backend:
    internal: true  # Sin acceso a Internet
```

> La seguridad en produccion no es una caracteristica que se agrega despues: es una propiedad que se construye desde la imagen base hasta la ejecucion del contenedor. Cada capa de la pila debe ser escaneada, aislada y minimizada en privilegios para crear un sistema que sea resilientente ante ataques.

