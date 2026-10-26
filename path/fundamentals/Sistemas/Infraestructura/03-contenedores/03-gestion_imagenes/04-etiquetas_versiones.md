# Etiquetas (tags) y versiones

Las etiquetas son la forma legible de referenciar imagenes de contenedores. Cada imagen tiene un ID unico (digest criptografico) que es ilegible para humanos; las etiquetas permiten asignar nombres descriptivos y versiones a esos IDs. Gestionar correctamente las etiquetas es esencial para mantener un historial claro de versiones y evitar confusiones en produccion.

## Estructura de una etiqueta

```
[registro/]organizacion/imagen[:tag][@digest]

Ejemplos:
nginx                              → docker.io/library/nginx:latest
nginx:1.25-alpine                  → docker.io/library/nginx:1.25-alpine
mi-empresa/mi-app:1.2.3            → docker.io/mi-empresa/mi-app:1.2.3
ghcr.io/usuario/proyecto:1.0       → GitHub Container Registry
registry.empresa.com/app:v2.1.0    → Registro privado
```

## Etiquetas recomendadas

**Version explicita (recomendado):**

```bash
docker pull postgres:16-alpine
docker pull node:20.11-slim
docker pull python:3.12.2-slim-bookworm
```

**latest (evitar en produccion):**

```bash
docker pull nginx:latest
# "latest" es una etiqueta que apunta a la ultima version publicada
# Puede cambiar sin aviso: hoy apunta a 1.25.4, manana a 1.26.0
# Esto puede romper aplicaciones en produccion
```

## Semantic Versioning (SemVer)

El estandar para versionar software: `MAJOR.MINOR.PATCH`

- **MAJOR:** cambios incompatibilidades con versiones anteriores.
- **MINOR:** nuevas funcionalidades compatibles con versiones anteriores.
- **PATCH:** correcciones de bugs compatibles.

```bash
# Ejemplo de etiquetas SemVer
mi-app:1.0.0    # Version inicial
mi-app:1.0.1    # Correccion de bug
mi-app:1.1.0    # Nueva funcionalidad
mi-app:2.0.0    # Cambio incompatible
```

## Tags inmutables vs. mutables

**Tags mutables (pueden cambiar):**
- `latest` siempre apunta a la ultima version.
- `stable` puede cambiar con cada release.

**Tags inmutables (no cambian):**
- `1.2.3` apunta a una version especifica.
- Los digest SHA256 son inmutables por definicion.

```bash
# Usar digest para maxima inmutabilidad
docker pull nginx@sha256:a1b2c3d4e5f6...
# Este digest nunca cambiara
```

## Etiquetar para diferentes propositos

```bash
# Tags de version
docker tag mi-app:latest mi-app:1.2.3
docker tag mi-app:latest mi-app:1.2
docker tag mi-app:latest mi-app:1

# Tags de entorno
docker tag mi-app:1.2.3 mi-app:staging
docker tag mi-app:1.2.3 mi-app:production

# Tags de plataforma
docker tag mi-app:1.2.3 mi-app:1.2.3-amd64
docker tag mi-app:1.2.3 mi-app:1.2.3-arm64
```

## Eliminar etiquetas

```bash
# Eliminar una etiqueta (no elimina la imagen si tiene otros tags)
docker rmi mi-app:1.0

# Verificar que la imagen sigue existiendo
docker images mi-app
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# mi-app       1.1       a1b2c3d4e5f6   1 hour ago     150MB
# mi-app       latest    a1b2c3d4e5f6   1 hour ago     150MB
```

> Las etiquetas son el contrato entre quien construye la imagen y quien la despliega. Un tag claro, consistente y versionado comunica exactamente que version de la aplicacion se esta ejecutando. En produccion, nunca uses `latest`; siempre usa una version explicita que puedas rastrear y reproducir.

