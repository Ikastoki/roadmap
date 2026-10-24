# Redireccion de entrada y salida

La shell de Linux asigna tres canales de comunicacion a todo proceso que se ejecuta: la entrada estandar (STDIN), la salida estandar (STDOUT) y la salida de error estandar (STDERR). Sin redireccion, estos canales estan conectados al terminal: el teclado como entrada y la pantalla como salida. La redireccion consiste en cambiar ese comportamiento por defecto para que un programa lea de un archivo en lugar del teclado o escriba en un archivo en lugar de la pantalla.

Este mecanismo es la base de la filosofia Unix: programas pequenos y especializados que pueden encadenarse sin conocer el origen o destino de sus datos. Un programa que escribe en STDOUT no necesita saber si su salida va a la pantalla, a un archivo o a otro programa; simplemente escribe en el descriptor de archivo 1 (STDOUT) y la shell se encarga del resto.

## Descriptores de archivo

Cada proceso en Unix tiene una tabla de descriptores de archivo. Los tres primeros estandarizados son:

- **0**: STDIN (entrada estandar) -- el programa lee datos de aqui
- **1**: STDOUT (salida estandar) -- el programa escribe datos normales aqui
- **2**: STDERR (salida de error) -- el programa escribe mensajes de error aqui

La separacion entre STDOUT y STDERR permite distinguir la salida normal de los errores. Un programa puede producir datos utiles por STDOUT y mensajes de diagnostico por STDERR, y el usuario puede decidir que hacer con cada canal de forma independiente.

```bash
# Ver los descriptores de archivo de un proceso en /proc
ls -la /proc/$$/fd
# fd/0 -> /dev/pts/0  (STDIN conectado al terminal)
# fd/1 -> /dev/pts/0  (STDOUT conectado al terminal)
# fd/2 -> /dev/pts/0  (STDERR conectado al terminal)
```

## Redireccion de salida: `>` y `>>`

El operador `>` redirige STDOUT a un archivo, sobrescribiendo su contenido si existe. El operador `>>` redirige STDOUT a un archivo anadiendo al final sin sobrescribir.

```bash
# Redirigir salida a un archivo (crea o sobrescribe)
echo "Hola, mundo" > saludo.txt

# Redirigir salida anadiendo al final
echo "Otra linea" >> saludo.txt

# Ver el contenido
cat saludo.txt
# Hola, mundo
# Otra linea
```

Si el archivo no existe, ambos operadores lo crean. La diferencia es crucial en logs: `>` pierde el historial anterior, `>>` lo conserva.

```bash
# Si el archivo no existe, se crea automaticamente
ls /tmp > listado.txt

# Redireccion a un directorio (error)
ls /tmp > /directorio_que_no_existe/archivo.txt
# -bash: /directorio_que_no_existe/archivo.txt: No such file or directory
```

> La shell crea el archivo de destino antes de ejecutar el comando. Si el archivo no se puede crear (permisos, ruta inexistente), el comando ni siquiera se ejecuta.

## Redireccion de entrada: `<`

El operador `<` redirige STDIN desde un archivo. El programa lee su entrada del archivo como si estuviera escribiendo el usuario desde el teclado.

```bash
# Contar lineas de un archivo usando redireccion de entrada
wc -l < saludo.txt
# 2

# Equivalente sin redireccion (wc acepta argumentos de archivo)
wc -l saludo.txt

# Ordenar el contenido de un archivo
sort < saludo.txt
```

La redireccion de entrada es util cuando el programa no acepta nombres de archivo como argumento y solo lee de STDIN:

```bash
# tr solo lee de STDIN, no acepta archivos como argumento
tr 'a-z' 'A-Z' < saludo.txt
# HOLA, MUNDO
# OTRA LINEA
```

## Redireccion de error: `2>`

El operador `2>` redirige STDERR a un archivo. Es identico a `>` pero actuando sobre el descriptor 2 en lugar del 1.

```bash
# Redirigir solo errores a un archivo
ls /etc/passwd /archivo_inexistente 2> errores.txt
# /etc/passwd (esto sale por STDOUT y se ve en pantalla)

cat errores.txt
# ls: cannot access '/archivo_inexistente': No such file or directory

# Anadir errores al final de un archivo
comando_que_falla 2>> errores.log
```

Esta separacion permite capturar errores sin mezclarlos con la salida normal:

```bash
# Procesar salida normal y errores por separado
find / -name "*.conf" 2>/dev/null | head -5
# Redirige errores a /dev/null (los descarta) y muestra solo aciertos
```

## Redireccion combinada: `&>`, `2>&1`, `1>&2`

Para redirigir tanto STDOUT como STDERR al mismo destino existen varias sintaxis equivalentes:

```bash
# Redirigir ambos canales al mismo archivo (bashismo)
comando &> salida_completa.txt

# Forma POSIX estandar (duplicar descriptor)
comando > salida_completa.txt 2>&1

# Anadir ambos canales al final
comando &>> salida_completa.txt
comando >> salida_completa.txt 2>&1
```

El orden de los operandos importa. La shell evalua las redirecciones de izquierda a derecha:

```bash
# CORRECTO: redirige STDOUT a archivo, luego STDERR a donde apunte STDOUT
comando > archivo 2>&1

# INCORRECTO: redirige STDERR a STDOUT (que aun apunta al terminal), luego STDOUT a archivo
comando 2>&1 > archivo
# STDERR seguira yendo al terminal porque cuando se evaluo 2>&1, STDOUT aun no estaba redirigido
```

> `2>&1` significa "duplica el descriptor 1 en el descriptor 2". No es un puntero: STDERR pasa a escribir exactamente donde este escribiendo STDOUT en ese momento, y si STDOUT se redirige despues, STDERR no lo seguira. Por eso el orden `> archivo 2>&1` es el correcto.

## Redireccion a /dev/null

`/dev/null` es un archivo especial que descarta todo lo que se le escribe. Leer de el devuelve EOF inmediatamente. Es util para silenciar salida no deseada:

```bash
# Descartar toda la salida (stdout + stderr)
comando &>/dev/null

# Descartar solo errores
comando 2>/dev/null

# Descartar solo salida normal
comando >/dev/null
```

## Here documents y here strings

La shell permite redirigir entrada directamente desde el propio script sin archivos intermedios:

```bash
# Here document: redirige stdin desde un bloque de texto hasta el delimitador
cat << EOF
Esto es un here document.
Puede contener multiples lineas.
EOF

# Here string: redirige stdin desde una cadena (bashismo)
grep "error" <<< "Todo correcto\nSe ha producido un error\nFin"
# Se ha producido un error
```

Los here documents son especialmente utiles en scripts para pasar entrada a programas interactivos o generar archivos de configuracion:

```bash
# Generar un archivo de configuracion desde un script
cat > /tmp/config.conf << CONF
server {
    listen 80;
    server_name ejemplo.com;
}
CONF
```

## Redireccion con exec

El comando `exec` integrado de la shell permite modificar los descriptores de archivo de la propia shell, no solo de un comando. Esto cambia el destino de la salida para el resto del script o sesion:

```bash
# Redirigir toda la salida del script a un archivo
exec > log.txt 2>&1
echo "Esto va al archivo"
echo "Esto tambien"
# A partir de aqui, todo STDOUT y STDERR van a log.txt

# Restaurar la salida estandar original (primero guardar el descriptor)
exec 3>&1  # duplicar STDOUT original en fd 3
exec > log.txt
echo "Esto va al archivo"
exec >&3   # restaurar STDOUT
echo "Esto vuelve a la pantalla"
```

> La redireccion con `exec` es potente pero peligrosa: un error de redireccion puede dejar el script sin salida visible. Es practica comun usarla solo al inicio de scripts de automatizacion que no requieren interaccion humana.

