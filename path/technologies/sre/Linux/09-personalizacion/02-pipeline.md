# Pipeline (tuberias)

Un pipeline es una secuencia de comandos conectados por el operador `|`, donde la salida estandar (STDOUT) de cada programa se convierte en la entrada estandar (STDIN) del siguiente. La shell ejecuta todos los procesos del pipeline simultaneamente, conectando los descriptores de archivo de modo que lo que escribe un proceso lo lee el siguiente sin pasar por archivos temporales ni por el terminal.

Este mecanismo encarna el principio Unix de composicion: programas pequenos que hacen una cosa bien y se conectan entre si para realizar tareas complejas. Un pipeline no requiere que los programas conozcan la existencia de otros; simplemente leen de STDIN y escriben en STDOUT, y la shell orquesta la conexion.

## Funcionamiento interno

Cuando la shell encuentra el operador `|`, crea un pipe (tuberia) del kernel: un buffer en memoria con capacidad tipica de 65536 bytes (64 KB) en Linux. La shell bifurca los procesos a izquierda y derecha del pipe, conectando STDOUT del proceso izquierdo al extremo de escritura del pipe y STDIN del proceso derecho al extremo de lectura.

```bash
# Sintaxis basica: comando1 | comando2
ls -la | wc -l
# cuenta cuantos archivos hay en el directorio actual
```

El buffer del pipe evita que procesos lentos bloqueen a procesos rapidos: el escritor puede adelantarse al lector hasta llenar el buffer, momento en que el kernel bloquea la escritura hasta que el lector consuma datos. Analogamente, si el lector va mas rapido que el escritor, el kernel bloquea la lectura hasta que haya datos disponibles.

```bash
# Visualizar el comportamiento del pipe con un ejemplo real
# dmesg produce mucha salida; less la pagina
dmesg | less
# dmesg escribe en el pipe; less lee del pipe y muestra una pagina cada vez
```

> Los pipes son unidireccionales. Para comunicacion bidireccional existen los named pipes (FIFO) creados con `mkfifo`, pero el pipe anonimo creado por `|` solo fluye en una direccion.

## Encadenamiento multiple

Un pipeline puede tener tantos elementos como se necesiten. Cada `|` conecta la salida de un comando con la entrada del siguiente:

```bash
# Pipeline de tres etapas
cat /var/log/syslog | grep "error" | wc -l
# Cuenta cuantas lineas contienen "error" en el log del sistema

# Pipeline de cuatro etapas
ps aux | grep apache | awk '{print $2}' | sort -n
# Lista los PID de procesos apache ordenados numericamente
```

Cada etapa se ejecuta en un proceso hijo independiente. La shell espera a que termine el ultimo proceso del pipeline, pero todos los procesos intermedios se ejecutan concurrentemente. Esto implica que un proceso intermedio puede morir (SIGPIPE) si el siguiente proceso deja de leer, por ejemplo al usar `head`:

```bash
# El proceso de la izquierda recibira SIGPIPE cuando head termine
journalctl -f | head -5
# journalctl seguira escribiendo hasta llenar el buffer del pipe,
# momento en que el kernel enviara SIGPIPE porque head ya cerro su extremo
```

## Redireccion en pipelines

Los pipelines se combinan con redirecciones. La sintaxis permite redirigir entrada y salida en cualquier etapa del pipeline:

```bash
# Redirigir entrada de la primera etapa desde un archivo
sort < entrada.txt | uniq > salida.txt

# Redirigir salida de la ultima etapa a un archivo
grep "ERROR" log.txt | cut -d' ' -f3- > errores_procesados.txt

# Redirigir errores de una etapa intermedia
find / -name "*.log" 2>/dev/null | xargs grep "error" 2>/dev/null
```

> STDERR no atraviesa el pipe. Solo STDOUT pasa al siguiente comando. Para incluir STDERR en el pipeline se usa `2>&1` antes del pipe: `comando 2>&1 | siguiente`.

## El comando tee

`tee` lee de STDIN y escribe simultaneamente a STDOUT y a uno o varios archivos. Su nombre viene de la forma de T en las tuberias de fontaneria: una entrada que se bifurca en dos salidas.

```bash
# Ver salida en pantalla y guardarla en archivo a la vez
ls -la | tee listado.txt

# Anadir a archivo en lugar de sobrescribir
ls -la | tee -a listado.txt

# Escribir en multiples archivos
echo "datos" | tee archivo1.txt archivo2.txt archivo3.txt

# Combinar con grep para filtrar y guardar
ps aux | tee procesos.txt | grep apache
```

`tee` es indispensable en scripts de automatizacion donde se quiere registrar la salida en un log sin perder la visibilidad en tiempo real:

```bash
# Script que registra toda su salida
./script.sh | tee -a /var/log/script.log
```

## xargs: construir y ejecutar comandos desde stdin

`xargs` lee de STDIN y construye lineas de comandos para ejecutar otro programa. Es el complemento natural de los pipelines cuando el comando destino no lee de STDIN:

```bash
# find + xargs: eliminar archivos .tmp
find /tmp -name "*.tmp" -type f | xargs rm -f

# xargs con delimitador nulo (seguro con nombres con espacios)
find /tmp -name "*.tmp" -type f -print0 | xargs -0 rm -f

# Limitar el numero de argumentos por invocacion
xargs -n 1 echo < archivo.txt
```

> Pasar archivos de `find` a `xargs` es mas eficiente que `find -exec {} +` en algunos casos, pero ambos son equivalentes funcionalmente. La opcion `-print0` de find combinada con `-0` de xargs es necesaria para manejar nombres de archivo con espacios o caracteres especiales.

## Procesamiento por lotes con xargs

`xargs` permite paralelizar ejecuciones, lo que lo hace util para tareas que admiten concurrencia:

```bash
# Ejecutar hasta 4 procesos simultaneamente
seq 1 10 | xargs -P 4 -I {} echo "Procesando {}"

# Aplicar comando a cada archivo encontrado
find . -name "*.jpg" -type f | xargs -P $(nproc) -I {} convert {} -resize 800x600 {}
```

## Ejemplos practicos combinados

Pipeline para diagnosticar que procesos consumen mas memoria:

```bash
ps aux | sort -rn -k 4 | head -10
```

Pipeline para analizar codigos de respuesta HTTP en logs de nginx:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

Pipeline para encontrar los 5 archivos mas grandes del sistema:

```bash
find / -type f -size +10M 2>/dev/null | xargs ls -lhS 2>/dev/null | head -5
```

Pipeline para contar conexiones activas por direccion IP:

```bash
ss -tn | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10
```

> La composicion de pipes, filtros y redirecciones convierte la shell en un lenguaje de procesamiento de datos en si mismo. Programas como `awk` y `sort` estan disenados especificamente para actuar como etapas intermedias en pipelines, y su potencia se multiplica al combinarlos.

