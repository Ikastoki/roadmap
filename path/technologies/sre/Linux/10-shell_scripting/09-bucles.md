# Bucles

Los bucles permiten ejecutar un bloque de codigo repetidamente, ya sea un numero fijo de veces, mientras se cumpla una condicion, o recorriendo una lista de elementos. Son la herramienta que transforma un script de una secuencia lineal de instrucciones en un programa capaz de procesar grandes volumenes de datos con pocas lineas de codigo.

## for

El bucle `for` itera sobre una lista de elementos. Puede ser una lista explicita, el resultado de un comando, un rango numerico o los archivos que coinciden con un glob:

```bash
# Lista explicita
for fruta in manzana pera platano; do
    echo "Me gusta la $fruta"
done

# Rango numerico
for i in {1..5}; do
    echo "Numero: $i"
done

# Resultado de comando
for archivo in $(ls *.txt); do
    echo "Procesando: $archivo"
done

# Globbing
for img in fotos/*.jpg; do
    convert "$img" -resize 800x600 "${img%.jpg}_redimensionada.jpg"
done
```

> Al iterar sobre archivos con globbing, el resultado ya viene expandido por el shell y no requiere comillas dobles en la asignacion, pero si se necesita usar `$img` dentro del bucle deben usarse comillas dobles para preservar espacios en los nombres de archivo.

La sintaxis estilo C con `(( ))` permite un control mas preciso del indice:

```bash
for (( i=0; i<10; i++ )); do
    echo "Iteracion $i"
done
```

## while

El bucle `while` ejecuta el bloque mientras la condicion sea verdadera (codigo de salida 0). Es ideal para bucles controlados por un contador, lectura de archivos linea a linea, o espera de condiciones externas:

```bash
# Contador
contador=0
while [ $contador -lt 5 ]; do
    echo "Contador: $contador"
    (( contador++ ))
done

# Lectura de archivo
while IFS= read -r linea; do
    echo "Linea: $linea"
done < archivo.txt

# Bucle infinito con condicion de salida
while true; do
    read -p "Comando: " cmd
    [ "$cmd" = "salir" ] && break
    eval "$cmd"
done
```

El uso de `IFS=` antes de `read` evita que los espacios al principio y final de la linea se eliminen. La opcion `-r` evita que las barras invertidas se interpreten como escapes. Esta combinacion es el estandar para leer archivos sin perdida de datos.

## until

`until` es el inverso de `while`: ejecuta el bloque mientras la condicion sea falsa. Resulta util cuando la logica natural es "repetir hasta que algo ocurra" en lugar de "mientras no ocurra":

```bash
until ping -c 1 -W 1 192.168.1.1 &>/dev/null; do
    echo "Esperando conectividad..."
    sleep 2
done
echo "Gateway accesible"
```

## break y continue

`break` sale del bucle inmediatamente, independientemente de la condicion. `continue` salta a la siguiente iteracion sin ejecutar el resto del bloque:

```bash
for i in {1..10}; do
    if [ $i -eq 3 ]; then
        continue    # salta el 3
    fi
    if [ $i -eq 8 ]; then
        break       # termina al llegar a 8
    fi
    echo "Numero: $i"
done
# Salida: 1 2 4 5 6 7
```

En bucles anidados, `break N` y `continue N` permiten especificar cuantos niveles de anidamiento se ven afectados:

```bash
for i in {1..3}; do
    for j in {1..3}; do
        [ $i -eq 2 -a $j -eq 2 ] && break 2
        echo "i=$i j=$j"
    done
done
# Sale completamente al encontrar i=2, j=2
```

## Iteracion sobre archivos con find

Cuando se necesita procesar archivos recursivamente, combinar `find` con `while` o `for` es mas robusto que usar globs:

```bash
find /var/log -name "*.log" -mtime +7 | while IFS= read -r archivo; do
    echo "Comprimiendo: $archivo"
    gzip "$archivo"
done

# Alternativa mas segura con process substitution
while IFS= read -r -d '' archivo; do
    echo "Procesando: $archivo"
done < <(find /tmp -name "*.tmp" -print0)
```

El uso de `-print0` con `read -d ''` maneja correctamente nombres de archivo con espacios, saltos de linea o cualquier otro caracter especial.

