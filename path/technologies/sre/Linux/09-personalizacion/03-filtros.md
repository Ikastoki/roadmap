# Filtros de texto

Los filtros son programas que leen datos de STDIN, los transforman de alguna manera y escriben el resultado en STDOUT. Estan disenados para usarse como etapas intermedias en pipelines: no producen informacion nueva, sino que procesan, seleccionan o reorganizan la informacion que reciben. La mayoria soporta tanto lectura desde archivo como desde STDIN, lo que los hace intercambiables segun el contexto.

## grep: busqueda de patrones

`grep` (global regular expression print) busca lineas que coincidan con un patron en uno o varios archivos o en STDIN. Es probablemente el filtro mas utilizado en la shell. Su eficiencia radica en que solo examina las lineas una vez, sin cargar el archivo completo en memoria.

```bash
# Buscar una cadena literal en un archivo
grep "error" /var/log/syslog

# Buscar en multiples archivos
grep "192.168.1." /var/log/auth.log /var/log/syslog

# Buscar desde STDIN (en un pipeline)
dmesg | grep "usb"

# Buscar ignorando mayusculas/minusculas
grep -i "warning" sistema.log

# Mostrar lineas que NO coinciden
grep -v "#" /etc/ssh/sshd_config
```

Opciones habituales:

```bash
# Mostrar numero de linea
grep -n "Listen" /etc/ssh/sshd_config

# Contar coincidencias
grep -c "error" app.log

# Mostrar solo el nombre del archivo (no el contenido)
grep -l "password" /etc/*.conf

# Busqueda recursiva en directorios
grep -r "main(" ./src/

# Mostrar contexto (lineas anteriores y posteriores)
grep -C 3 "Exception" stacktrace.log
grep -B 2 "ERROR" log.txt  # 2 lineas antes
grep -A 5 "def " script.py  # 5 lineas despues
```

### Expresiones regulares basicas en grep

Por defecto, grep usa expresiones regulares basicas (BRE). En este modo, los metacaracteres `?`, `+`, `{`, `|`, `(`, `)` deben escaparse con `\` para activar su significado especial. Con `-E` (extended regex, ERE) estos caracteres funcionan sin escapar.

```bash
# BRE: el punto coincide con cualquier caracter
grep "fi.e" palabras.txt
# fire, five, file, etc.

# BRE: el asterisco coincide con 0 o mas repeticiones
grep "fo*" palabras.txt
# f, fo, foo, fooo, etc.

# ERE con -E: el mas coincide con 1 o mas
grep -E "go+l" palabras.txt
# gol, gool, goool, etc.

# ERE: alternancia
grep -E "error|warning|critical" log.txt

# ERE: cuantificadores exactos
grep -E "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" ips.txt
```

> La diferencia entre BRE y ERE solo afecta a la sintaxis de los metacaracteres. La mayoria de usos cotidianos se benefician de `grep -E` por su escritura mas natural. Para expresiones muy complejas, `grep -P` activa el motor PCRE (Perl-compatible), aunque no esta disponible en todas las implementaciones.

Anclas y clases de caracteres:

```bash
# Lineas que empiezan por #
grep "^#" /etc/fstab

# Lineas que terminan en .conf
grep "\.conf$" listado.txt

# Clases predefinidas POSIX
grep "[[:digit:]]" archivo.txt     # digitos
grep "[[:space:]]" archivo.txt     # espacios, tabs
grep "[[:alpha:]]" archivo.txt     # letras
grep "[[:alnum:]]" archivo.txt     # letras y digitos
```

## sort: ordenacion

`sort` ordena las lineas de un archivo o STDIN. Su algoritmo implementa merge sort externo, capaz de ordenar archivos que no caben en memoria.

```bash
# Orden alfabetico
sort palabras.txt

# Orden numerico
sort -n numeros.txt

# Orden inverso
sort -r palabras.txt

# Ordenar ignorando mayusculas
sort -f palabras.txt

# Ordenar por campo (separador por defecto: espacios/tabs)
sort -k 2 -t: /etc/passwd

# Eliminar duplicados (solo lineas adyacentes identicas)
sort -u palabras.txt
```

La combinacion con otros filtros es donde `sort` muestra su potencia:

```bash
# Top 10 procesos por uso de memoria
ps aux | sort -rn -k 4 | head -10

# Archivos mas grandes del directorio
du -sh /* | sort -rh | head -5

# IPs con mas conexiones
ss -tn | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn
```

> `-h` (human numeric sort) interpreta sufijos como K, M, G. Sin el, `sort` trata "100M" como menor que "50K" porque "1" < "5". `-h` los interpreta correctamente como 100000000 y 50000.

## cut: extraccion de columnas

`cut` extrae secciones de cada linea de texto. Trabaja por posiciones de caracter, delimitadores o campos.

```bash
# Extraer por posiciones de caracter (1-indexed)
cut -c 1-5 archivo.txt       # primeros 5 caracteres
cut -c 3- archivo.txt        # desde el caracter 3 hasta el final
cut -c -10 archivo.txt       # primeros 10 caracteres

# Extraer por campos con delimitador
cut -d: -f1 /etc/passwd      # nombres de usuario
cut -d: -f1,3 /etc/passwd    # usuario y UID
cut -d: -f1-3 /etc/passwd    # tres primeros campos

# Con pipeline
who | cut -d' ' -f1 | sort -u  # usuarios conectados sin repetir
```

> `cut` no puede reordenar campos ni usar delimitadores variables (como espacios multiples). Para eso se necesita `awk`. `cut` es rapido y simple cuando el delimitador es fijo y no se necesita logica condicional.

## uniq: eliminacion de duplicados

`uniq` elimina o cuenta lineas duplicadas _consecutivas_. No detecta duplicados no adyacentes, por lo que su uso tipico va precedido de `sort`.

```bash
# Eliminar duplicados consecutivos
uniq archivo.txt

# Contar ocurrencias
uniq -c archivo.txt

# Mostrar solo las lineas que aparecen exactamente una vez
uniq -u archivo.txt

# Mostrar solo las lineas que aparecen mas de una vez
uniq -d archivo.txt

# Ignorar diferencias de mayusculas
uniq -i archivo.txt
```

La combinacion canonica para contar frecuencias:

```bash
sort archivo.txt | uniq -c | sort -rn
```

Ejemplo real: encontrar los codigos de estado HTTP mas frecuentes:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
#   4523 200
#   1234 404
#    345 500
#     12 403
```

## tr: traduccion o eliminacion de caracteres

`tr` (translate) no acepta archivos como argumento; solo lee de STDIN. Transforma caracteres segun reglas de correspondencia.

```bash
# Convertir mayusculas a minusculas
echo "HOLA" | tr 'A-Z' 'a-z'

# Convertir minusculas a mayusculas
echo "hola" | tr 'a-z' 'A-Z'

# Eliminar caracteres
echo "hola mundo" | tr -d ' '      # eliminar espacios
tr -d '\r' < windows.txt > unix.txt  # eliminar retornos de carro

# Comprimir repeticiones
echo "hola    mundo" | tr -s ' '   # hola mundo (un solo espacio)

# Complementar (todo excepto)
echo "hola123" | tr -cd '[:digit:]'  # 123 (solo digitos)
```

Casos de uso frecuentes:

```bash
# Convertir archivos DOS a Unix
tr -d '\r' < archivo_dos.txt > archivo_unix.txt

# Normalizar espacios en blanco
cat desordenado.txt | tr -s ' \t\n' ' '

# Contar palabras (conjuntamente con wc)
cat texto.txt | tr -s ' ' '\n' | sort | uniq -c | sort -rn | head -10
```

## wc: conteo de lineas, palabras y caracteres

`wc` (word count) cuenta lineas, palabras y caracteres (o bytes) de uno o varios archivos o de STDIN.

```bash
# Lineas, palabras y caracteres
wc archivo.txt

# Solo lineas
wc -l archivo.txt

# Solo palabras
wc -w archivo.txt

# Solo caracteres
wc -c archivo.txt   # bytes
wc -m archivo.txt   # caracteres (multibyte)

# Multiples archivos
wc -l *.log

# Desde STDIN
cat archivo.txt | wc -l
```

Combinaciones utiles:

```bash
# Lineas de codigo en un proyecto
find . -name "*.py" -exec wc -l {} + | tail -1

# Cuantas veces aparece una palabra
grep -o "error" log.txt | wc -l

# Archivos en un directorio
ls -1 | wc -l
```

