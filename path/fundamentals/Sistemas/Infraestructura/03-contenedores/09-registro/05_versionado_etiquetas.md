# Politicas de etiquetado y versionado

El etiquetado de imagenes es la forma de identificar y versionar cada construccion. Un sistema de etiquetado coherente permite rastrear que version del codigo esta ejecutandose, facilitar rollbacks y comunicar estabilidad. El Semantic Versioning (SemVer) es el estandar para versionado de software.

## Semantic Versioning

```bash
# Formato: MAJOR.MINOR.PATCH
# MAJOR: cambios incompatibles con versiones anteriores
# MINOR: nuevas funcionalidades compatibles
# PATCH: correccion de errores

# Ejemplos
v1.0.0    # Version inicial
v1.0.1    # Correccion de error
v1.1.0    # Nueva funcionalidad
v2.0.0    # Cambio incompatible
```

## Etiquetado en Docker

```bash
# Con SHA del commit (uniqueness)
docker tag mi-app:latest miusuario/mi-app:sha-abc1234

# Con version SemVer
docker tag mi-app:latest miusuario/mi-app:v1.2.3

# Con fecha (para nightly builds)
docker tag mi-app:latest miusuario/mi-app:2024.01.15

# latest solo para desarrollo
docker tag mi-app:latest miusuario/mi-app:latest  # NO usar en produccion
```

## Estrategia de etiquetas

```bash
# Tags recomendados
miusuario/mi-app:v1.2.3      # Version estable
miusuario/mi-app:v1.2         # Ultima v1.2.x
miusuario/mi-app:v1           # Ultima v1.x.x
miusuario/mi-app:sha-abc1234  # Commit especifico
miusuario/mi-app:latest       # Desarrollo (NO produccion)

# Tags NO recomendados
miusuario/mi-app:latest       # En produccion: incierto
miusuario/mi-app:dev          # Ambiguo
miusuario/mi-app:latest-v2    # Confuso
```

## Politicas por entorno

```bash
# Desarrollo: latest o SHA
miusuario/mi-app:latest
miusuario/mi-app:sha-abc1234

# Staging: RC (Release Candidate)
miusuario/mi-app:v1.2.3-rc.1

# Produccion: version exacta
miusuario/mi-app:v1.2.3
```

## Automatizacion con CI/CD

```bash
# GitHub Actions: etiquetar automaticamente
# En el push a main: v1.2.3-sha-abc1234
# En tag v1.2.3: v1.2.3

# GitLab CI
docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG .
docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
```

> Un sistema de etiquetado coherente es la base de la trazabilidad en la distribucion de imagenes. Cada imagen debe tener una etiqueta que permita rastrear exactamente que codigo esta ejecutandose, y el Semantic Versioning proporciona el marco para comunicar estabilidad y compatibilidad.

