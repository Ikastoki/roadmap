# Sed y Awk

Dos de las herramientas mas poderosas del arsenal Unix para procesamiento de texto: `sed` (Stream Editor) para edicion en flujo y `awk` para analisis estructurado de datos. Ambas operan linea por linea sobre archivos de texto y se integran perfectamente en tuberias con otros comandos.

## Sed: editor de flujo

`sed` procesa texto linea a linea aplicando operaciones de edicion sin cargar el archivo completo en memoria. Su uso mas comun es la sustitucion de patrones, pero soporta eliminacion, insercion, transformacion y escritura selectiva.

### Sustituciones

La operacion basica es `s/patron/reemplazo/banderas`. Por defecto, `sed` trabaja sobre stdin e imprime el resultado en stdout:

```bash
echo "hola mundo" | sed 's/mundo/MUNDO/'
# hola MUNDO

sed 's/foo/bar/g' archivo.txt    # sustituye todas las ocurrencias (global)
sed 's/foo/bar/2' archivo.txt    # solo la segunda ocurrencia de cada linea
sed 's/^/  /' archivo.txt        # anade dos espacios al inicio (identar)
sed 's/$/---/' archivo.txt       # anade --- al final de cada linea
```

Las banderas mas comunes son:
- `g` global (todas las ocurrencias por linea).
- `i` o `I` ignorar mayusculas/minusculas.
- `p` imprimir lineas coincidentes (util con `-n`).
- `w archivo` escribir lineas coincidentes a un archivo.

### Direccion de lineas

Se puede restringir la operacion a lineas especificas mediante direcciones:

```bash
sed '3s/error/ERROR/g' log.txt       # solo linea 3
sed '5,10s/foo/bar/g' archivo.txt    # lineas 5 a 10
sed '/^#/d' config.txt               # eliminar lineas comentadas
sed '/^$/d' archivo.txt              # eliminar lineas vacias
sed -n '/ERROR/p' log.txt            # mostrar solo lineas con ERROR
```

### Eliminacion y edicion multiple

El comando `d` elimina lineas que coinciden con la direccion. Con `-e` se pueden encadenar multiples operaciones:

```bash
sed '/^#/d; /^$/d' archivo.txt                # elimina comentarios y vacios
sed -e 's/[[:space:]]*$//' -e '/^$/d' file    # elimina trailing spaces y vacios

sed -i.bak 's/192.168.1/10.0.0/g' config.conf # edicion in-place con backup
```

La opcion `-i` edita el archivo directamente (in-place). Es recomendable usar siempre una extension de backup (`-i.bak`) por si algo sale mal.

## Awk: procesamiento estructurado

`awk` es un lenguaje de programacion completo orientado al procesamiento de texto estructurado en columnas. Opera bajo el paradigma "patron/accion": lee cada linea, la divide en campos, evalua patrones y ejecuta acciones cuando se cumplen.

### Estructura basica

```bash
awk '{print $1, $3}' archivo.txt          # imprime columnas 1 y 3
awk '/error/ {print $0}' log.txt          # lineas que contienen "error"
awk '$3 > 50 {print $1, $3}' datos.txt    # col3 > 50
```

`$0` representa la linea completa, `$1` el primer campo, `$2` el segundo, etc. El separador de campos por defecto es espacio en blanco, pero se puede cambiar con `-F`:

```bash
awk -F: '{print $1, $6}' /etc/passwd      # usuario y home
awk -F',' '{print $2, $4}' datos.csv      # archivo CSV
```

### Patrones y acciones

Un programa awk puede tener multiples bloques patron/accion:

```bash
awk '
BEGIN {print "=== Reporte ==="}
/ERROR/ {errores++}
/WARN/  {warnings++}
END {print "Errores:", errores+0, "Warnings:", warnings+0}
' log.txt
```

- `BEGIN` se ejecuta antes de procesar la primera linea (inicializaciones).
- `END` se ejecuta despues de procesar todas las lineas (resumenes).
- Los patrones pueden ser expresiones regulares, comparaciones numericas o combinaciones logicas.

### Variables internas

`awk` proporciona variables predefinidas utiles:

| Variable | Descripcion |
|----------|-------------|
| `NR` | Numero de registro actual (linea) |
| `NF` | Numero de campos en la linea actual |
| `FS` | Separador de campos (input) |
| `OFS` | Separador de campos (output) |
| `RS` | Separador de registros (input) |
| `ORS` | Separador de registros (output) |

```bash
awk '{print NR, NF, $0}' archivo.txt      # numero de linea, campos y contenido
awk 'NF > 0' archivo.txt                  # solo lineas no vacias
awk '{print $(NF-1), $NF}' archivo.txt    # penultimo y ultimo campo
```

### Casos practicos

**Procesamiento de logs:**

```bash
# Contar peticiones HTTP por codigo de estado
awk '{estados[$9]++} END {for (e in estados) print e, estados[e]}' access.log

# Filtrar peticiones 5xx
awk '$9 ~ /^5[0-9][0-9]/ {print $1, $7, $9}' access.log
```

**Analisis de archivos CSV:**

```bash
# Sumar columna de ventas por categoria
awk -F',' '{ventas[$2] += $3} END {for (cat in ventas) print cat, ventas[cat]}' ventas.csv

# Calcular promedio de una columna
awk -F',' '{sum+=$3; count++} END {print "Promedio:", sum/count}' datos.csv
```

**Formateo de salida con printf:**

```bash
awk '{printf "%-20s %8d\n", $1, $3}' archivo.txt
awk 'BEGIN {printf "%-20s %8s\n", "Nombre", "Saldo"} {printf "%-20s %8.2f\n", $1, $3}' cuentas.txt
```

> Aunque `sed` y `awk` tienen areas de responsabilidad que se solapan, la practica habitual es usar `sed` para sustituciones simples y transformaciones de una linea, y `awk` para procesamiento que requiere operaciones por campo, acumuladores o logicas condicionales mas complejas.

