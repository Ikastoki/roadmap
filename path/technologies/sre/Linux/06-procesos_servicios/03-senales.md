# Envio de senales

Las senales son notificaciones asincronas que el kernel o un proceso envian a otro proceso para indicarle que debe realizar una accion. Son el mecanismo fundamental de comunicacion entre procesos en Unix y la unica forma de controlar un proceso desde fuera de su propio codigo. Una senal puede interrumpir la ejecucion normal de un programa, forzar su terminacion, suspenderlo o activar un manejador personalizado definido por el programador.

## Senales comunes

Cada senal tiene un nombre simbolico, un numero y un comportamiento predeterminado. El kernel define unas 30 senales estandar; las mas relevantes para la administracion del sistema son tres:

**SIGTERM (15).** Solicitud educada de terminacion. El proceso recibe la senal y puede ejecutar su rutina de limpieza antes de salir: cerrar archivos, liberar memoria, enviar mensajes a otros procesos. Es la forma correcta de detener un proceso en la mayoria de los casos.

**SIGKILL (9).** Terminacion forzosa e incondicional. El kernel elimina el proceso inmediatamente sin notificarlo. El proceso no puede capturar, ignorar ni bloquear esta senal. Se usa solo cuando SIGTERM no funciona, ya que puede dejar archivos temporales, sockets abiertos o estados inconsistentes.

**SIGHUP (1).** Historicamente enviada al colgarse una conexion de modem (HangUP). Hoy en dia, muchos servicios (demonios) la interpretan como orden de recargar su configuracion sin detenerse. Apache, nginx, sshd y systemd recargan su configuracion al recibir SIGHUP.

```bash
# Ver lista completa de senales
kill -l

# Numeros y nombres
kill -l | tr ' ' '\n' | nl
```

Otras senales de uso frecuente:
| Senal | Numero | Comportamiento |
|-------|--------|----------------|
| SIGINT | 2 | Interrupcion desde teclado (Ctrl+C) |
| SIGQUIT | 3 | Salida con volcado de memoria (Ctrl+\) |
| SIGSTOP | 19 | Suspender ejecucion (no capturable) |
| SIGCONT | 18 | Reanudar ejecucion suspendida |
| SIGUSR1 | 10 | Definida por el usuario |
| SIGUSR2 | 12 | Definida por el usuario |

> SIGTERM es siempre la primera opcion. SIGKILL es el ultimo recurso. Entre medias, SIGHUP permite recargar servicios sin interrumpir el servicio. Si un proceso ignora SIGTERM, prueba con SIGQUIT (3) antes de recurrir a SIGKILL.

## kill: enviar senales por PID

El comando `kill` envia senales a procesos identificados por su PID. Su nombre es historico (el comportamiento predeterminado es SIGTERM, no necesariamente "matar"). Sin especificar senal, `kill` envia SIGTERM.

```bash
# Enviar SIGTERM a un proceso
kill 1234

# Enviar una senal especifica
kill -15 1234    # SIGTERM (por defecto)
kill -9 1234     # SIGKILL
kill -1 1234     # SIGHUP
kill -SIGTERM 1234
kill -s SIGTERM 1234

# Enviar senal a varios procesos
kill 1234 5678 9012
```

`kill` devuelve 0 si la senal se envio correctamente y 1 si fallo (por ejemplo, porque el PID no existe o el proceso pertenece a otro usuario). No indica si el proceso termino efectivamente; solo confirma que la senal se entrego al kernel.

```bash
# Verificar si un proceso responde
kill -0 1234  # No envia senal, solo comprueba existencia
echo $?       # 0 si existe, 1 si no
```

> El comando `kill` no recibe su nombre por su efecto, sino por la llamada al sistema que lo implementa. `kill(2)` puede enviar cualquier senal, incluyendo aquellas que no matan el proceso.

## killall y pkill: matar por nombre

`killall` termina procesos por su nombre de comando. A diferencia de `kill`, no requiere conocer el PID. Es util en scripts donde el nombre del proceso es conocido pero su PID puede variar.

```bash
# Detener todos los procesos llamados "nginx"
sudo killall nginx

# Enviar senal especifica
sudo killall -9 chrome
sudo killall -1 rsyslog

# Ignorar mayusculas/minusculas
killall -I firefox

# Preguntar antes de matar
killall -i nombre
```

`killall` por defecto actua sobre todo el sistema. Para restringirlo al usuario actual: `killall -u usuario nombre`.

`pkill` ofrece filtros mas flexibles que `killall`. Permite buscar por patrones, por usuario, por terminal y por otros criterios.

```bash
# Matar procesos cuyo nombre contenga "java"
pkill java

# Matar procesos de un usuario especifico
pkill -u nombre_usuario ssh

# Coincidencia exacta (no patron)
pkill -x nginx

# Enviar senal especifica con pkill
pkill -9 -u www-data php-fpm
```

> La diferencia practica: `killall` coincide con el nombre exacto del ejecutable, mientras que `pkill` usa expresiones regulares. `pkill java` matara procesos llamados "java", "javac" y "javascript". Para evitar falsos positivos, usa `-x` (coincidencia exacta) o `killall`.

## Trabajos en segundo plano

La shell permite ejecutar comandos en segundo plano, liberando la terminal para otras tareas. Este mecanismo no es gestionado por el kernel directamente, sino por el control de trabajos del shell.

**Ejecucion en segundo plano con `&`.** Anadir `&` al final de un comando lo lanza inmediatamente en segundo plano. La shell muestra el numero de trabajo (job) entre corchetes y el PID, y devuelve el prompt inmediatamente.

```bash
# Ejecutar en segundo plano
sleep 100 &

# Lanzar proceso y redirigir su salida
python server.py > log.txt 2>&1 &
```

**Suspender un proceso con Ctrl+Z.** Detiene la ejecucion del proceso en primer plano y lo envia a segundo plano en estado Stopped (T). El proceso queda suspendido, no ejecutandose.

**Gestionar trabajos con `jobs`, `bg`, `fg`.** El comando `jobs` lista los trabajos asociados a la terminal actual con su numero de trabajo y estado.

```bash
# Listar trabajos
jobs

# Reanudar un trabajo suspendido en segundo plano
bg %1
bg          # Reanuda el ultimo trabajo suspendido

# Traer un trabajo a primer plano
fg %1
fg          # Trae el ultimo trabajo

# Matar un trabajo por su numero de trabajo
kill %1
```

Los trabajos se identifican con `%n` donde n es el numero de trabajo. El caracter `%` solo se reconoce en la shell actual, no desde otros procesos. Los trabajos en segundo plano continuan ejecutandose incluso si la terminal se cierra si se lanzan con `nohup` o `disown`.

```bash
# Ejecutar proceso inmune a SIGHUP (cierre de terminal)
nohup long_running_task &

# Desvincular un trabajo ya lanzado del shell
long_running_task &
disown %1
```

> Un trabajo en segundo plano no esta a salvo de SIGHUP si la terminal se cierra. Usa `nohup` al lanzarlo o `disown` despues para protegerlo. `tmux` o `screen` son alternativas mas robustas para sesiones de larga duracion.

