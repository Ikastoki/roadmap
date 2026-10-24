# Expresiones regulares y grep

Las expresiones regulares (regex) son un lenguaje de patrones que permite describir conjuntos de cadenas de texto mediante una notacion concisa y precisa. En el shell, combinadas con herramientas como `grep`, `sed`, `awk` y las condiciones `[[ ]]`, constituyen la tecnica mas potente para busqueda, filtrado y transformacion de texto.

## Sintaxis basica (BRE) y extendida (ERE)

Existen dos estandares principales de expresiones regulares en Unix: la sintaxis basica (BRE, Basic Regular Expressions) y la extendida (ERE, Extended Regular Expressions). La diferencia fundamental es que en BRE los metacaracteres `?`, `+`, `{`, `}`, `(` y `)` deben escaparse con barra invertida para que actuen como metacaracteres; en ERE actuan como metacaracteres sin escapar.

```bash
# BRE: metacaracteres escapados
grep 'color\?' archivo.txt          # colo seguido de r opcional
grep '^a\+' archivo.txt             # lineas que empiezan con una o mas a

# ERE: metacaracteres sin escapar
grep -E 'color?' archivo.txt
grep -E '^a+' archivo.txt
```

### Metacaracteres basicos

| Simbolo | Significado |
|---------|-------------|
| `.` | Cualquier caracter individual (excepto nueva linea) |
| `^` | Inicio de linea |
| `$` | Fin de linea |
| `*` | Cero o mas repeticiones del elemento anterior |
| `+` | Una o mas repeticiones (ERE) |
| `?` | Cero o una repeticion (ERE) |
| `{n,m}` | Entre n y m repeticiones (ERE) |
| `[abc]` | Cualquier caracter del conjunto |
| `[^abc]` | Cualquier caracter NO del conjunto |
| `\|` | Alternancia (ERE): esto O aquello |
| `()` | Agrupacion (ERE) |

### Clases de caracteres predefinidas

```bash
[[:alnum:]]   # alfanumerico
[[:alpha:]]   # alfabetico
[[:digit:]]   # digito
[[:lower:]]   # minuscula
[[:upper:]]   # mayuscula
[[:space:]]   # espacio, tabulador, nueva linea
[[:punct:]]   # signo de puntuacion
```

## grep avanzado

`grep` busca patrones en archivos o entrada estandar y muestra las lineas coincidentes. Sus variantes son `grep` (BRE por defecto), `grep -E` (ERE) y `grep -P` (PCRE, Perl-compatible, no siempre disponible).

### Opciones clave

```bash
grep -i "error" log.txt               # ignorar mayusculas/minusculas
grep -v "#" config.txt                # lineas que NO coinciden (invertir)
grep -c "ERROR" log.txt               # contar coincidencias
grep -n "funcion" *.sh               # mostrar numero de linea
grep -r "TODO" /src/                  # busqueda recursiva
grep -l "main" *.c                    # solo nombres de archivo
grep -w "word" archivo.txt            # palabra completa (word boundaries)
grep -A2 -B2 "error" log.txt          # contexto: 2 lineas antes y despues
```

### Patrones frecuentes

```bash
# IPs en un archivo
grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' access.log

# Correos electronicos
grep -Eo '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' contactos.txt

# URLs
grep -Eo 'https?://[^[:space:]]+' texto.txt

# Lineas con exactamente n caracteres
grep -E '^.{80}$' archivo.txt

# Numeros de telefono (formato espanol)
grep -E '\+34[0-9]{9}' contactos.txt

# Comentarios en codigo
grep -E '^\s*(#|//)' *.sh *.c
```

### grep con salida solo de la coincidencia

La opcion `-o` muestra unicamente la parte de la linea que coincide con el patron, no toda la linea. Esto es util para extraer datos estructurados sin procesamiento adicional:

```bash
grep -oP 'status":\s*"\K[^"]+' response.json
grep -oP '<title>\K[^<]+' pagina.html
```

La construccion `\K` descarta lo que coincide antes de ella (soporte PCRE con `-P`). No esta disponible en todos los sistemas; como alternativa portable se puede usar `sed` o `awk`.

## Regex en scripts

Dentro de Bash, `[[ ]]` soporta el operador `=~` para evaluar expresiones regulares ERE:

```bash
#!/bin/bash
if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Email valido"
else
    echo "Email invalido"
fi
```

A diferencia de `grep`, la regex en `[[ =~ ]]` no se entrecomilla (si se entrecomilla, se interpreta como cadena literal). Los parentesis de captura en Bash almacenan los grupos en la variable `BASH_REMATCH`:

```bash
if [[ "error: archivo no encontrado" =~ ^([a-z]+):\ (.+)$ ]]; then
    echo "Tipo: ${BASH_REMATCH[1]}"
    echo "Mensaje: ${BASH_REMATCH[2]}"
fi
```

## Buenas practicas con regex

- Usar `grep -E` para patrones complejos en lugar de escaparlo todo en BRE.
- Probar las regex en un archivo de prueba pequeno antes de aplicarlas a datos reales.
- Recordar que `grep` trabaja por lineas; si un patron se extiende entre lineas, no coincidira.
- Escapar puntos literales (`\.`) en IPs y URLs para evitar que `.` coincida con cualquier caracter.
- Usar clases POSIX (`[[:digit:]]`) en lugar de rangos (`[0-9]`) para codificaciones no ASCII.

