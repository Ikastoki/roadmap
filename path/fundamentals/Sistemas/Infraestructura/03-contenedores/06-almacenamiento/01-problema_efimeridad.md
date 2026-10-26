# El problema de la efimeridad

El sistema de archivos de un contenedor es efímero: todos los datos escritos dentro del contenedor se pierden irreversiblemente cuando el contenedor se elimina. Esto es una caracteristica, no un bug: las imagenes son inmutables y los contenedores son descartables. Pero crea un problema fundamental para datos que deben persistir: bases de datos, archivos de usuarios, logs historicos y cualquier dato que deba sobrevivir al ciclo de vida del contenedor.

## Por que se pierden los datos

```bash
# Crear contenedor con datos
docker run --name mi-db postgres:16-alpine
# Dentro del contenedor, crear tablas, insertar datos

# Detener y eliminar el contenedor
docker stop mi-db
docker rm mi-db
# Todos los datos se han perdido permanentemente

# Crear nuevo contenedor
docker run --name mi-db-nueva postgres:16-alpine
# Base de datos vacia, sin datos anteriores
```

## La solucion: almacenamiento externo

Los datos deben almacenarse fuera del ciclo de vida del contenedor, en volumenes gestionados por el motor de contenedores o en bind mounts del host.

```
Sin volumen:
  Contenedor ──► filesystem efímero ──► datos perdidos al eliminar

Con volumen:
  Contenedor ──► volumen persistente ──► datos sobreviven al eliminar
```

## Datos que requieren persistencia

- **Bases de datos:** tablas, indices, transacciones.
- **Archivos de usuarios:** uploads, documentos, imagenes.
- **Logs historicos:** registros de actividad pasados.
- **Configuracion:** archivos de configuracion generados dinamicamente.
- **Cache persistente:** datos de cache que deben sobrevivir reinicios.
- **Secretos:** archivos de certificados, claves.

## Datos que NO requieren persistencia

- **Cache temporal:** datos regenerables.
- **Archivos temporales:** logs de sesion, archivos de proceso.
- **Dependencias:** paquetes instalados (se reinstalan desde la imagen).
- **Codigo fuente:** esta en la imagen o en un volumen montado.

> La efimeridad de los contenedores es su mayor fortaleza para el despliegue (siempre se puede recrear un contenedor desde la imagen) y su mayor debilidad para los datos (necesitan almacenamiento externo). Separar claramente el codigo (en la imagen) de los datos (en volumenes) es el principio fundamental del almacenamiento en contenedores.

