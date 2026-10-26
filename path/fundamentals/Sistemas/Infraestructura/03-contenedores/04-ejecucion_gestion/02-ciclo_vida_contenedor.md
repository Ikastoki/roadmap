# Ciclo de vida de un contenedor

Un contenedor sigue un ciclo de vida bien definido: desde su creacion hasta su eliminacion, pasando por diferentes estados que determinan que puede hacer y que recursos consume. Comprender este ciclo es fundamental para gestionar contenedores de forma eficiente.

## Estados de un contenedor

```
Imagen ──run──► Created ──start──► Running ──pause──► Paused
                              │         ▲
                              │    unpause
                              │         │
                         stop │         │ start
                              ▼         │
                           Stopped ─────┘
                              │
                              │ rm
                              ▼
                           Deleted
```

- **Created:** el contenedor fue creado pero no esta en ejecucion. Ocupa espacio en disco pero no consume CPU ni memoria.
- **Running:** el contenedor esta en ejecucion. El proceso principal (PID 1) esta activo. Consume CPU, memoria y otros recursos.
- **Paused:** el proceso del contenedor esta congelado (señal SIGSTOP). No consume CPU pero mantiene el estado en memoria.
- **Stopped:** el proceso principal termino (señal SIGTERM o SIGKILL). El contenedor sigue existiendo pero no consume recursos.
- **Deleted:** el contenedor fue eliminado permanentemente.

## Regla fundamental

**Un contenedor vive mientras su proceso principal (PID 1) este activo.** Si PID 1 termina, el contenedor se detiene, incluso si hay otros procesos corriendo dentro.

```bash
# Ejemplo: contenedor que se detiene cuando el proceso termina
docker run --rm alpine sh -c "echo Hola; exit 0"
# El contenedor imprime "Hola" y se detiene inmediatamente

# Ejemplo: contenedor que sigue ejecutandose
docker run -d --name web nginx:alpine
# nginx es PID 1 y se ejecuta indefinidamente
# El contenedor sigue vivo mientras nginx este corriendo
```

## Gestion del ciclo de vida

```bash
# Crear sin iniciar
docker create --name mi-web nginx:alpine
# Estado: Created

# Iniciar
docker start mi-web
# Estado: Running

# Pausar
docker pause mi-web
# Estado: Paused

# Reanudar
docker unpause mi-web
# Estado: Running

# Detener (envia SIGTERM, luego SIGKILL despues de timeout)
docker stop mi-web
# Estado: Stopped

# Iniciar de nuevo
docker start mi-web
# Estado: Running

# Reiniciar
docker restart mi-web
# Detiene y luego inicia

# Eliminar (requiere estar detenido)
docker rm mi-web
# Elimina el contenedor permanentemente

# Forzar eliminacion (mata el contenedor primero)
docker rm -f mi-web
```

## Tiempo de espera en stop

```bash
# stop espera 10 segundos por defecto antes de enviar SIGKILL
docker stop mi-web
# SIGTERM → espera 10s → SIGKILL

# Cambiar el timeout
docker stop -t 30 mi-web    # Esperar 30 segundos
docker stop -t 0 mi-web     # Enviar SIGKILL inmediatamente

# La aplicacion debe manejar SIGTERM para un apagado limpio
```

## Eliminacion automatica

```bash
# --rm elimina el contenedor automaticamente al detenerse
docker run --rm alpine sh -c "echo Hola; exit 0"
# El contenedor se crea, ejecuta y elimina sin dejar rastro

# Ideal para tareas efimeras, pruebas y scripts
docker run --rm -v $(pwd):/app -w /app python:3.12 python script.py
# Ejecuta Python, procesa el script y se elimina
```

## Inspeccionar estado

```bash
# Ver estado de un contenedor
docker inspect --format='{{.State.Status}}' mi-web
# running

# Ver si esta saludable
docker inspect --format='{{.State.Health.Status}}' mi-web
# healthy

# Ver tiempo de ejecucion
docker inspect --format='{{.State.StartedAt}}' mi-web
# 2024-01-15T10:30:00Z
```

> El ciclo de vida de un contenedor es simple pero riguroso: PID 1 activo = contenedor vivo; PID 1 terminado = contenedor detenido. Esta regla fundamental determina como se diseñan las aplicaciones para contenedores: el proceso principal debe manejar señales de terminacion graceful y no depender de procesos secundarios.

