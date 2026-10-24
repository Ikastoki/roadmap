# Recursos y limites

Un contenedor sin restricciones puede consumir toda la memoria disponible del sistema, saturar la CPU o llenar el disco, afectando a otros contenedores y al propio kernel del anfitrion. Docker permite establecer limites precisos sobre los recursos que un contenedor puede utilizar, apoyandose en los cgroups (control groups) del kernel de Linux. Estos limites se definen en el momento de crear el contenedor y el kernel los hace cumplir en tiempo de ejecucion.

## Limite de memoria: --memory

La opcion `--memory` (o `-m`) limita la cantidad maxima de memoria RAM que un contenedor puede consumir. Si el contenedor supera este limite, el kernel comienza a matar procesos dentro del contenedor (OOM kill) para liberar memoria.

```bash
# Limitar a 256 megabytes
docker run -d --name mi-app --memory="256m" nginx

# Limitar a 2 gigabytes
docker run -d --name mi-app --memory="2g" nginx
```

El valor puede expresarse en bytes (b), kilobytes (k), megabytes (m) o gigabytes (g). Cuando se establece solo `--memory` sin `--memory-swap`, el limite de swap es igual al de memoria, por lo que el contenedor no puede usar mas swap que memoria fisica.

```bash
# Memoria 256m, swap 512m (256m RAM + 256m swap)
docker run -d --memory="256m" --memory-swap="512m" nginx

# Sin swap (deshabilitar swap para el contenedor)
docker run -d --memory="256m" --memory-swap="256m" nginx
```

La reserva de memoria (`--memory-reservation`) establece un valor por debajo del cual Docker intenta mantener el consumo. Es un objetivo blando, no un limite estricto: si hay presion de memoria en el host, el kernel intenta reducir el consumo del contenedor hasta este valor.

```bash
# Reserva de 128m, limite duro de 512m
docker run -d --memory="512m" --memory-reservation="128m" nginx
```

## Limite de CPU: --cpus

La opcion `--cpus` limita la cantidad de nucleos de CPU que el contenedor puede utilizar. Acepta valores decimales: `--cpus="1.5"` permite al contenedor usar el equivalente a un nucleo y medio.

```bash
# Usar como maximo 1 nucleo completo
docker run -d --name mi-app --cpus="1" nginx

# Usar como maximo 2.5 nucleos
docker run -d --name mi-app --cpus="2.5" nginx
```

Docker implementa este limite mediante el periodo de CPU de los cgroups. Por defecto, el periodo es de 100000 microsegundos. Asignar `--cpus="1.5"` equivale a darle al contenedor 150000 microsegundos de tiempo de CPU por cada periodo de 100000 microsegundos, lo que se traduce en un nucleo y medio de forma consistente.

Existen opciones mas granulares para controlar la CPU:

```bash
# Prioridad relativa (peso): 1024 por defecto, mas alto = mas prioridad
docker run -d --name app-prioritaria --cpu-shares=2048 nginx

# Nucleos especificos (pinning)
docker run -d --name app-aislada --cpuset-cpus="0,2" nginx
```

`--cpu-shares` asigna un peso relativo entre contenedores. Si hay dos contenedores con 1024 y 2048, el segundo recibe el doble de tiempo de CPU cuando ambos compiten por recursos. `--cpuset-cpus` fija el contenedor a nucleos fisicos especificos, util para cargas de trabajo que requieren cache de CPU predecible.

## Limites combinados

En produccion es practica habitual combinar limites de memoria y CPU para evitar que un contenedor degrade el rendimiento del sistema:

```bash
docker run -d \
  --name api-produccion \
  --memory="512m" \
  --memory-reservation="256m" \
  --cpus="1.5" \
  --cpuset-cpus="0,1" \
  mi-api:1.0
```

## Limite de reinicios: --restart

La opcion `--restart` define la politica de reinicio automatico cuando el contenedor termina. Es esencial para servicios que deben estar siempre disponibles.

```bash
# Reintentar siempre (hasta que se detenga explicitamente con docker stop)
docker run -d --name mi-app --restart=always nginx

# Reintentar solo si falla (codigo de salida distinto de cero)
docker run -d --name mi-app --restart=on-failure nginx

# Reintentar hasta 3 veces solo si falla
docker run -d --name mi-app --restart=on-failure:3 nginx

# No reiniciar (por defecto)
docker run -d --name mi-app --restart=no nginx
```

## Eliminacion automatica: --rm

La opcion `--rm` elimina automaticamente el contenedor cuando este se detiene. Es especialmente util para contenedores temporales, pruebas, comandos batch o sesiones interactivas que no deben dejar residuos.

```bash
# Ejecutar una tarea y limpiar automaticamente
docker run --rm alpine echo "Tarea completada"
# El contenedor se elimina inmediatamente despues de ejecutar el echo

# Shell interactivo que se autodestruye al salir
docker run -it --rm alpine /bin/sh

# Pruebas rapidas sin acumular contenedores detenidos
docker run --rm -p 8080:80 nginx
# Al hacer Ctrl+C, el contenedor se detiene y se elimina
```

Sin `--rm`, cada `docker run` genera una entrada en `docker ps -a` que ocupa espacio en disco y contamina la lista de contenedores. Los contenedores con `--rm` no se pueden reiniciar (no existe un contenedor que reiniciar) y sus logs se pierden al eliminarse.

```bash
# Sin --rm: el contenedor persiste detenido
docker run alpine echo "Hola"
docker ps -a  # El contenedor aparece como exited

# Con --rm: el contenedor desaparece al terminar
docker run --rm alpine echo "Hola"
docker ps -a  # No hay rastro del contenedor
```

## Verificar limites en ejecucion

Una vez que el contenedor esta en marcha, se pueden verificar los limites aplicados:

```bash
# Ver limites de recursos
docker inspect mi-app --format '{{.HostConfig.Memory}}'
docker inspect mi-app --format '{{.HostConfig.NanoCpus}}'
docker inspect mi-app --format '{{.HostConfig.RestartPolicy.Name}}'

# Monitorizar uso en tiempo real
docker stats mi-app
```

> Establecer limites no solo protege al anfitrion de contenedores mal comportados, sino que tambien ayuda a dimensionar correctamente los recursos necesarios para cada servicio. En entornos de orquestacion como Kubernetes, estos limites son obligatorios y se definen en las especificaciones del pod.

